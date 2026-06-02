# skills

Personal collection of [Claude skills](https://code.claude.com/docs/en/skills).

Each subfolder is a self-contained skill: a `SKILL.md` (required) plus any reference docs, scripts, or assets it needs.

## Structure

```
skills/
├── README.md                    ← this file
├── commit-push-pr/              ← skill: git commit + push + PR workflow
│   ├── SKILL.md
│   └── references/
│       ├── commit-conventions.md
│       └── examples.md
└── <next-skill>/
    └── SKILL.md
```

## Conventions

- One folder per skill, named in `kebab-case`.
- Every skill has a `SKILL.md` at its root with YAML frontmatter (`name`, `description`).
- Larger context goes in `references/` and is loaded only when needed.
- Executable helpers go in `scripts/`.
- Static templates, fonts, images go in `assets/`.
- Keep `SKILL.md` under ~500 lines; push detail into `references/` if it grows.

## Installing a skill

Copy the skill folder into your Claude Code skills directory:

```bash
cp -r commit-push-pr ~/.claude/skills/
```

Or, on Claude.ai, package the folder into a `.skill` file and upload it.

## Skills in this repo

| Skill | What it does |
|-------|--------------|
| [`commit-push-pr`](./commit-push-pr/) | Commit staged changes with Conventional Commits format. Detects Claude-assisted vs human-only commits, marks them with `Assisted-by:` (or nothing), archives selected prompts to `prompts/` at repo root, pushes, and opens a PR. |
