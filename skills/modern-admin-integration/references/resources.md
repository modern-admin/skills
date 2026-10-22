# Resources, properties, hide-vs-expose

## Adding a resource — the canonical recipe

Every resource in this framework is built from **three** pieces:

1. A **source registry** — one per host app, maps logical resource ids
   to adapter-specific raw source objects.
2. A **resource controller** — extends `AdminController<RowType>`,
   carries `@AdminResource(...)` metadata, hooks (`@Before`/`@After`)
   and custom actions (`@Action`).
3. A **NestJS module** that registers the controller.

### Source registry — host-owned, never imported from the reference app

`@modern-admin/app-shared` is a private workspace package used only by the
framework's own reference apps. It is not installable by consumers. Keep the
small registry implementation in the host project:

```ts
// src/admin/source-registry.ts
export type AdminSourceFactory = () => unknown;

const registry = new Map<string, AdminSourceFactory>();

export const registerAdminSources = (entries: Record<string, AdminSourceFactory>): void => {
  for (const [id, factory] of Object.entries(entries)) {
    registry.set(id, factory);
  }
};

export const adminSource = (id: string): unknown => {
  const factory = registry.get(id);
  if (!factory) {
    throw new Error(`[admin] no source registered for resource "${id}"; ` + "register it before NestJS bootstrap");
  }
  return factory();
};
```

Then register the adapter-specific sources in one boot-time file:

```ts
// src/admin-sources.ts (Prisma host)
import type { PrismaResourceConfig } from "@modern-admin/adapter-prisma";
import { registerAdminSources } from "./admin/source-registry.js";
import { dmmf, prisma } from "./db.js";

const lowerFirst = (s: string): string => (s.length === 0 ? s : s.charAt(0).toLowerCase() + s.slice(1));

// Prisma model name → logical resource id.
// Keep this map in sync with `registerAdminSources` below.
const MODEL_TO_LOGICAL: Record<string, string> = {
  MaUser: "admins",
  MaRole: "roles",
  Product: "products",
  Order: "orders",
  // …one entry per managed model
};

const buildPrismaSource =
  (modelName: string, logicalId: string): (() => PrismaResourceConfig) =>
  () => {
    const model = dmmf.datamodel.models.find((m) => m.name === modelName);
    if (!model) {
      throw new Error(
        `[admin] Prisma model "${modelName}" not found in DMMF — ` +
          `did you forget \`bun run prisma:generate\` after editing schema.prisma?`,
      );
    }
    // CRITICAL: rewrite relation field `type` from Prisma model name to
    // logical id, so the adapter's FK→reference map resolves the right
    // resource. Skip this and reference autocompletes, dashboard charts,
    // and `relatedResources` lookups will silently fail.
    const fields = model.fields.map((f) => {
      if (f.kind !== "object") return f;
      const mapped = MODEL_TO_LOGICAL[f.type];
      return mapped ? { ...f, type: mapped } : f;
    });
    return {
      model: { ...model, name: logicalId, fields },
      client: prisma,
      clientKey: lowerFirst(modelName),
      enums: dmmf.datamodel.enums,
      // 0.10 defaults to 'pg'. Set 'mysql' or 'sqlite' explicitly for those DBs.
      dialect: "pg",
    };
  };

registerAdminSources({
  admins: buildPrismaSource("MaUser", "admins"),
  roles: buildPrismaSource("MaRole", "roles"),
  products: buildPrismaSource("Product", "products"),
  orders: buildPrismaSource("Order", "orders"),
});
```

Import this file **before** `NestFactory.create` in `main.ts`:

```ts
import "./admin-sources.js"; // side-effect: populates the registry
import { NestFactory } from "@nestjs/core";
```

### Resource controller

```ts
// src/admin/products/products.controller.ts
import {
  Action,
  AdminController,
  AdminResource,
  Before,
  type AdminActionContext,
  type RecordActionResponse,
} from "@modern-admin/nest";
import { adminSource } from "../source-registry.js";

interface ProductRow {
  id: string;
  name: string;
  slug: string;
  price: number;
  stock: number;
  published: boolean;
  publishedAt: Date | null;
  updatedAt: Date;
}

@AdminResource({
  source: () => adminSource("products"),
  // Stable metadata keys; runtimeConfig.metadataTranslations supplies labels.
  navigation: { icon: "Package", group: "catalog" },
  titleProperty: "name",
  listProperties: ["name", "price", "stock", "published", "updatedAt"],
  // `filterProperties` is a whitelist + order for the Filters sheet. Always
  // lead with "id": the default filter view drops id (`isVisibleIn('filter')`
  // is false for the id property), so without this the panel offers no way to
  // look a row up by its primary key.
  filterProperties: ["id", "published"],
  sort: { sortBy: "updatedAt", direction: "desc" },
  // Tabs on the show page — one entry per FK relation pointing AT this
  // resource. Inspect the Prisma model: every `Foo[]` or `Foo?` field
  // is a candidate. Skip relations that already show as a `reference`
  // property on the parent side.
  relatedResources: [{ resourceId: "orderItems", foreignKey: "productId" }],
  properties: {
    id: { isVisible: { new: false, edit: false } },
    updatedAt: { isVisible: { new: false, edit: false } },
  },
})
export class ProductsAdminController extends AdminController<ProductRow> {
  @Before("new")
  @Before("edit")
  fillSlug(ctx: AdminActionContext<ProductRow>): void {
    if (!ctx.payload.slug && typeof ctx.payload.name === "string") {
      ctx.payload.slug = ctx.payload.name
        .toLowerCase()
        .replace(/[^a-z0-9]+/g, "-")
        .replace(/(^-|-$)/g, "");
    }
  }

  @Action({
    actionType: "record",
    name: "publish",
    component: null,
    isVisible: (core) => core.record?.params.published !== true,
    custom: { icon: "Send" },
  })
  async publish(ctx: AdminActionContext<ProductRow>): Promise<RecordActionResponse> {
    const record = ctx.record!;
    await record.update({ published: true, publishedAt: new Date() });
    return {
      record: record.toJSON(),
    };
  }
}
```

### NestJS module

```ts
// src/admin/products/products.module.ts
import { Module } from "@nestjs/common";
import { ProductsAdminController } from "./products.controller.js";

@Module({ controllers: [ProductsAdminController] })
export class ProductsAdminModule {}
```

### Wire it up

```ts
// src/admin.module.ts
imports: [
  ModernAdminModule.forRoot({
    /* … */
  }),
  ProductsAdminModule, // ← one line per resource
];
```

Then restart the dev server (`scripts/dev.sh restart api-prisma`).
Note: `bun --watch` does NOT pick up changes in symlinked workspace
packages, so changes inside `packages/*` always need a restart.

### Virtual related tables and relation-path filters

**Availability:** Chained relation paths are unreleased. Check the host's
installed framework implementation before applying them; the 0.10 baseline
alone does not imply support.

Use `relatedResources` to reuse an existing resource's list action as a
show-page table. A scalar `foreignKey` path can follow several references:

```ts
// In the customers resource's @AdminResource options:
relatedResources: [
  { resourceId: "posts", foreignKey: "authorId" },
  { resourceId: "comments", foreignKey: "authorId" },
  {
    resourceId: "comments",
    foreignKey: "postId.authorId",
    label: "common:commentsOnPosts",
  },
],
```

The third table contains comments on the customer's posts, regardless of
who wrote the comments. `comments.postId` must reference `posts`, and
`posts.authorId` must reference `customers` using the host's logical ids.
Use scalar property names, not Prisma relation-field names. The table locks
the path to the current record id with exact equality; filtering, pagination
and export retain that lock. `showRelatedResources: false` hides the section.

Expose fields through those references in the comments resource:

```ts
filterProperties: ["id", "postId", "postId.authorId", "postId.authorId.tier"],
```

Keep `id` first. The terminal property's type, reference picker and available
values are inferred. These extra properties appear only in the filter view
and cannot be sorted. Override filter metadata using the full path in
`properties`; translate it using the same key in
`runtimeConfig.metadataTranslations.<locale>.resources.comments.properties`.
Translate distinct tabs of the same resource with
`relatedResources["comments::postId.authorId"]`; `relatedResources.comments`
is a fallback. Explicit tab labels also accept dictionary keys.

REST and `admin.invoke()` accept structured criteria, for example
`{ "postId.authorId.tier": { operator: "eq", value: "pro" } }` in
`query.filters`. This API does not add dotted fields to GraphQL's typed
`where` inputs. Conditions on the same reference are ANDed against the same
related row; a missing relation never matches, even for terminal `empty`.

Every intermediate list goes through `invoke()` with the same principal,
list permissions, property access checks and row-scoping hooks. Relation
reads bypass action/REST response caches. There are limits of five scalar
reference hops and 10,000 matching intermediate records per hop; broader
matches fail validation rather than returning a partial table. Traverse the
child/junction resource's scalar FKs for reverse-array or m2m relationships.
Keep these checks intact when adding custom handlers or hooks.

### Resource-creation checklist (apply to EVERY new resource)

For each Prisma model you turn into a resource:

1. **Add to `MODEL_TO_LOGICAL`** in `admin-sources.ts` and register via
   `buildPrismaSource('ModelName', 'logical-id')`. Logical ids are
   lowercase plural; Prisma model names are PascalCase singular.
2. **Define a `<Model>Row` TypeScript interface** matching the row
   shape so hook params get typed.
3. **Extend `AdminController<Row>`** — body-less classes are a smell;
   they leave you unable to add hooks/actions later without rewriting
   every reference.
4. **For each FK column** on the model: confirm it lands as a
   `reference` property (auto-detected). For each "reverse" relation
   (`Foo[]` or `Foo?`), decide whether it deserves a `relatedResources`
   entry on the show page. Skip only when the related rows are an
   implementation detail.
5. **For each sensitive column** (password hash, API key, refresh
   token, internal secret): set `isAccessible: false` — NOT just
   `isVisible: false`. See _What to hide vs. expose_ below.
6. **For each "operation"** the domain expert mentions (publish,
   approve, regenerate, archive, send, retry, discard): add a custom
   `@Action({ actionType: 'record' | 'bulk' | 'resource' })` method.
   Built-in `edit` is rarely the right place for business state
   transitions. **This step is mandatory** — a resource without any
   custom `@Action` on a transactional model (`Order`, `Review`,
   `Task`, `Subscription`, …) is almost certainly under-spec'd.

   **Verb → actionType mapping:**

   | Verb in spec                       | `actionType` | Notes                       |
   | ---------------------------------- | ------------ | --------------------------- |
   | Publish, archive, approve, lock    | `record`     | Per-row state transition    |
   | Resend, retry, regenerate, refresh | `record`     | Idempotent retry on one row |
   | Test credentials, ping, validate   | `record`     | Diagnostics on one row      |
   | Approve selected, archive selected | `bulk`       | Same verb, multi-row        |
   | Export, import, recompute totals   | `resource`   | Toolbar-level, no row scope |
   | Force-sync from external API       | `resource`   | Cross-row maintenance       |

   Skeleton:

   ```ts
   @Action({
     actionType: 'record',
     name: 'approve',
     component: null,                        // server-only, no custom UI
     isVisible: (core) =>                    // hide when not applicable
       core.record?.params.state === 'AWAITING_HUMAN',
     custom: { icon: 'CheckCircle', label: 'Approve' },
     // guard: 'confirmApprove',             // optional confirm prompt
   })
   async approve(ctx: AdminActionContext<Row>): Promise<RecordActionResponse> {
     const r = ctx.record!
     await r.update({ state: 'AUTO_SENDING', /* … */ })
     return { record: r.toJSON(), notice: { message: 'Approved', type: 'success' } }
   }
   ```

7. **For each derived/normalised column** (slug, search-tsvector,
   `updatedBy`): add a `@Before('new')`/`@Before('edit')` hook.
8. **For each "this shouldn't ever happen" invariant** (third-party id
   must not be hand-edited, audit row must not be mutated): pin it
   with `isAccessible: false` on the relevant property/action. Do NOT
   use code-pinned `isAccessible` for ordinary role gating — see
   permissions.md §6a.
9. **Declare `filterProperties` with `'id'` first.** Every resource gets an
   explicit `filterProperties` whitelist and `'id'` is always its first
   entry. The id property is invisible in the `filter` view by default
   (`isVisibleIn('filter')` returns `property.isVisible() && !property.isId()`),
   so a resource that omits the whitelist — or declares one without `id` —
   offers no way to filter by primary key at all. Looking a row up by an id
   copied from a log line, a webhook payload or a support ticket is the most
   common admin task, so this is not optional. A uuid has no distinct values
   to offer, so it renders as a free-text input and the applied filter lands
   in the URL as `filters[id][operator]=co&filters[id][value]=…` — shareable
   as a link.

   ```ts
   // ✅ id first, then the domain filters that matter
   filterProperties: ["id", "status", "customerId", "createdAt"],

   // ❌ no filterProperties at all → the Filters sheet has no Id field
   // ❌ filterProperties: ['status'] → same, id is still unfilterable
   ```

## Property type selection matrix

When deciding `type` (or letting it be inferred), use this table.
Inferred types are usually correct — override only when the database
column type is ambiguous.

| DB shape / intent                       | Use `type:`               | Reason                                                       |
| --------------------------------------- | ------------------------- | ------------------------------------------------------------ |
| Plain short text                        | `string`                  | Default text input                                           |
| Multi-paragraph body                    | `textarea`                | Resizable textarea                                           |
| HTML body, blog post                    | `richtext`                | TipTap editor with toolbar                                   |
| Markdown body, README-style             | `markdown`                | Markdown editor with preview                                 |
| Integer count, ID                       | `number`                  | Number input, integer step                                   |
| Decimal price                           | `money` or `currency`     | Formatted with currency symbol                               |
| Float without currency                  | `float`                   | Decimal number input                                         |
| `true/false` flag                       | `boolean`                 | Switch / checkbox                                            |
| Date only (no time)                     | `date`                    | Date picker                                                  |
| Date + time                             | `datetime`                | Date-time picker                                             |
| Closed enum (Prisma enum / string list) | `enum`                    | Select; `availableValues` required                           |
| FK to another resource                  | `reference`               | Auto-detected; set `reference: 'id'` to override             |
| Many-to-many through join table         | `m2m`                     | Requires `@modern-admin/feature-m2m`                         |
| Free-form JSON, known keys              | `json` + `keyValueFields` | Friendly per-key editor                                      |
| Free-form JSON, unknown keys            | `json`                    | Raw JSON editor                                              |
| File upload (single)                    | `file`                    | Use `uploadFeature`; setting only the type is insufficient   |
| Multiple files                          | `file` + `isArray: true`  | Configure with `uploadFeature` and an array-capable DB field |
| Phone number                            | `phone`                   | International phone input                                    |
| Color picker                            | `color`                   | Hex color swatch                                             |
| Password (hash stored)                  | `password`                | Hidden by default; use `feature-password`                    |
| UUID id                                 | `uuid`                    | Monospace, read-only                                         |
| Image preview from URL                  | `previewMedia`            | Renders thumbnail in list                                    |

Heuristic checklist for each property:

1. Is the column a Prisma enum? → leave inferred (`enum`).
2. Is the column a JSON column with a known schema? → `keyValueFields`.
3. Is it a price/amount? → `money` (locale-aware) over raw `number`.
4. Does the column store an FK? → leave inferred (`reference`).
5. Is the field name `password`/`apiKey`/`secret`/`token`? → `password`
   type AND `isVisible: false` on list/show. Mutation through a custom
   action only.

## What to hide vs. expose

**Hide everywhere** (`isVisible: false` plus `isAccessible: false`
when leaks would be sensitive):

- Hashed passwords, OAuth refresh tokens, API keys, internal IDs that
  are not stable contracts, audit columns the admin should not edit
  (`createdAt`, `updatedAt` — show on `show`/`list`, hide from `edit`).
- Any column the host backend treats as a foreign system's authority
  (Stripe customer id, Sentry org id, …) unless the admin explicitly
  needs to debug it. In that case: `isVisible: { show: true }` only.

**Hide from list, keep on show** (`isVisible: { list: false }`):

- Long text fields, descriptions, blobs of JSON, attached files
  metadata — they explode the list table width.

**Hide from create and edit** (`isVisible: { new: false, edit: false }`):

- Server-computed fields: `id`, `createdAt`, `updatedAt`, derived
  totals, denormalised counters.

`new` visibility falls back to `edit` when omitted. Set both explicitly when
documenting an invariant, and use `newProperties` when the creation form needs
a whitelist/order different from `editProperties`.

**Defaults already provided** by `BaseProperty.isVisible()`:

- Anything matching `/password/i` is hidden by default.
- IDs are not editable by default (`isEditable() === false`).
- IDs are **not filterable by default** either — the `filter` view excludes
  the id property. That default is the one you always override: put `'id'`
  first in `filterProperties` on every resource (checklist step 9).

Do not "hide" by removing properties from `listProperties` only — that
just changes the default column set; the field is still in the
response payload. Use `isAccessible: false` if the field must never
leave the server.
