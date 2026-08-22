# Auth — what to wire and what NOT to wire

Modern Admin targets Better Auth 1.7+. Its account identity is the unique pair
`(issuer, accountId)`: `issuer` and the provider-side `accountId` are required,
and `userId` remains an indexed foreign key. Credential rows must use the full
tuple `providerId: 'credential'`, `issuer: 'local:credential'`, and
`accountId: user.id`. Checking `providerId` alone is not sufficient.

For a populated Better Auth 1.6 database, stop authentication writes before
deploying 1.7. Use the transactional migration shipped by
`@modern-admin/system-prisma` (or its `system-drizzle` sibling): it adds
`issuer` nullable, inventories providers, backfills only explicit verified
mappings, checks projected `(issuer, accountId)` collisions, then applies the
non-null column and unique constraint. Unknown providers and collisions must
abort; never infer issuers from email/profile/request data or merge/delete
users automatically.

`@modern-admin/auth-better-auth` is the only supported auth provider.
Better Auth is already mounted by the scaffold. The agent's job:

- Enable the login strategies the host project needs (email/password,
  OAuth, passkey, magic link) in `src/auth.ts`.
- Add `plugins: [admin({...}), apiKey({...})]` to Better Auth — these
  power role gating and machine-to-machine access. **The `admin()`
  plugin is mandatory if you use `rolesResourceId`** — see §11e.
- Never re-implement login pages, sessions, or password hashing.
- Never invent custom JWT logic — Better Auth + cookie session is
  the path.

API keys for headless access:

```ts
import { apiKey } from "@better-auth/api-key";

plugins: [
  apiKey({
    apiKeyHeaders: "x-api-key",
    requireName: true,
    enableSessionForAPIKeys: true,
    schema: { apikey: { modelName: "MaApiKey" } },
  }),
];
```

Note `modelName: 'MaApiKey'` (PascalCase Prisma model), not the
physical table `ma_api_key`. The `@@map` directive handles the rename.

## 11a. Wiring `BetterAuthProvider` — direct import works, `globalThis` needs a structured cast

When `auth` is imported directly from a sibling module that exports the
result of `betterAuth({...})`, no cast is required and the construction
typechecks cleanly:

```ts
// ✅ Direct import: no cast needed.
import { BetterAuthProvider } from "@modern-admin/auth-better-auth";
import { auth } from "./auth.js";

const authProvider = new BetterAuthProvider({ auth });
```

The only situation that needs a cast is the indirect lookup pattern
the reference apps use, where the Better Auth instance is published
onto `globalThis` at import time and read back as `unknown` to break a
circular import. There the cast target is **structural**, not
`as never`:

```ts
// apps/_shared/src/admin/build-auth-provider.ts (reference)
import { BetterAuthProvider, type BetterAuthProviderOptions } from "@modern-admin/auth-better-auth";

const auth = (globalThis as { __betterAuth?: unknown }).__betterAuth;
return new BetterAuthProvider({
  auth: auth as BetterAuthProviderOptions["auth"],
});
```

`as never` works as a sledgehammer but loses the contract entirely —
prefer `BetterAuthProviderOptions['auth']` so future drift in the
instance shape produces a real error instead of being silently
swallowed.

## 11b. Wiring `RedisCacheProvider` — pass a `client`, not a URL

`RedisCacheOptions` expects a Redis-like **client object**
(`packages/cache-redis/src/index.ts`), not a connection string:

```ts
// ❌ TS2353: Object literal may only specify known properties,
//    and 'url' does not exist in type 'RedisCacheOptions'.
new RedisCacheProvider({ url: process.env.REDIS_URL });
```

Construct an `ioredis` client and pass it as `client`. For pub/sub
invalidation you also need a dedicated `subscriber` client (ioredis
won't multiplex pub/sub on a connection that's also serving commands):

```ts
import Redis from "ioredis";
import { RedisCacheProvider } from "@modern-admin/cache-redis";

const cacheProvider = process.env.REDIS_URL
  ? new RedisCacheProvider({
      client: new Redis(process.env.REDIS_URL),
      subscriber: new Redis(process.env.REDIS_URL),
    })
  : undefined;
```

Same constructor pattern for `RedisRealtimeBus` in
`@modern-admin/realtime`. The cache and bus may share the _command_
client but must each have their own _subscriber_.

## 11c. Serving the SPA — `ModernAdminStaticUiModule` is REQUIRED

Mounting the REST API at `/admin/api/*` is **not enough**. Without an
SPA mount, hitting `/admin` returns `404 Not Found` and a hard refresh
of any in-app route (e.g. `/admin/resources/users`) also 404s — the
exact symptom users report as "main page redirects to `/` but a refresh
breaks". The canonical scaffold imports `ModernAdminStaticUiModule`
from `@modern-admin/nest` and serves the prebuilt `@modern-admin/web`
bundle alongside the API:

```ts
// src/app.module.ts (or admin.module.ts — either works)
import { ModernAdminStaticUiModule } from "@modern-admin/nest";

@Module({
  imports: [
    AdminModule,
    ModernAdminStaticUiModule.forRoot({
      path: "/admin", // must match the API prefix root
      title: "Acme Admin",
      runtimeConfig: {
        apiUrl: "", // same-origin — relative URLs
        credentials: "include",
        // authBasePath: '/admin/api/auth',  // default; override only if you
        // mount Better Auth elsewhere
      },
    }),
  ],
})
export class AppModule {}
```

The module installs an Express middleware that:

- streams `assets/*` directly from `@modern-admin/web/dist/standalone/`,
- rewrites the build's relative `./assets/...` to absolute
  `${path}/assets/...` so deep links survive, and
- serves the SPA shell with `window.__MODERN_ADMIN__` injected from
  `runtimeConfig` for **every** unknown sub-path under `${path}` —
  i.e. browser-history routes like `/admin/resources/users/edit/<id>`
  refresh cleanly instead of 404-ing.

The middleware _excludes_ `${path}/api/*` so the regular admin REST
controllers keep handling API traffic.

The mount path cannot be `/` in 0.7; the module rejects it at boot because a
root SPA mount would swallow the fixed `/admin/api/*` routes.

### 11c.1. Modern Admin 0.7 runtime config, branding, and translations

Use stable metadata keys in resource decorators and localize them through
`runtimeConfig.metadataTranslations`. Use `translations` for framework-owned
UI strings. `title` changes the HTML document title; `brand.title` changes the
product name rendered inside the SPA.

```ts
import { join } from "node:path";

ModernAdminStaticUiModule.forRoot({
  path: "/admin",
  title: "Acme Admin",
  themeCss: ":root{--primary:221 83% 53%}.dark{--primary:217 91% 60%}",
  faviconUrl: "/admin/brand/favicon.svg",
  staticFiles: {
    "/brand/favicon.svg": join(process.cwd(), "brand/favicon.svg"),
    "/brand/logo.svg": join(process.cwd(), "brand/logo.svg"),
  },
  // Static string or a request-aware async factory. Host-authored markup only;
  // never interpolate user input because headHtml is injected verbatim.
  headHtml: '<meta name="theme-color" content="#2563eb">',
  runtimeConfig: {
    apiUrl: "",
    credentials: "include",
    brand: { title: "Acme Admin", logoUrl: "/admin/brand/logo.svg" },
    defaultLocale: "en",
    fallbackLocale: "en",
    locales: ["en", "ru"],
    // forceLocale: 'ru', // pins one locale and hides the switcher
    translations: {
      en: { "common:appName": "Acme Admin" },
      ru: { "common:appName": "Админ-панель Acme" },
    },
    metadataTranslations: {
      en: {
        navigation: { groups: { catalog: "Catalog" } },
        resources: {
          products: { actions: { publish: { label: "Publish" } } },
        },
      },
      ru: {
        navigation: { groups: { catalog: "Каталог" } },
        resources: {
          products: { actions: { publish: { label: "Опубликовать" } } },
        },
      },
    },
  },
});
```

`runtimeConfig` may also be a request-aware sync/async factory. Prefer that
for per-tenant branding or locale selection. Keep `staticFiles` values as
trusted absolute paths owned by the host.

### 11c.2. Bootstrap config is authenticated by default

In 0.7, `GET /admin/api/config` requires a session. This is the safe default:
the snapshot contains resource ids, property paths, enum values, and feature
flags. The stock login screen can use `runtimeConfig.brand` without opening
the endpoint.

Set `publicConfig: true` only if the unauthenticated screen must fetch server-
derived config:

```ts
ModernAdminModule.forRoot({
  // ...database, auth, resources, stores
  publicConfig: true,
});
```

Public responses are still evaluated with a null principal and filtered by
resource/action/property access predicates. Treat the option as schema
disclosure, not as a bypass for those predicates.

### 11c.3. `forRootAsync` must declare AI synchronously

Nest fixes controllers and imports before `useFactory` resolves. Therefore
the outer `aiAssistant` boolean must exactly match whether the returned
options contain an `aiAssistant` block:

```ts
ModernAdminModule.forRootAsync({
  aiAssistant: true,
  useFactory: async () => ({
    // ...database, auth, stores
    aiAssistant: {
      enabled: true,
      apiKey: process.env.OPENAI_API_KEY,
    },
  }),
});
```

When enabled, also configure the host's root `BullModule.forRoot(...)`.
If AI is disabled, omit both the outer flag and the returned block. Either
mismatch now fails at boot instead of exposing a half-configured feature.

## 11d. Mounting Better Auth — use `createBetterAuthMiddleware`, not bare `toNodeHandler`

`toNodeHandler(auth)` is greedy: it intercepts **every** path under its
mount prefix and returns its own `404` for paths it doesn't own. When
mounted at `/admin/api/auth` this shadows three NestJS endpoints that
`@modern-admin/nest`'s `AuthController` owns:

| path                            | owner                                                   |
| ------------------------------- | ------------------------------------------------------- |
| `POST /admin/api/auth/login`    | `AuthController` — records login event, returns session |
| `GET  /admin/api/auth/me`       | `AuthController` — session bootstrap for the SPA        |
| `GET  /admin/api/auth/ui-props` | `AuthController` — public auth config for the SPA       |

A bare `app.use('/admin/api/auth', toNodeHandler(auth))` causes
`POST /admin/api/auth/login 404` — Better Auth handles it first and
returns its own 404 before NestJS sees the request.

**Always use `createBetterAuthMiddleware` instead:**

```ts
import { toNodeHandler } from "better-auth/node";
import { createBetterAuthMiddleware } from "@modern-admin/nest";

// main.ts — BEFORE any body parser:
app.use("/admin/api/auth", createBetterAuthMiddleware(toNodeHandler(auth)));
```

`createBetterAuthMiddleware` wraps the given handler and calls `next()`
for `/me`, `/login`, and `/ui-props`, letting NestJS handle those three
paths while routing everything else (sign-in, sign-out, session, etc.)
to Better Auth.

### `authBasePath` — only override when Better Auth lives elsewhere

The SPA's sign-in form posts to `${authBasePath}/sign-in/email`. The
default `authBasePath` is `/admin/api/auth`, matching the canonical
scaffold where `main.ts` does
`app.use('/admin/api/auth', createBetterAuthMiddleware(toNodeHandler(auth)))`
and `auth.ts` sets `betterAuth({ basePath: '/admin/api/auth' })`.
As long as those three values agree, you do not have to set anything.

Override `authBasePath` when — and only when — you intentionally mount
Better Auth at a non-default path (e.g. you're embedding the admin
inside a host app that already owns `/api/auth/*` for end-user auth):

```ts
ModernAdminStaticUiModule.forRoot({
  path: "/admin",
  runtimeConfig: {
    authBasePath: "/api/auth", // host already owns this — share the handler
  },
});
```

If you change the auth basePath, change all three coordinated values:

- `app.use(<basePath>, createBetterAuthMiddleware(toNodeHandler(auth)))` in `main.ts`,
- `betterAuth({ basePath: <basePath> })` in `auth.ts`,
- `runtimeConfig.authBasePath: <basePath>` in the SPA mount.

A mismatch surfaces as `POST /api/auth/sign-in/email 404 Not Found` on
login — the SPA points at the default while the server moved.

## 11e. The `admin()` plugin is mandatory for role gating

`BetterAuthProvider.getCurrentUser()` reads `currentAdmin.role` from
`session.user.role`. That field is **only** populated when Better
Auth's `admin()` plugin is mounted. Without it the session payload
has no `role`, and any of the following silently breaks:

- `rolesResourceId` permission gate — a session with no role is rejected
  by the default-deny anonymous gate before action execution.
- Per-action `isAccessible: ({currentAdmin}) => currentAdmin?.role === 'admin'`
  predicates — always evaluate to `false`, returning **403 Forbidden**
  on every protected action, even for users whose `ma_user.role`
  column is set to `'admin'` in the database.
- `/admin/api/auth/me` — `user.role` is missing from the response, so
  the SPA's permission hint layer hides nothing (UI shows actions the
  server will then 403 on).

The trap is that the database column _is_ populated — `ma_user.role`
will read `'admin'` via Prisma, and the `admins` resource in the
panel renders it correctly. So the bug looks like "permissions are
wrong" when it is in fact "the session never carries the role at
all". A direct DB write (e.g. `prisma.maUser.update({ data: { role:
'admin' }})`) bypasses the plugin and gives you exactly this state.

**Always mount `admin()` when you use `rolesResourceId` or any role
predicate:**

```ts
import { betterAuth } from "better-auth";
import { apiKey } from "@better-auth/api-key";
import { admin } from "better-auth/plugins";

export const auth = betterAuth({
  // … database, baseURL, modelNames …
  plugins: [
    apiKey({
      /* … */
    }),
    admin({
      // 'admin' is convenient for demos so the seeded user can do
      // everything; use 'user' (or your equivalent) in production.
      defaultRole: process.env.DEMO_ADMIN_ROLE ?? "user",
    }),
  ],
});
```

Reference: `apps/api-prisma/src/auth.ts` mounts the plugin via
`extraPlugins:` on `buildBetterAuth()`.

## 11f. `BigInt` columns survive serialisation by default

Prisma surfaces `BigInt` columns as native `bigint`. The framework
normalises those to decimal strings at the `BaseRecord.toJSON()`
boundary, so every list/show response is JSON-stringifiable
end-to-end — both the Express response writer and
`@modern-admin/cache-redis` accept records carrying `BigInt` fields
without throwing `TypeError: JSON.stringify cannot serialize BigInt`.
No host-side workaround is needed.

The frontend therefore receives `BigInt` columns as **strings**. If a
custom UI cell needs numeric maths, parse with `BigInt(str)` (or
`Number(str)` when the value provably fits in `Number.MAX_SAFE_INTEGER`).
Do not patch `BigInt.prototype.toJSON` globally — the framework
already handles this at the right layer.

## 11g. Modern Admin 0.7 store and query contracts

- The shipped `IQueryableLogStore` implementations default `list()` to 50
  rows. Custom stores should apply a bounded default, honour `limit`, and use
  stable keyset paging with the `(before, beforeId)` pair; timestamp-only
  paging can skip rows that share a millisecond.
- `IHistoryStore.prune?(retention)` is the optional retention hook. The shipped
  Prisma store implements it. Hosts may call it from their own scheduled
  retention job; custom stores may implement it or manage retention externally
  (for example, with database TTL policies).
- A client-supplied `sortBy` is validated against each property's
  `isSortable()`. A request for a non-sortable or unknown property returns 400
  before reaching the ORM. The trusted resource-level default is not subject
  to that request check, so keep `sort.sortBy` on a real adapter field.
- The base `streamFind` implementation is offset-based and rejects
  `StreamOptions.cursor`; only pass cursors to adapters that explicitly
  implement them.
