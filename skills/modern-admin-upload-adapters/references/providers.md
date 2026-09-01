# Upload providers

## Built-in provider matrix

| Provider                 | Required host packages         | Use                                                         | Important options                                                                                     |
| ------------------------ | ------------------------------ | ----------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `LocalUploadProvider`    | `@modern-admin/feature-upload` | Local development or one persistent filesystem              | `uploadDir`, optional `baseUrl` (defaults to `/uploads`)                                              |
| `S3UploadProvider`       | plus `@aws-sdk/client-s3`      | AWS S3, MinIO, R2, Spaces, other S3-compatible storage      | `bucket`, `region`, optional `endpoint`, `forcePathStyle`, `prefix`, `acl`, `publicBaseUrl`, `signed` |
| Custom `IUploadProvider` | provider SDK                   | GCS, Azure Blob, media pipelines, encrypted/private storage | `upload`, `getUrl`, `delete`, optional `urlTemplate`                                                  |

## Local storage

```ts
import { resolve } from "node:path";
import { LocalUploadProvider } from "@modern-admin/feature-upload";

const provider = new LocalUploadProvider({
  uploadDir: resolve(process.cwd(), "uploads"),
  baseUrl: "/uploads",
});
```

The provider creates directories and writes/deletes files, but it does not
serve them. Mount exactly `uploadDir` at exactly `baseUrl`, and put the
directory on a persistent volume if it must survive restarts. Do not use local
disk when multiple replicas can write different filesystems.

`LocalUploadProvider` resolves every key within `uploadDir`; unsafe or absolute
upload keys throw, while an unsafe delete key is ignored. Preserve this
containment property in custom local adapters.

## S3-compatible storage

Install only the SDK pieces used by the selected mode:

```sh
bun add @aws-sdk/client-s3
# Optional streaming/multipart uploader used when installed:
bun add @aws-sdk/lib-storage
# Required only for `signed: true`:
bun add @aws-sdk/s3-request-presigner
```

Public bucket or CDN-backed example:

```ts
import { S3UploadProvider } from "@modern-admin/feature-upload";

const provider = new S3UploadProvider({
  bucket: process.env.UPLOAD_BUCKET!,
  region: process.env.AWS_REGION!,
  prefix: "admin",
  publicBaseUrl: process.env.UPLOAD_CDN_URL,
});
```

Let the AWS credential chain resolve credentials when possible. If passing
`accessKeyId` and `secretAccessKey`, keep them in server-only secrets. Scope
the principal to the configured bucket/prefix and the minimum required object
operations.

For MinIO or another endpoint:

```ts
const provider = new S3UploadProvider({
  bucket: "assets",
  region: "us-east-1",
  endpoint: "http://minio:9000",
  forcePathStyle: true,
  publicBaseUrl: "https://assets.example.com",
});
```

### Current signed-URL limitation

`signed: true` makes `getUrl(key)` return a presigned URL and makes
`urlTemplate()` return `undefined`. The upload response can therefore preview
a newly uploaded file, but the current React file renderer cannot request a
fresh URL for a persisted key after reload: without a template it only renders
a link when the stored value itself starts with `http`.

Do not store an expiring presigned URL in the database. Until the renderer has
a key-to-URL resolver, use one of these approaches:

1. expose objects through an authenticated/public CDN URL that can be expressed
   as `publicBaseUrl + key`;
2. register a host custom file display/editor that requests a fresh URL from a
   protected endpoint; make that endpoint authorize the record/field and derive
   its key from stored data instead of accepting an arbitrary client key; or
3. use a custom provider/component contract owned by the host.

## Custom provider contract

Implement all required methods and keep `delete` idempotent:

```ts
import { extname } from "node:path";
import { uuidv7 } from "@modern-admin/core";
import { isUnsafeKey, type IUploadProvider, type UploadedFile } from "@modern-admin/feature-upload";

export class GcsUploadProvider implements IUploadProvider {
  constructor(
    private readonly bucket: {
      file(key: string): {
        save(data: Buffer, options: object): Promise<unknown>;
        delete(options: object): Promise<unknown>;
      };
    },
    private readonly publicBaseUrl: string,
  ) {}

  async upload(file: UploadedFile, key?: string): Promise<string> {
    const resolvedKey = key ?? `admin/${uuidv7()}${extname(file.originalName)}`;
    if (isUnsafeKey(resolvedKey)) throw new Error("Unsafe upload key");
    await this.bucket.file(resolvedKey).save(file.buffer, {
      contentType: file.mimeType,
      resumable: false,
    });
    return resolvedKey;
  }

  getUrl(key: string): string {
    if (isUnsafeKey(key)) throw new Error("Unsafe upload key");
    return `${this.publicBaseUrl}/${key}`;
  }

  async delete(key: string): Promise<void> {
    if (isUnsafeKey(key)) return;
    await this.bucket.file(key).delete({ ignoreNotFound: true });
  }

  urlTemplate(): string {
    return `${this.publicBaseUrl}/{key}`;
  }
}
```

Add any stricter backend-specific key rules on top of the public
`isUnsafeKey` helper. Always reject absolute paths, traversal segments, NUL
bytes, and any prefix escape. Return the exact key that the database should
persist.

`getUrl` may be sync or async. Implement `urlTemplate()` only when the browser
can construct a stable URL by replacing `{key}`. Omit it for request-specific
or expiring URLs and supply a host UI/resolver for persisted records.

Test at minimum:

- provided key and generated-key upload paths;
- MIME metadata forwarded to storage;
- safe URL generation;
- missing-key delete;
- traversal/absolute/NUL keys;
- provider errors and retry expectations.
