# Modern Admin Skills

AI agent skills for the [Modern Admin](https://github.com/modern-admin/modern-admin)
framework, in the [Agent Skills](https://agentskills.io/) format. They give
AI coding assistants (Claude Code, Cursor, Copilot, Windsurf, Aider,
opencode, Kiro, …) the framework's non-obvious integration rules so you don't
have to re-discover them by reading the source.

Each skill loads lazily based on its `description` frontmatter — there's no
context cost unless a task actually touches Modern Admin.

## Install

In any project where you use an AI coding agent:

```sh
npx skills add modern-admin/skills
```

The CLI clones this repo, lists the skills under `skills/`, and walks you
through:

1. **Pick skills** to install (space to toggle).
2. **Pick agents** — Claude Code, Cursor, GitHub Copilot, Windsurf, Aider,
   opencode, Kiro, …
3. **Pick scope** — project (`./.claude/skills/`, …) or global
   (`~/.claude/skills/`, …).

`npx skills` writes a `skills-lock.json` at your project root with the
resolved source and a content hash for every installed skill, so teammates
and CI restore the same files later:

```sh
npx skills experimental_install   # like `npm ci` — restore from lock
```

Non-interactive, single-skill, or pinned-to-version install:

```sh
npx skills add modern-admin/skills --skill modern-admin-integration --yes
npx skills add modern-admin/skills --skill modern-admin-upload-adapters --yes
npx skills add modern-admin/skills#v2.2.0
```

Claude Code users can also install via the plugin mechanism:

```
/plugin install modern-admin/skills
```

## Skills in this repo

| Skill | What it covers |
|-------|----------------|
| [`modern-admin-integration`](./skills/modern-admin-integration/SKILL.md) | Modern Admin 0.10 wiring: `ModernAdminModule`, resources, Better Auth + Prisma + Redis, runtime i18n/branding, authenticated config, media generation and retention, custom UI, actions/hooks, role permissions, the 14 `Ma*` models, and auth/SPA troubleshooting. |
| [`modern-admin-upload-adapters`](./skills/modern-admin-upload-adapters/SKILL.md) | Connecting Local, S3-compatible, or custom `IUploadProvider` storage; mounting REST/GraphQL upload transports; configuring single/multi-file and JSON-by-key fields; and handling limits, cleanup, security, previews, and deployment constraints. |

## Contributing

See [`AGENTS.md`](./AGENTS.md) for the repo layout, skill file format,
validation, and release process.

## License

MIT. See [`LICENSE`](./LICENSE).
