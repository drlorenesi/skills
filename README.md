# skills

Personal [Claude Code](https://claude.com/claude-code) skills by Diego Lorenesi, installable via [skills.sh](https://www.skills.sh).

## Install

In the root of a Claude Code project:

```bash
npx skills add drlorenesi/skills
```

When prompted:

1. **Which agents do you want to install to?** → select **Claude Code**.
2. **Installation method** → select **Symlink** (single source of truth, easy updates).

Each skill is symlinked into `.claude/skills/<skill-name>` in your project. Updates land automatically the next time the source repo is pulled.

You can also install a single skill from the repo:

```bash
npx skills add drlorenesi/skills/shadcn-forms
```

## Skills

| Skill                            | Description                                                                                                                                                      |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`shadcn-forms`](./shadcn-forms) | Build type-safe forms with shadcn/ui, react-hook-form, and zod. Covers schema definition, form setup, field wiring, validation display, and submission patterns. |

## Adding a new skill

1. Create a kebab-case folder at the repo root (e.g. `nextjs-server-actions/`).
2. Add a `SKILL.md` inside it with the frontmatter shown under **Conventions** below.
3. Put long-form supporting docs in `rules/*.md` and link to them from `SKILL.md`.
4. Add a row to the **Skills** table above.

## Conventions

Every skill in this repo follows the same shape:

```
<skill-name>/
├── SKILL.md
└── rules/
    └── <topic>.md   ← optional, for long-form rules linked from SKILL.md
```

**SKILL.md frontmatter** (required):

```yaml
---
name: <kebab-case-name> # must match the folder name
description: <one-line summary> # what the skill does and when to use it
user-invocable: true # exposed as a /<name> slash command
---
```

**Rules of thumb:**

- Folder name = kebab-case, matches `name:` in frontmatter exactly.
- Keep `SKILL.md` scannable — pattern + quick reference. Push long explanations into `rules/`.
- Open `SKILL.md` with a clear **When to use** signal so Claude knows when to load it.

## License

[MIT](./LICENSE)
