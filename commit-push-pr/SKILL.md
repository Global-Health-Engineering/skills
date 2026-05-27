---
name: commit-push-pr
description: Commit staged or unstaged changes with a well-crafted Conventional Commits message, optionally attach selected user prompts from the current session as a commit trailer, then push and open a pull request. Use this skill whenever the user asks to commit, push, open a PR, or "save changes" — and also when they say things like "ship this", "wrap this up", "submit this", or otherwise signal they're done with a unit of work, even if they don't explicitly say "commit". Triggers on phrases including "commit", "push", "pr", "pull request", "ship it", "save changes", "log this work", "make a commit with the prompts".
---

# commit-push-pr

A skill for turning a working-directory change into a clean commit (and optionally a pushed branch + pull request), with an option to record the prompts that produced the change in the commit message itself.

The goal is a tidy `git log` that future-you (or a collaborator) can read in one pass — and, when wanted, a breadcrumb trail back to the prompts that produced the change.

---

## When to use this skill

Trigger on any of:
- Explicit asks: "commit this", "push this", "open a PR", "make a pull request", "save my changes".
- Implicit asks: "ship it", "wrap this up", "I'm done", "let's submit this".
- The user has been editing files and signals they want to move on.

If the user only says "commit", default to commit-only (no push, no PR) unless they previously expressed a workflow preference. Ask before pushing if the branch is new, or if pushing would open a PR.

---

## Workflow

Follow these steps in order. Skip a step only if the user has already done it manually or explicitly opted out.

### 1. Survey the state

Run these in one batch to understand what you're working with:

```bash
git status
git diff --staged
git diff
git branch --show-current
git log --oneline -5
```

From the output, determine:
- **What changed** — files, scope, whether it's one logical change or several.
- **Whether anything is staged** — if nothing is staged but there are unstaged changes, ask whether to stage all or selectively.
- **The current branch** — if it's `main`, `master`, `trunk`, or the repo's default, warn the user and offer to create a feature branch before committing.
- **Recent commit style** — match the tone/format of recent commits in this repo. If they use Conventional Commits, follow that. If they're more freeform, don't impose ceremony.

### 2. Decide: one commit or several?

If the diff spans multiple unrelated concerns (e.g. a bug fix *and* a new feature *and* a docs update), recommend splitting. Ask the user:

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
- Body wraps at 72 characters. Explains *why*, not *what* — the diff already shows what.
- One logical change per commit.
- Reference issues/PRs in the body or as trailers (`Closes #123`, `Refs #456`).

Show the drafted message to the user before committing. Don't just commit silently.

For deeper guidance and worked examples, see `references/commit-conventions.md`.

### 4. Offer the prompt-attachment step

**This is the distinctive part of this skill.** After drafting the message and before committing, ask:

> "Include prompts from this session in the commit message? (y/N)"

If **no** (or the user doesn't respond about it): proceed to commit without trailer.

If **yes**:

1. **Collect candidate prompts.** Look back through the current session at the user's messages (not your own replies, not tool calls). A "prompt" is one user turn. Skip:
   - One-word acknowledgments ("yes", "ok", "thanks", "go ahead")
   - Pure clarifications about the skill itself ("what does step 3 mean")
   - Anything before the most recent commit in this repo (it's already captured)

2. **Present a numbered menu.** For each candidate prompt, show:
   ```
   [N] <first ~80 chars of prompt, with newlines collapsed>...
   ```
   Number them in chronological order (oldest first). Include a final option:
   ```
   [a] all
   [n] none
   ```

3. **Ask for selection.** Accept:
   - Comma-separated numbers: `1,3,5`
   - Ranges: `1-3`
   - `all` or `a`
   - `none` or `n`

4. **Format the trailer.** Append to the commit message body, separated by a blank line:

   ```
   Prompts:
   - <prompt 1, full text if short, else truncated to ~200 chars with "...">
   - <prompt 3, ...>
   ```

   For each prompt:
   - Strip surrounding whitespace.
   - Collapse internal newlines to single spaces (keeps `git log --oneline` clean).
   - Strip code fences and their contents — replace with `[code omitted]` if the prompt was primarily code.
   - Truncate to ~200 chars per prompt; append `...` if truncated.
   - If the total trailer would exceed ~1000 chars, ask the user whether to summarize or only keep the most pivotal prompts.

For trailer format examples, see `references/examples.md`.

### 5. Commit

Use a heredoc for multi-line messages so quoting doesn't bite:

```bash
git commit -m "$(cat <<'EOF'
feat(parser): handle nested quotes in CSV cells

Previously, quoted strings containing commas inside other quoted
strings would be split incorrectly. Now uses a state machine that
tracks nesting depth.

Prompts:
- Can you fix the CSV parser? It's choking on the customer-export file.
- The bug is that nested quotes get split as if they were field separators.

Closes #142
EOF
)"
```

After committing, run `git log -1 --stat` and show the user the result.

### 6. Push (only if asked or implied)

If the user said "push" or "PR" or "ship", continue. Otherwise stop here and confirm the commit is done.

Before pushing:
- If the current branch has no upstream, push with `git push -u origin <branch>`.
- If on `main`/`master` and a remote default-branch protection might exist, double-check with the user before pushing.

### 7. Open the PR (only if asked or implied)

Use `gh pr create` if the GitHub CLI is available:

```bash
gh pr create --title "<same as commit subject, or summary if multiple commits>" \
             --body "$(cat <<'EOF'
## What
<one-line summary>

## Why
<context, motivation, link to issue>

## How
<approach, notable decisions>
EOF
)"
```

PR description guidelines:
- **Active voice, present tense.** "Adds X" not "Added X".
- **Optimize for reviewer time.** Orient them in 30 seconds: what changed, why, where to start reading.
- **Don't attribute to Claude or AI** unless the user explicitly asks. No `Co-Authored-By: Claude`, no "Generated with Claude Code" footer. The user's commits, the user's PR.
- **Preserve every link** if updating an existing PR description.

If `gh` is not installed, output the `git push` command and the PR URL skeleton the user can open manually.

---

## What this skill does NOT do

- **Force-push.** Never `git push --force` or `--force-with-lease` without explicit user approval per push.
- **Rewrite shared history.** No `rebase -i` or `commit --amend` on commits that have been pushed, unless the user is explicit.
- **Commit sensitive files.** Scan the diff for things that look like secrets (API keys, `.env` files, private keys, tokens). If anything matches, stop and ask before committing.
- **Auto-add Claude attribution.** Default is no `Co-Authored-By: Claude` trailer. Users who want it can ask.

---

## Edge cases

- **Pre-commit hooks fail:** show the hook output, ask the user how to proceed. Don't silently `--no-verify`.
- **Merge conflicts on push:** stop and surface the conflict; don't auto-resolve.
- **Detached HEAD:** stop and explain; offer to create a branch from the current commit.
- **Empty diff after staging:** confirm there's something to commit. `git commit --allow-empty` only on explicit request.
- **Very long prompts:** in the prompt-attachment menu, if a prompt is so long that even its preview is unwieldy, mark it `[N] [long prompt: ~X words]` so the user can still pick it without seeing the full text inline.

---

## Reference files

- `references/commit-conventions.md` — full Conventional Commits cheatsheet with type definitions and scope guidance.
- `references/examples.md` — worked examples of commit messages with and without prompt trailers.
