# Upload transports and lifecycle

## REST transport

Import `ModernAdminUploadModule` from its Nest-only subpath so importing the
root feature does not pull Nest, Busboy, RxJS, or Swagger into non-Nest hosts:

```ts
import { ModernAdminUploadModule } from "@modern-admin/feature-upload/nest";
```

The module requires `ModernAdminModule.forRoot({ global: true })` in the DI
tree. It exposes:

- `POST /admin/api/resources/:resourceId/actions/upload?field=<property>` —
  multipart upload; returns `UploadedFileInfo[]`;
- `DELETE /admin/api/resources/:resourceId/actions/upload?field=<property>&key=<key>` —
  delete a still-pending upload; returns 204 or 404.

The controller accepts files only for a property carrying the provider id
installed by `uploadFeature`/`jsonByKeyFeature`. It sanitizes the client
filename before calling `uploadPath`, rejects unsafe computed keys, enforces
declared MIME patterns, and buffers each file under configured limits.

Module options and defaults:

| Option                      |   Default | Meaning                                                         |
| --------------------------- | --------: | --------------------------------------------------------------- |
| `global`                    |   `false` | Export upload module options globally                           |
| `pendingTtlMs`              |    1 hour | Delay before abandoned keys become sweepable                    |
| `sweepIntervalMs`           | 5 minutes | Sweep cadence; `0` disables scheduling                          |
| `maxFileSize`               |    25 MiB | Hard per-file in-memory buffer cap                              |
| `maxFiles`                  |        20 | Per-request cap for `isArray`; single fields always cap at 1    |
| `acknowledgeSingleInstance` |   `false` | Silence the deployment warning; does not add distributed safety |

Per-property `maxSize` can tighten but never raise the module cap. MIME
patterns use HTML `accept` syntax (`image/*`, `image/jpeg`, etc.) and match the
declared multipart content type, not the file signature.

## Pending and persisted file lifecycle

Uploads happen before the record form is saved:

```text
select file -> provider.upload -> track pending key -> submit form
                                               |          |
                                         abandon/cancel   +-> confirm key
                                               |          +-> persist key
                                               +-> sweeper/provider.delete
```

The process-level `PendingUploadsRegistry` tracks a key after storage succeeds.
`new.after`/`edit.after` hooks confirm stored keys. Removing an unsaved file
calls the cancel endpoint. The sweeper deletes expired, abandoned keys.

This lifecycle is unsafe across processes because the registry is an in-memory
`Map`. Run one Nest process. `acknowledgeSingleInstance: true`, a longer TTL,
or sticky load balancing only hides/reduces the race; none creates a shared
source of truth.

Replacement/deletion cleanup is best-effort: upload feature hooks swallow
provider deletion failures so a storage outage does not fail the database
action. Add storage observability and an independent reconciliation/garbage
collection job for production.

## GraphQL transport

Use GraphQL only when the host already exposes the Modern Admin GraphQL module:

```sh
bun add @modern-admin/graphql graphql busboy
```

```ts
import { ModernAdminGraphqlModule } from "@modern-admin/graphql";
import { uploadGraphqlExtension } from "@modern-admin/feature-upload/graphql";

ModernAdminGraphqlModule.forRoot({
  maxFileSize: 25 * 1024 * 1024,
  maxFiles: 10,
  extensions: [uploadGraphqlExtension({ pendingTtlMs: 60 * 60 * 1000 })],
});
```

This adds:

- `adminUpload(resourceId, field, file)` returning one `UploadedFileInfo`;
- `adminCancelUpload(resourceId, field, key)` returning `Boolean`.

The wire format follows the GraphQL multipart request specification. The
GraphQL parser defaults to 25 MiB per file and 10 file parts per request. An
array property still uploads one file per `adminUpload` mutation invocation.
Set the extension `pendingTtlMs` equal to the REST module value when both
transports are enabled.

## File children inside `jsonByKeyFeature`

For a JSON column split into virtual children, configure upload on the child,
not with a separate top-level `uploadFeature` entry:

```ts
import { extname } from "node:path";
import { uuidv7 } from "@modern-admin/core";
import { jsonByKeyFeature } from "@modern-admin/feature-json-by-key";

jsonByKeyFeature({
  controlField: "region",
  keys: ["eu", "us", "asia"],
  properties: {
    previews: {
      child: {
        type: "file",
        isArray: false,
        upload: {
          provider,
          mimeTypes: ["image/*"],
          maxSize: 10 * 1024 * 1024,
          uploadPath: (filename, ctx) => `regional/${ctx.key}/${uuidv7()}${extname(filename)}`,
        },
      },
    },
  },
});
```

The callback context contains `{ key, property }`. The feature creates a
deterministic provider id for every virtual child and installs the same pending,
replace, and delete lifecycle as ordinary file fields.

## Troubleshooting

| Symptom                                   | Check                                                                                                            |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Upload endpoint returns 404               | Import `ModernAdminUploadModule`; use the fixed `/admin/api/resources/.../actions/upload` route                  |
| Provider is not registered                | Attach `uploadFeature` to the resource; do not set `type: 'file'` manually; ensure resource decoration completed |
| `MODERN_ADMIN` DI error                   | Set `ModernAdminModule.forRoot({ global: true })` and keep modules in the same Nest app                          |
| Local upload succeeds but preview 404s    | Mount `uploadDir` statically at the exact `baseUrl`; check process working directory                             |
| Existing S3 key shows only a filename     | Supply a stable `publicBaseUrl`/URL template, or a custom protected URL resolver for private objects             |
| 413 / oversized error                     | Compare property `maxSize`, REST `maxFileSize`, GraphQL `maxFileSize`, and proxy/body limits                     |
| Multiple selection truncates to one       | Set `isArray: true`, use an array-capable DB field, and keep `maxFiles` above 1                                  |
| Saved files disappear after about an hour | Multiple replicas are racing through process-local pending state; run a single process                           |
| Old files remain in storage               | Confirm the resource uses standard edit/delete hooks; inspect provider `delete` logs and run reconciliation      |
| Custom key rejected                       | Remove leading slash, `..`, backslash traversal, or other unsafe segments; generate a relative key               |

## Canonical source files

- `packages/feature-upload/src/types.ts`
- `packages/feature-upload/src/upload-feature.ts`
- `packages/feature-upload/src/providers/local.ts`
- `packages/feature-upload/src/providers/s3.ts`
- `packages/feature-upload/src/nest/upload.module.ts`
- `packages/feature-upload/src/nest/upload.controller.ts`
- `packages/feature-upload/src/pending-registry.ts`
- `packages/feature-upload/src/graphql.ts`
- `packages/feature-json-by-key/src/types.ts`
- `apps/_shared/src/admin/products/products.controller.ts`
- `apps/_shared/src/admin/regional/regional-content.controller.ts`
