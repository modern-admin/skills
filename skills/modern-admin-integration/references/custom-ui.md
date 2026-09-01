# Custom UI components — when and how

The frontend ships rich defaults for every property type. Reach for a
custom component only when:

1. The data shape is unique to the host project (e.g. geo-coords
   needing a map).
2. The default display is technically correct but visually wrong
   (e.g. a `status` enum needing colored badges).
3. You need to compose multiple fields into one widget (rare; usually
   `showWhen` is enough).

## Scaffold and connect the host bundle

Run from the package that owns `package.json` and the Nest app:

```sh
bunx @modern-admin/create setup-ui
```

The command creates the standard `ui/` Vite/React bundle, adds the required
host dependencies and `ui:dev`/`ui:build`/`ui:typecheck` scripts, composes the
host `dev`/`build`/`typecheck` scripts, and adds
`webPackage: './ui'` to the existing
`ModernAdminStaticUiModule.forRoot({ ... })` options.

If the backend dev origin differs from the scaffold default:

```sh
bunx @modern-admin/create setup-ui --api-proxy http://localhost:3334
```

Useful overrides: `--dir frontend/admin`, `--module src/app.module.ts`, and
`--base-path /backoffice` (normally inferred from the Nest module). A custom
Better Auth mount is inferred from `runtimeConfig.authBasePath`, or can be set
with `--auth-base-path /api/auth`. The UI directory must stay inside the host
root and cannot traverse symbolic links.
The command refuses to overwrite an unmarked non-empty directory or an
existing custom `webPackage` (the explicit `@modern-admin/web` default may be
replaced safely); re-runs preserve all generated UI files.

Modern Admin 0.10 disables production source maps by default. Keep
`sourcemap: false` for a publicly reachable bundle. Use `sourcemap: 'hidden'`
only when an error tracker needs maps without browser-visible
`sourceMappingURL` comments; `true` exposes the custom bundle's source through
served `.map` assets.

After generation:

```sh
bun install
bun run ui:build
bun run dev
# Optional React Fast Refresh in a second terminal:
bun run ui:dev
```

## Build and register a component

```tsx
// 1. Build the component using @modern-admin/ui primitives.
import type { PropertyDisplayProps } from "@modern-admin/react";
import { Badge } from "@modern-admin/ui";

export function StatusBadge(props: PropertyDisplayProps) {
  const paid = props.value === "paid";
  return <Badge variant={paid ? "success" : "destructive"}>{String(props.value)}</Badge>;
}
```

```ts
// 2. Register in generated ui/src/admin-components.ts.
import { ComponentLoader } from "@modern-admin/react";
import { StatusBadge } from "./components/status-badge.js";

export const adminComponents = new ComponentLoader().add("StatusBadge", StatusBadge);
```

```ts
// 3. Reference by name in resource options.
properties: {
  status: { components: { list: 'StatusBadge', show: 'StatusBadge' } },
}
```

Action pages use the same loader name via `@Action({ component: '...' })`.
The component prop contracts (`PropertyDisplayProps`, `PropertyEditorProps`,
and action contexts) are public exports from `@modern-admin/react`; do not
invent a `record` prop for property display components — their value is
`props.value`.

Host-owned components render inside the framework providers and may import
`useI18n()` from `@modern-admin/react`. Keep project strings in the host's own
locale map and select them by `locale`; reuse `t(...)` for built-in keys.
Only framework primitives inside `packages/ui` follow the separate
`labels?: {...}` rule and must not import `useI18n`.

Always prefer composition of `@modern-admin/ui` primitives
(`Button`, `Badge`, `Card`, `Dialog`, `Sheet`, `Tabs`, `Select`,
`Combobox`, `Field`, `DataTable`) over raw HTML/CSS. They already
follow Tailwind 4 conventions, dark-mode tokens, and shadcn variants.
Pair every `border` className with an explicit color
(`border border-border`) — in Tailwind 4 `border` alone falls back to
`currentColor`.
