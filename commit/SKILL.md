---
name: commit
description: Commit staged or unstaged changes with a well-crafted Conventional Commits message. Detects whether the work was Claude-assisted or human-only and marks the commit accordingly (Assisted-by trailer + prompts/ archive file, or nothing). Use this skill when the user asks to commit. Triggers on explicit git verbs: "commit", "save my changes to git", "make a commit with the prompts". Do NOT auto-trigger on softer phrases like "ship it" or "wrap this up" unless the user has also named a git action. Push and PR creation are out of scope; see the `open-pr` skill for opening a pull request.
---

# commit

A skill for turning a working-directory change into a clean commit, with built-in support for distinguishing Claude-assisted commits from human-only commits and for archiving the prompts that produced Claude-assisted changes.

The goal is a tidy `git log` that future-you (or a collaborator, or a journal reviewer) can read in one pass, and, when wanted, an auditable trail back to the prompts that produced each Claude-assisted change.

This skill assumes the harness's built-in "Committing changes with git" instructions are in effect. It does not restate them. What it adds:

1. **Two commit paths** (Claude-assisted vs human-only), detected automatically and confirmed before committing.
2. **Conventional Commits guidance** (type, scope, subject, body rules).
3. **A prompt archive** under `prompts/` at repo root, with a slim `Prompts:` trailer in the commit message that references archive IDs.
4. **An `Assisted-by:` trailer** for Claude-assisted commits, replacing the harness default `Co-Authored-By: Claude` trailer.
5. **Splitting heuristics** for diffs that span multiple concerns.

Push and pull-request creation are handled separately. Run `git push` yourself, then invoke the `open-pr` skill if you want a PR.

---

## When to use this skill

Trigger only on explicit git verbs from the user: "commit", "save my changes to git", "make a commit".

Do NOT auto-trigger on:

- "ship it", "wrap this up", "I'm done", "let's submit this" (these often mean "summarize" or "move on", not "commit").
- General editing or reviewing requests.

If the user says "commit and push" or "commit and open a PR", run this skill for the commit, then hand off: tell the user to run `git push` themselves, and invoke `open-pr` if they want a PR.

---

## Workflow

Follow these steps in order. Skip a step only if the user has already done it or explicitly opted out.

### 0. Detect the path (Claude-assisted vs human-only)

Before anything else, decide which path this commit follows. The two paths differ in three places (authorship trailer, prompt capture, examples) and are otherwise identical.

**Heuristic.** Scan the current session for Claude tool calls that wrote to the working tree: `Edit`, `Write`, `NotebookEdit`, `MultiEdit`, or `Bash` commands that modified tracked files (`mv`, `rm`, `>`, `>>`, `sed -i`, code generators, formatters invoked by you, etc.). If any are present and their effects are still in the staged or unstaged diff, default to the **Claude-assisted path**. Otherwise default to the **human-only path**.

Edge cases:

- If Claude edited files earlier but the user reverted those edits by hand and only their own changes remain in the diff, treat as **human-only**.
- If the diff contains a mix (Claude edited file A, the user separately edited file B without you), default to **Claude-assisted** and offer to split (see Step 2).
- If the session was loaded from compaction and you cannot see whether Claude edited anything, ask the user once: "Claude-assisted or human-only commit?"

**Confirm in one sentence, then continue.** Example: "Detected Claude-assisted path (you asked me to edit `abstract.qmd` earlier). Override with 'human-only' if that's wrong." Do not block on this confirmation; if the user does not push back, proceed with the detected path.

**Override.** If the user's request contains "human-only", "no Claude", or similar, force the human-only path even if Claude tool calls are visible. If it contains "Claude-assisted" or "with prompts", force the Claude-assisted path.

The path determines:

| Step | Claude-assisted | Human-only |
|---|---|---|
| Step 4 (prompt capture) | Run | Skip |
| Step 5 trailers | `Prompts:` + `Assisted-by:` | None |
| `prompts/` files written | Yes (one per selected prompt) | No |

### 1. Survey the state

Run these in one batch:

```bash
git status
git diff --stat
git branch --show-current
git log --oneline -5
```

If `git diff --stat` shows a small change, follow up with `git diff` and `git diff --staged`. For large changes, ask the user which paths to focus on instead of dumping the full diff.

From the output, determine:

- **What changed**: files, scope, whether it's one logical change or several.
- **Whether anything is staged**: if nothing is staged but there are unstaged changes, ask whether to stage all or selectively.
- **The current branch**: if it's `main`, `master`, `trunk`, or the repo's default, warn the user and offer to create a feature branch before committing.
- **Recent commit style**: match the tone/format of recent commits in this repo. If they use Conventional Commits, follow that. If they're freeform, don't impose ceremony.

### 2. Decide: one commit or several?

If the diff spans multiple unrelated concerns (e.g. a bug fix and a new feature and a docs update), recommend splitting. Ask the user something like:

> "These changes look like ~N separate concerns: [list]. Commit as one, or split into N commits?"

If splitting, walk through them one at a time. Use `git add -p` or path-specific `git add` to stage each commit's content.

### 3. Draft the commit message

Follow **Conventional Commits** format by default (override if the repo's history clearly uses something else):

```
<type>(<scope>): <subject>

<body, optional>

<trailers, optional>
```

**Types:** `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `build`, `ci`, `revert`.

**Rules for a good commit message:**

- Subject line ≤ 50 characters. Imperative mood ("add" not "added", "fix" not "fixes"). No trailing period.
- Leave a blank line between subject and body.
- Body wraps at 72 characters. Explains *why*, not *what*; the diff already shows what.
- One logical change per commit.
- Reference issues/PRs in the body or as trailers (`Closes #123`, `Refs #456`).

Show the drafted message to the user before committing. Don't just commit silently.

For deeper guidance and worked examples, see `references/commit-conventions.md`.

### 4. Archive prompts (Claude-assisted path only)

Skip this step on the human-only path.

**This is the distinctive part of this skill.** After drafting the message and before committing, ask whether to archive prompts from this session.

Prompts are archived in two places:

- **Full text** in `prompts/YYYY-MM-DD-NNN-slug.md` at repo root (one file per prompt). This is the archive a journal Methods section or supplementary material can cite.
- **IDs only** in a `Prompts:` trailer in the commit message. This is the breadcrumb that survives squash, rebase, and `git log` queries.

Prefer using `AskUserQuestion` for the selection step (structured choices). If `AskUserQuestion` isn't available, present a numbered list in plain text and accept a natural-language reply.

If **no** (or the user doesn't engage): proceed to commit without the `Prompts:` trailer and without writing any files to `prompts/`. The `Assisted-by:` trailer still goes in (this is still a Claude-assisted commit).

If **yes**:

1. **Collect candidate prompts.** Look back through the user's messages still visible in the current context window (not your own replies, not tool calls). A "prompt" is one user turn. Don't fabricate prompts you can't see verbatim; if compaction has dropped earlier turns, tell the user and offer only what's available. Skip:
   - One-word acknowledgments ("yes", "ok", "thanks", "go ahead").
   - Pure clarifications about the skill itself ("what does step 3 mean").
   - Anything before the most recent commit in this repo (it's already captured).

2. **Present numbered candidates.** For each, show:

   ```
   [N] <first ~80 chars of prompt, with newlines collapsed>...
   ```

   Number chronologically (oldest first). Include `[a] all` and `[n] none`.

   If a prompt's preview would itself be unwieldy, mark it as `[N] [long prompt: ~X words]`.

3. **Ask for selection.** Accept comma-separated numbers (`1,3,5`), ranges (`1-3`), `all`/`a`, or `none`/`n`. In `AskUserQuestion`, model this as multi-select.

4. **Write one archive file per selected prompt** to `prompts/YYYY-MM-DD-NNN-slug.md`. Create `prompts/` if it does not exist. Naming rules:

   - `YYYY-MM-DD` is today's date in the local timezone.
   - `NNN` is a zero-padded sequence number per day. Pick by scanning `prompts/` for existing files dated today and taking the next free integer, starting at `001`.
   - `slug` is a short kebab-case slug derived from the first 6-8 meaningful words of the prompt, lowercased, with punctuation stripped. Keep it under ~50 chars.

   File contents:

   ```markdown
   ---
   id: 2026-06-02-001-tighten-abstract-intro
   timestamp: 2026-06-02T14:32:11+02:00
   model: claude-opus-4-7
   files_touched:
     - manuscript/abstract.qmd
   ---

   <full prompt text, verbatim, no truncation, code fences preserved>
   ```

   Notes:
   - `model` is the model ID (lowercased, no version label suffixes), pulled from the environment block (`claude-opus-4-7`, `claude-sonnet-4-6`, etc.).
   - `files_touched` is the output of `git diff --staged --name-only` at archive time, one entry per line.
   - The prompt body is the full user turn, unedited. Do not strip code fences. Do not collapse newlines. This is the archive copy; readability of `git log` is handled by the trailer.
   - No `commit_sha:` field. The `id` is unique and the `Prompts:` trailer in the matching commit pairs them. To find the commit for a given id, run `git log --all --grep='<id>'`. Recording a SHA in the file would require a fixed-point amend (writing the SHA you are about to compute), which is not possible.

5. **Format the slim trailer.** Append to the commit message body, separated by a blank line:

   ```
   Prompts:
   - 2026-06-02-001-tighten-abstract-intro
   - 2026-06-02-002-add-citation-for-cohort
   ```

   IDs only, one per line, in the order the prompts were given. Chronological, oldest first.

6. **Stage the new `prompts/*.md` files** alongside the rest of the diff (`git add prompts/2026-06-02-*.md`). They are part of the same commit.

For file and trailer format examples, see `references/examples.md`.

### 5. Commit

The harness already covers commit mechanics (heredoc quoting, never `--amend`, never `--no-verify`, never `git add -A`). Follow those rules; do not restate them here.

This skill overrides one harness default: do **not** append `Co-Authored-By: Claude ... <noreply@anthropic.com>`. Instead, use the trailers below. The harness only appends `Co-Authored-By` if the skill does not provide its own identity trailer, so by emitting `Assisted-by:` (or nothing) the skill takes ownership of authorship attribution.

**Trailer rules by path:**

| Path | Trailers (in this order) |
|---|---|
| Claude-assisted | `Prompts: <IDs>` (if step 4 ran), then `Closes #...` / `Refs #...`, then `Assisted-by: Claude <model-id>` |
| Human-only | `Closes #...` / `Refs #...` only. No `Assisted-by:`, no `Co-Authored-By:`, no `Prompts:`. |

The model ID is the lowercased model name from your environment (e.g. `claude-opus-4-7`, `claude-sonnet-4-6`). Format the trailer as `Assisted-by: Claude claude-opus-4-7`.

**After committing:**

Run `git log -1 --stat` and show the user the result.

No post-commit fixup is needed. The `Prompts:` trailer in the commit message references the archive file by its `id`, and the archive file references the commit back through that same `id`. To go from an archive file to its commit, run `git log --all --grep='<id>'`. To go from a commit to its archive files, read the `Prompts:` trailer.

If the user asked to "commit and push" or "commit and open a PR", stop after the commit. Tell them the commit is done and that push is theirs to run; if they want a PR, point them at the `open-pr` skill.

---

## What this skill does NOT do

- **Push.** This skill never runs `git push`. Run it yourself after the commit.
- **Open pull requests.** See the `open-pr` skill.
- **Rewrite shared history.** No `rebase -i`, no `commit --amend`. The harness's "create a new commit, never `--amend`" rule applies without exception.
- **Use `Co-Authored-By: Claude`.** This skill replaces the harness default with `Assisted-by: Claude <model-id>` on Claude-assisted commits and omits it entirely on human-only commits. Do not let the harness re-introduce `Co-Authored-By`.
- **Publish `prompts/` to external services.** The directory stays in the repo. The skill does not upload it anywhere, does not post it as a gist.
- **Truncate or paraphrase prompts in the archive.** Files in `prompts/` are verbatim copies of the user turn. The archive must be lossless so the Methods section can quote it.
- **Commit sensitive files.** Scan the diff and `git status` for things that look like secrets (API keys, `.env`, private keys, tokens, credential files). Respect `.gitignore`: if an ignored-looking file appears as untracked-but-about-to-be-staged, flag it. Stop and ask before committing anything suspicious. This includes prompts: if a prompt to be archived under `prompts/` contains what looks like a secret, flag it and ask before writing the file.

---

## Edge cases

- **Pre-commit hooks fail:** show the hook output, ask the user how to proceed. Don't silently `--no-verify`. Per harness rules, fix the underlying issue and create a new commit; do not `--amend`.
- **Detached HEAD:** stop and explain; offer to create a branch from the current commit.
- **Empty diff after staging:** confirm there's something to commit. `git commit --allow-empty` only on explicit request.
- **Compaction dropped earlier prompts:** for the prompt-archive step, only offer prompts you can still see verbatim. Don't reconstruct from summaries.
- **Compaction dropped the session's tool-call history:** path detection (Step 0) cannot rely on it. Ask the user once whether this is a Claude-assisted or human-only commit and continue.
- **`prompts/` is gitignored or excluded by a global gitignore:** the archive files will not be staged. Flag this to the user before committing and ask whether to force-add (`git add -f prompts/...`) or skip the archive for this commit.
- **`prompts/YYYY-MM-DD-NNN-slug.md` collision:** another process or session created the same NNN. Pick the next free integer; do not overwrite.
- **Looking up the commit for an archive file later:** run `git log --all --grep='<id>' --pretty=format:'%H %s'`. The `id` from the file's YAML front matter is unique and appears verbatim in the commit's `Prompts:` trailer.

---

## Reference files

- `references/commit-conventions.md`: full Conventional Commits cheatsheet with type definitions, scope guidance, and the authorship-attribution convention (`Assisted-by:` vs human-only).
- `references/examples.md`: worked examples of commit messages and prompt archive files for both paths, plus extraction recipes for a Methods section.
