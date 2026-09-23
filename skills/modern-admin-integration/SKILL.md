---
name: modern-admin-integration
description: >
  Use this skill when integrating, configuring, or extending Modern Admin
  (`@modern-admin/*`) in a host project — i.e. wiring
  `ModernAdminModule.forRoot`, adding admin resources, configuring Better
  Auth/Prisma/Redis, declaring properties or `@Action`/`@Before`/`@After`
  hooks, setting up role permissions (`MaRole.permissions`), or
  troubleshooting auth/SPA 404s. Triggers on tasks that mention
  `@AdminResource`, `AdminController`, `adminSource`, `BetterAuthProvider`,
  `ModernAdminStaticUiModule`, `publicConfig`, `metadataTranslations`,
  `setupPrismaSystem`, `MaRole`,
  `rolesResourceId`, the `ma_*` schema fragment, custom UI components,
  `modern-admin setup-ui`, `mediaGenerationFeature`, `RetentionModule`,
  `createRedisCacheProvider`, or scaffolding `bun create @modern-admin`.
license: MIT
compatibility: Modern Admin (@modern-admin/*) 0.10.x
metadata:
  author: modern-admin
  version: "2.2.1"
allowed-tools: Bash Read Write Edit Glob Grep
---

# Modern Admin — Integration skill

Modern Admin is a vendor framework. Treat `@modern-admin/*` as read-only;
only call its public exports. This skill encodes the project's non-obvious
rules so you don't re-discover them.

## Ground rules (always apply)

1. **bun only.** Never `npm`/`yarn`/`pnpm`.
2. **UUID v7 for every generated id.** `import { uuidv7 } from '@modern-admin/core'`.
   Never `crypto.randomUUID()`, `randomUUID()`, `nanoid`. Don't trust
   `@default(uuid(7))` — generate in app code.
3. **Latest stable versions.** Adapt code to new APIs; never pin older to dodge
   breaking changes.
4. **i18n everywhere.** No hardcoded user-visible strings. Framework changes
   add keys to all 9 locale files and keep `packages/ui` i18n-unaware;
   host-owned custom components use `useI18n()` or a project-owned locale map.
   See `references/conventions.md`.
5. **Mobile-first.** Verify every new UI at ≤375px viewport.
6. **Auto-discovery beats config.** Override only what diverges.
7. **`id` is always filterable.** Every resource declares
   `filterProperties` and `'id'` is its first entry — the framework hides the
   id property from the `filter` view by default, so without the whitelist no
   one can look a row up by primary key. See `references/resources.md`
   checklist step 9.

## Current API baseline — Modern Admin 0.10

This skill targets the latest stable `@modern-admin/*` 0.10 line. The upgrade-
sensitive contracts are:

- `GET /admin/api/config` requires a session by default. Set
  `publicConfig: true` only when the login experience must read the filtered
  config before authentication.
- `ModernAdminModule.forRootAsync({ aiAssistant: true, useFactory })` must set
  the synchronous flag exactly when the factory returns an `aiAssistant`
  block; a mismatch fails at boot.
- Standalone UI branding and locale overrides belong to
  `ModernAdminStaticUiModule` (`themeCss`, `faviconUrl`, `headHtml`,
  `staticFiles`) and `runtimeConfig` (`brand`, `translations`,
  `metadataTranslations`, `forceLocale`).
- Prisma resources default to `dialect: 'pg'`; set `'mysql'` or `'sqlite'`
  explicitly for those databases. The dialect controls generated display SQL
  and whether Prisma's PostgreSQL-only case-insensitive mode is emitted.

See `references/auth-and-infra.md` and `references/resources.md` for complete
examples.

## When to use Modern Admin

Use it for CRUD-over-Postgres admin panels (NestJS 12, Prisma 7 / Drizzle
0.45, Bun 1.4, TypeScript 7) with role-gated actions, search, filters,
uploads, audit log, revisions, webhooks, AI assistant, and media generation.
**Don't** use it for public UI, non-relational stores, or
when admin shouldn't share a DB with main app (use it as a separate service
against a replica instead).

## Deployment model — default to standalone

Recommended layout: separate NestJS 12 service against the same Postgres.
Scaffold with `bun create @modern-admin admin-service` — **no registry token
required**. The scaffolder and all open-core `@modern-admin/*` packages are
public on npm, so a missing `MODERN_ADMIN_TOKEN`/`NPM_TOKEN` is never a
blocker. A token is only needed for **Pro** packages (`@modern-admin-pro/*`:
webhooks, AI-fill, logging) on their restricted registry.
Mount behind `/admin` of the main domain via reverse proxy. See
`references/deployment.md` for the existing-Prisma reuse rules (one schema,
one client; never add a second `generator`) and the **fourteen** `Ma*` models
that must be copied verbatim — `setupPrismaSystem` fails at boot if any are
missing.

## Custom UI — scaffold the standard flow

When a host needs custom property/action components, run
`bunx @modern-admin/create setup-ui` from the host package root instead of
hand-copying the Vite shell. Pass `--api-proxy` when the backend is not on port 3001. The command creates `ui/`, adds host dependencies/scripts, and sets
`ModernAdminStaticUiModule.webPackage` without overwriting existing UI files.
See `references/custom-ui.md`.

## The three pieces of every resource

1. **Host-owned source registry** (`src/admin/source-registry.ts`) plus
   registrations (`src/admin-sources.ts`) — maps logical resource ids to
   adapter-specific raw source objects via `registerAdminSources({...})`.
   Import the registrations as a side effect **before**
   `NestFactory.create` in `main.ts`. Never import the reference app's private
   `@modern-admin/app-shared` workspace package.
2. **Resource controller** — `extends AdminController<Row>` (never body-less),
   carries `@AdminResource(...)` metadata + `@Before`/`@After` hooks +
   `@Action` methods.
3. **NestJS module** — `@Module({ controllers: [FooAdminController] })`.

See `references/resources.md` for the canonical Prisma source factory
(including the **critical** relation-field-`type` rewrite for FK→reference
resolution), the resource-creation 9-step checklist, and the property-type
selection matrix.

## NestJS-style permissions — `actions:` key is FORBIDDEN

> The #1 mistake AI agents make.

`AdminResourceMeta = Omit<ResourceOptions, 'actions'> & {source, ...}`.
Writing `@AdminResource({ actions: {...} })` is a `TS2353`. All action
config goes through `@Action`/`@Before`/`@After` method decorators, or —
for role gating — through `MaRole.permissions` (DB-driven, edited in the
panel). Code-pinned `isAccessible` is reserved for invariants no role may
bypass. See `references/permissions.md` for the role-matrix recipe, true-
invariant `Promise<never>` stubs, base-class signature matching for
overrides (avoid `TS2416`), and the `guard:` vs `isAccessible:` distinction.

## Auth wiring — the eight must-knows

1. **`admin()` plugin is MANDATORY** when using `rolesResourceId`. Without
   it `session.user.role` is `undefined` and every role-gated action returns
   403 — even when `ma_user.role` is `'admin'` in the database.
2. **Mount Better Auth via `createBetterAuthMiddleware(toNodeHandler(auth))`,
   not bare `toNodeHandler`.** The bare handler is greedy and shadows
   `AuthController`'s `/login`, `/me`, `/ui-props` with its own 404s.
3. **`ModernAdminStaticUiModule.forRoot({ path: '/admin', ... })` is
   REQUIRED** to serve the SPA + handle deep-link refreshes. Without it
   `/admin` and every refresh 404s.
4. **Use `createRedisCacheProvider({ url })` for URL-based setup, or pass a
   Redis client to `new RedisCacheProvider({ client, subscriber? })`.** The
   direct constructor has no `url` option; it can duplicate a capable client
   automatically. `RedisRealtimeBus` accepts one `client` and duplicates it.
5. **Keep `app.use(<basePath>, ...)`, `betterAuth({ basePath })`, and
   `runtimeConfig.authBasePath` in lockstep.** A mismatch surfaces as
   `POST /api/auth/sign-in/email 404` on login.
6. **Better Auth 1.7 account identity is `(issuer, accountId)`.** Every
   `MaAccount` schema needs required `issuer`, compound uniqueness on that
   pair, and the `userId` index. Credentials are exactly
   `providerId: 'credential'`, `issuer: 'local:credential'`,
   `accountId: user.id`. Never reuse a 1.6 schema with 1.7.
7. **`GET /admin/api/config` is authenticated by default.** Do not expose the
   resource schema accidentally; opt in with `publicConfig: true` only when
   unauthenticated bootstrap config is required.
8. **`forRootAsync.aiAssistant` must agree with the resolved options.** Set the
   outer boolean when and only when `useFactory` returns an `aiAssistant`
   block, and configure the root BullMQ connection when it is enabled.

Full details + code examples: `references/auth-and-infra.md`.

## References

- `./references/deployment.md` — §2 standalone deploy, §2.5 reusing existing
  Prisma schema, the 14 `Ma*` models, anti-pattern of duplicated generators.
- `./references/resources.md` — §3 the three-pieces recipe, source registry,
  resource controller, 8-step per-resource checklist, §4 property-type matrix,
  §5 hide vs. expose (`isVisible` vs `isAccessible`).
- `./references/permissions.md` — §6 roles × actions, DB-driven vs code-pinned,
  `actions:` forbidden, read-only via roles, base-class signature, true
  invariants, global property redaction, `guard:` vs permissions.
- `./references/actions-and-plugins.md` — §7 built-in actions and when to
  override, §8 plugin/feature/media-generation selection, retention, §10 hooks
  vs actions vs handlers.
- `./references/custom-ui.md` — §9 `modern-admin setup-ui`, component
  registration, `@modern-admin/ui` primitives, i18n, Tailwind 4 rules.
- `./references/auth-and-infra.md` — §11 a–g: `BetterAuthProvider` wiring,
  `RedisCacheProvider`, `ModernAdminStaticUiModule`, `createBetterAuthMiddleware`,
  0.10 runtime config/branding, authenticated bootstrap config,
  `forRootAsync` AI agreement, `admin()` plugin requirement, `BigInt`
  serialisation.
- `./references/conventions.md` — §12 i18n boundary, §13 UUID v7,
  §14 cache + realtime.
- `./references/anti-patterns.md` — §15 full anti-pattern list,
  §16 verification checklist, §17 reference index of canonical files.

## Verification checklist (always run before "done")

- [ ] `bun run dev` (or `scripts/dev.sh start api-prisma web`) starts clean;
      `.dev-logs/` show no errors.
- [ ] Seed admin can log in at `/admin` and sees every registered resource.
- [ ] Every resource's Filters sheet offers an `Id` field (i.e. every
      `filterProperties` array starts with `'id'`).
- [ ] Sensitive columns (`password`, `apiKey`, …) absent from both JSON and UI.
- [ ] Every destructive action has a `guard:` confirmation and is granted only
      to intended roles by `MaRole.permissions`; code-pinned `isAccessible` is
      used only for invariants no role may bypass.
- [ ] Every new visible string exists in all 9 locale files.
- [ ] `bun run typecheck` is green workspace-wide.
- [ ] Tests added for every custom hook, action handler, UI component.
- [ ] 375px viewport: list/show/edit pages have no horizontal page scroll.

## Canonical files to read before writing new code

- `apps/_shared/src/admin/posts/posts.controller.ts` — hooks + record + bulk
  actions.
- `apps/_shared/src/admin/customers/customers.controller.ts` —
  `passwordsFeature` + hooks + custom record action.
- `apps/_shared/src/admin/source-registry.ts` — source registry.
- `apps/api-prisma/src/admin-sources.ts` — logical-id mapping + relation
  field rewriting.
- `apps/api-prisma/src/admin.module.ts` — admin module wiring.
- `packages/core/src/decorators/{resource,property,action}-options.ts` —
  schemas.
- `packages/core/src/actions/action.ts`,
  `packages/core/src/ports/current-admin.ts` — `ActionContext`,
  `CurrentAdmin` types for `isAccessible`/`isVisible` callbacks.
- `packages/nest/src/admin/decorators.ts` — `AdminController`, `@Before`,
  `@After`, `@Action`.

When in doubt, read the source — every public export has JSDoc explaining
its contract.
