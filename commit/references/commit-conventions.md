# Commit conventions

Reference for the `commit` skill. Loaded when Claude needs deeper guidance on type/scope choice or message structure.

## Conventional Commits format

```
<type>(<scope>): <subject>
<BLANK LINE>
<body>
<BLANK LINE>
<trailers>
```

Only the first line (`<type>(<scope>): <subject>`) is required.

## Types

| Type       | When to use                                                                 |
|------------|------------------------------------------------------------------------------|
| `feat`     | A new user-facing feature.                                                   |
| `fix`      | A bug fix.                                                                   |
| `docs`     | Documentation only. README, comments, JSDoc, changelogs.                     |
| `style`    | Formatting, whitespace, semicolons. No code behavior change.                 |
| `refactor` | Code restructured without changing behavior. No new feature, no fix.         |
| `perf`     | A change that improves performance.                                          |
| `test`     | Adding or correcting tests. No production code change.                       |
| `chore`    | Maintenance: deps, build config, tooling. Nothing user-visible.              |
| `build`    | Changes to build system or external dependencies (npm, cargo, make, etc.).   |
| `ci`       | CI configuration (GitHub Actions, CircleCI, etc.).                           |
| `revert`   | Reverts a previous commit. Body should reference the reverted SHA.           |

If a commit could be two types, pick the one that best describes *the primary user-visible impact*. A refactor that incidentally fixes a bug is a `fix`. A new feature that required a refactor is a `feat`.

## Scope

Optional but recommended. The scope is the area of the codebase affected, typically a module, package, or component name.

Examples:

- `feat(auth): add OAuth provider for GitHub`
- `fix(parser): handle empty input`
- `docs(readme): clarify install steps`
- `refactor(api/users): extract validation into middleware`

Keep scope short (one or two words). Use the same scope name consistently across commits; check `git log --oneline | head -50` to see what's already in use.

If the change spans multiple scopes equally, omit scope rather than listing several.

## Subject line

- **≤ 50 characters.** Hard limit ~72.
- **Imperative mood.** "add", "fix", "remove", "update", not "added", "fixes", "adding".
  - Mnemonic: it should complete the sentence "If applied, this commit will ___".
- **Lowercase.** Except proper nouns and acronyms (`fix(API): ...`).
- **No trailing period.**
- **No issue numbers in the subject**; they go in the body or trailers.

Bad → Good:

- `Fixed the bug.` → `fix(parser): handle nested quotes`
- `WIP` → (squash before committing, or use `chore: wip` only on private branches)
- `Update stuff` → `refactor(api): rename UserService methods for clarity`

## Body

Optional. Use it when the subject can't convey enough.

- **Wrap at 72 characters.**
- **Explain *why*, not *what*.** The diff shows what. The commit message explains the reasoning, the alternatives considered, the constraints.
- **Separate from subject with a blank line.**

Good body content:

- Why this change was needed.
- What alternatives were considered and rejected.
- Side effects or follow-ups required.
- Performance numbers if relevant.

Skip the body if the subject line is self-explanatory.

## Trailers

Trailers are key-value pairs at the bottom of the message, separated from the body by a blank line. They're machine-parseable.

Common trailers:

- `Closes #123`: closes the referenced issue on merge.
- `Refs #123`: references but doesn't close.
- `Assisted-by: Claude <model-id>`: marks a Claude-assisted commit. This skill uses this trailer instead of the harness default `Co-Authored-By: Claude ...`. See the "Authorship attribution" section below.
- `Co-Authored-By: Name <email>`: adds a human collaborator (GitHub recognizes this). Use only for actual human co-authors. Do NOT use for Claude.
- `Signed-off-by: Name <email>`: DCO sign-off.
- `Reverts: <sha>`: for revert commits.
- `Prompts:`: custom trailer used by this skill to reference archived prompt IDs (see `examples.md`). Body of each ID points to a file under `prompts/` in the repo.

Each trailer on its own line. The block is separated from the body by a blank line.

Ordering convention used by this skill, top to bottom:

1. `Prompts:` (content-like, references in-repo archive files).
2. Issue references (`Closes`, `Refs`).
3. Identity trailers (`Co-Authored-By` for humans, `Signed-off-by`).
4. `Assisted-by:` (last, so it sits as the bottom-line attribution).

## Authorship attribution

This skill distinguishes two kinds of commit:

- **Claude-assisted.** Claude wrote or substantially edited the change. Commit gets an `Assisted-by: Claude <model-id>` trailer (e.g. `Assisted-by: Claude claude-opus-4-7`). The `model-id` should be the lowercased model name from the current environment, no friendly name.
- **Human-only.** The human authored the change without Claude's involvement (typo fix, hand revision in editor, manual refactor). Commit gets no `Assisted-by:` trailer, no `Co-Authored-By: Claude`, no `Prompts:` trailer.

**Absence is the signal.** Querying the log:

- `git log --grep='^Assisted-by:'` lists Claude-assisted commits.
- `git log --invert-grep --grep='^Assisted-by:'` lists human-only commits.

### Why `Assisted-by:` instead of `Co-Authored-By:`?

The harness's default `Co-Authored-By: Claude ... <noreply@anthropic.com>` has two problems for projects that need an honest authorship record:

1. It implies legal co-authorship (the GitHub UI treats `Co-Authored-By` as a contributor). Several journal publishers (COPE, Nature, Elsevier, JOSS) have flagged this as inappropriate for LLMs, since an LLM cannot accept responsibility for a contribution.
2. It fires on every commit Claude Code makes, so it cannot distinguish "Claude wrote this" from "Claude ran `git commit` on the human's behalf". As a marker of actual Claude authorship, it is noise.

`Assisted-by:` is an emerging community standard (used in guidelines from the Linux Kernel, Apache, LLVM, Fedora, OpenInfra, OpenTelemetry) for the case where an AI agent contributed to a change but is not a legal author. It is the right semantic fit and queryable in `git log`.

The harness still adds `Co-Authored-By: Claude` by default; this skill overrides that by emitting its own trailer block. If you see `Co-Authored-By: Claude` in a commit made through this skill, it is a bug.

## Splitting commits

If `git diff --staged` shows changes across multiple unrelated concerns, split them.

Heuristic: imagine reverting just one of the concerns. If the revert is straightforward, the commits should be separate. If unwinding one would require unwinding the other, they belong together.

To split:

1. `git reset HEAD` (or `git restore --staged .`) to unstage everything.
2. `git add -p` to stage hunks interactively, or `git add <specific files>`.
3. Commit. Repeat for the next concern.

## Repo conventions override these defaults

If `git log --oneline -20` shows the repo doesn't use Conventional Commits, **match the existing style instead**. A consistent log is more valuable than a "correct" one. Flag the style mismatch to the user only if they ask, or if the existing style is actively bad (e.g. all commits are "update" or "fix stuff").
