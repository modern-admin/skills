---
name: modern-admin-upload-adapters
description: >
  Connect and configure file-storage adapters for Modern Admin resources.
  Use this skill when adding single or multi-file fields with `uploadFeature`,
  choosing or configuring `LocalUploadProvider` or `S3UploadProvider`,
  implementing an `IUploadProvider` for another storage backend, mounting
  `ModernAdminUploadModule`, enabling `uploadGraphqlExtension`, handling file
  children in `jsonByKeyFeature`, storing generated media in an upload-enabled
  property, or diagnosing upload, preview, cleanup, size, MIME, and
  multi-replica problems.
license: MIT
compatibility: Modern Admin (@modern-admin/*) 0.x
metadata:
  author: modern-admin
  version: "1.0.0"
allowed-tools: Bash Read Write Edit Glob Grep
---

# Modern Admin — upload adapters

Treat `@modern-admin/*` as read-only and use only public exports. Inspect the
host's resource definition, database field shape, Nest bootstrap, deployment
replica count, and URL-serving model before choosing a provider.

## Choose the storage model

- Use `LocalUploadProvider` only for development or a genuinely persistent
  single-node filesystem. Mount `uploadDir` as static files yourself.
- Use `S3UploadProvider` for S3-compatible object storage. Prefer a public CDN
  base URL for the built-in file renderer.
- Implement `IUploadProvider` for GCS, Azure Blob, a media service, encrypted
  storage, or another backend. Read `./references/providers.md` before doing so.
- Store the returned storage **key**, not file bytes, in the resource field.
  Use a nullable string for one file and a database shape that round-trips a
  `string[]` for `isArray: true`.

## Wire the REST upload path

Install the feature and the optional Nest transport dependency with bun:

```sh
bun add @modern-admin/feature-upload busboy
```

Register the upload module next to a global Modern Admin module:

```ts
import { ModernAdminModule } from "@modern-admin/nest";
import { ModernAdminUploadModule } from "@modern-admin/feature-upload/nest";

@Module({
  imports: [
    ModernAdminModule.forRoot({ global: true /* adapters, auth, resources */ }),
    ModernAdminUploadModule.forRoot({
      maxFileSize: 25 * 1024 * 1024,
      maxFiles: 20,
      pendingTtlMs: 60 * 60 * 1000,
      // Set only after verifying this deployment has one Nest process.
      acknowledgeSingleInstance: true,
    }),
  ],
})
export class AdminModule {}
```

`ModernAdminUploadModule` owns the authenticated REST upload/cancel endpoints
and the orphan sweeper. Do not write a second multipart controller. The current
transport buffers each bounded file in memory; keep the module hard cap and
the request file-count cap proportional to available memory.

## Attach a provider to a resource

Add `uploadFeature(...)` to `@AdminResource({ features: [] })`; do not set
`type: 'file'` by hand because that omits provider registration and cleanup
hooks.

```ts
import { extname, resolve } from "node:path";
import { uuidv7 } from "@modern-admin/core";
import { LocalUploadProvider, uploadFeature } from "@modern-admin/feature-upload";

const files = new LocalUploadProvider({
  uploadDir: resolve(process.cwd(), "uploads"),
  baseUrl: "/uploads",
});

@AdminResource({
  source: () => adminSource("products"),
  features: [
    uploadFeature({
      properties: {
        thumbnail: {
          provider: files,
          mimeTypes: ["image/*"],
          maxSize: 10 * 1024 * 1024,
          uploadPath: (filename) => `products/thumbnails/${uuidv7()}${extname(filename)}`,
        },
        gallery: {
          provider: files,
          isArray: true,
          mimeTypes: ["image/*"],
          maxSize: 10 * 1024 * 1024,
          uploadPath: (filename) => `products/gallery/${uuidv7()}${extname(filename)}`,
        },
      },
    }),
  ],
})
export class ProductsAdminController extends AdminController<ProductRow> {}
```

For local storage, expose the same directory at `baseUrl`:

```ts
app.useStaticAssets(resolve(process.cwd(), "uploads"), { prefix: "/uploads" });
```

Use a relative `baseUrl` for same-origin deployments. When the SPA and storage
origins differ, use an absolute CDN URL and configure the storage CORS policy.

## Respect the built-in lifecycle

`uploadFeature` automatically:

1. marks configured properties as `type: 'file'`;
2. registers deterministic provider ids per resource/property;
3. confirms pending keys after successful `new` and `edit` actions;
4. deletes replaced or removed keys after `edit`;
5. deletes referenced keys after `delete` (including built-in bulk delete,
   which delegates through per-record delete hooks).

Keep custom feature/action hooks chainable and idempotent. Do not manually
delete the same key in an additional hook unless ownership is intentionally
transferred away from `uploadFeature`.

When using `mediaGenerationFeature`, point `targetProperty` at a property in
this same `uploadFeature` configuration. The media-generation service resolves
its `uploadProviderId` and stores the downloaded result through that provider.

## Apply security and deployment constraints

- Treat `mimeTypes` as a UX and declared-`Content-Type` allowlist, not content
  sniffing. Add signature inspection, malware scanning, image decoding, or
  quarantine in the provider/application when the threat model requires it.
- Generate keys from trusted prefixes plus `uuidv7()` and a derived extension.
  Never preserve an arbitrary client path. Built-in providers reject absolute
  and traversal keys; custom providers must provide equivalent containment.
- Keep `maxSize` at or below the transport-wide `maxFileSize`. REST defaults
  to 25 MiB per file and 20 files for array fields.
- Treat the current pending-upload registry and sweeper as **single-process
  only**. Do not run uploads behind multiple Nest replicas: a save on replica B
  cannot confirm a key tracked on replica A, so A may later delete a valid file.
  Route upload, cancel, and the subsequent `new`/`edit` save through the same
  single process; pinning only the multipart endpoint is insufficient. Sticky
  sessions do not make failover safe.
- Remember that the REST controller currently requires authentication but does
  not add a separate per-role upload capability. Add a host guard/policy layer
  if upload permission must be stricter than access to the admin session.

## Load detailed references only when needed

- Read `./references/providers.md` for Local/S3 options, current signed-URL
  rendering limitations, and a custom `IUploadProvider` implementation.
- Read `./references/transports-and-lifecycle.md` for GraphQL wiring,
  `jsonByKeyFeature` file children, pending cleanup, endpoints, limits, and a
  troubleshooting matrix.

## Verify before finishing

- [ ] Upload, preview, save, reload, replace, remove-before-save, and record
      delete all behave correctly for both single and array properties.
- [ ] The database stores provider keys and the provider resolves those keys to
      reachable URLs after a fresh page load.
- [ ] Disallowed MIME declarations, oversized files, excess file counts, and
      unsafe custom keys are rejected server-side.
- [ ] Local files cannot escape `uploadDir`; object-store credentials remain
      server-side and least-privileged.
- [ ] The deployment is one Nest process, or the upload feature is withheld
      until pending state is moved to a shared store.
- [ ] `bun run typecheck` and tests covering the provider and resource hooks pass.
