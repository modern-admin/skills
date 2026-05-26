# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Cursor, Copilot,
…) when working **inside this repository** — i.e. when editing or adding
skills, not when consuming them in a host project.

## Repository overview

This repository contains AI agent skills for the
[Modern Admin](https://github.com/modern-admin/modern-admin) framework, in
the [Agent Skills](https://agentskills.io/) format. Skills are markdown
documentation bundles that AI coding assistants load on demand to provide
context-aware help with Modern Admin.

## Repository structure

```
skills/
└── <skill-name>/
    ├── SKILL.md       # Main skill file (required) — YAML frontmatter + body
    └── references/    # Supporting documentation (optional)
        └── *.md
```

## Skill file format

Every `SKILL.md` must include YAML frontmatter with:

- `name` — skill identifier (lowercase, hyphenated; must match folder name)
- `description` — multi-line description with explicit trigger conditions
- `license` — `MIT`
- `compatibility` — runtime requirements (e.g. `Modern Admin 0.x`)
- `metadata.author` — `modern-admin`
- `metadata.version` — semver string (`"X.Y.Z"`)
- `allowed-tools` — permitted Claude Code tools (e.g. `Bash Read Write Edit Glob Grep`)

Example:

```yaml
---
name: skill-name
description: >
  What this skill does. Use this skill when:
  (1) first use case,
  (2) second use case.
license: MIT
compatibility: Modern Admin (@modern-admin/*) 0.x
metadata:
  author: modern-admin
  version: "1.0.0"
allowed-tools: Bash Read Write Edit Glob Grep
---
```

## Validation

Skills should be validated against the Agent Skills specification at
<https://agentskills.io/specification>.

Validate locally with `gh skill publish --dry-run` (or
`skills-ref validate skills/<name>`) before opening a PR. CI runs both.

## Adding a new skill

1. Create `skills/<skill-name>/SKILL.md` with the frontmatter above.
2. Add reference files in `skills/<skill-name>/references/` as needed.
3. Update `README.md` to document the new skill.
4. Bump `.claude-plugin/plugin.json#version` (any plugin content change).
5. Open a PR — CI will require both `plugin.json#version` and the per-skill
   `metadata.version` to strictly increase if their content changed.

## Releasing & versioning

Two versions matter.

**`.claude-plugin/plugin.json#version`** is what `claude plugin update` checks.
Bump it whenever you touch `skills/`, `commands/`, `agents/`, `hooks/`, or
`plugin.json` itself — otherwise existing users won't see the change. CI
catches forgotten bumps. New installs always pull the latest content
regardless.

**Per-skill `metadata.version` in `SKILL.md`** is per-skill semver. CI also
requires this to bump when the skill's own content changes.

The git tag is automatic. Pushes to `main` run `release.yml`, which reads
`.claude-plugin/plugin.json#version` and creates `vX.Y.Z` if it doesn't
exist yet. Pins like `npx skills add modern-admin/skills#v2.0.0` resolve
to that tag.

Use semver: patch for fixes, minor for new skills, major when you remove
or rename one.

You can ignore `marketplace.json#metadata.version` and `package.json#version`
— the marketplace barely changes for a single-plugin repo, and `package.json`
isn't published anywhere.
