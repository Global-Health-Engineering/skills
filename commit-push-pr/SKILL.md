---
name: commit-push-pr
description: Commit staged or unstaged changes with a well-crafted Conventional Commits message, optionally attach selected user prompts from the current session as a commit trailer, then push and open a pull request. Use this skill when the user asks to commit, push, or open a PR. Triggers on explicit git verbs: "commit", "push", "pr", "pull request", "open a pr", "make a commit with the prompts". Do NOT auto-trigger on softer phrases like "ship it" or "wrap this up" unless the user has also named a git action.
---

# commit-push-pr

A skill for turning a working-directory change into a clean commit (and optionally a pushed branch + pull request), with an option to record the prompts that produced the change in the commit message itself.

The goal is a tidy `git log` that future-you (or a collaborator) can read in one pass, and, when wanted, a breadcrumb trail back to the prompts that produced the change.

This skill assumes the harness's built-in "Committing changes with git" and "Creating pull requests" instructions are in effect. It does not restate them. What it adds:

1. **Conventional Commits guidance** (type, scope, subject, body rules).
2. **A prompt-attachment trailer** that records the user prompts behind the change.
3. **Splitting heuristics** for diffs that span multiple concerns.

---

## When to use this skill

Trigger only on explicit git verbs from the user: "commit", "push", "open a PR", "make a pull request", "save my changes to git".

Do NOT auto-trigger on:

- "ship it", "wrap this up", "I'm done", "let's submit this" (these often mean "summarize" or "move on", not "commit").
- General editing or reviewing requests.

If the user only says "commit", default to commit-only (no push, no PR). Confirm before pushing if the branch has no upstream, or before opening a PR.

---

## Workflow

Follow these steps in order. Skip a step only if the user has already done it or explicitly opted out.

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

### 4. Offer the prompt-attachment step

**This is the distinctive part of this skill.** After drafting the message and before committing, ask whether to include prompts from this session in the commit message.

Prefer using `AskUserQuestion` for the selection step (structured choices). If `AskUserQuestion` isn't available, present a numbered list in plain text and accept a natural-language reply.

If **no** (or the user doesn't engage): proceed to commit without the trailer.

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

4. **Format the trailer.** Append to the commit message body, separated by a blank line:

   ```
   Prompts:
   - <prompt 1, full text if short, else truncated>
   - <prompt 3, ...>
   ```

   For each prompt:
   - Strip surrounding whitespace.
   - Collapse internal newlines to single spaces (keeps `git log --oneline` clean).
   - Strip code fences and their contents; replace with `[code omitted]` if the prompt was primarily code.
   - Truncate long prompts so the trailer stays readable; if the total trailer would dominate the commit message, ask the user whether to summarize or only keep the most pivotal prompts. (The conversation history is the source of truth; the trailer is a breadcrumb, not an archive.)

For trailer format examples, see `references/examples.md`.

### 5. Commit

The harness already covers commit mechanics (heredoc quoting, the `Co-Authored-By` trailer, never `--amend`, never `--no-verify`, never `git add -A`). Follow those rules; do not restate them here.

This skill adds two things on top:

- The `Prompts:` trailer (step 4) goes in the body, before any `Closes #...`, `Refs #...`, or `Co-Authored-By:` trailers.
- After committing, run `git log -1 --stat` and show the user the result.

### 6. Push (only if asked)

If the user said "push" or "PR", continue. Otherwise stop after the commit.

The harness's PR-creation rules apply. This skill adds nothing to the push step itself; just remember:

- New branch without upstream: `git push -u origin <branch>`.
- On a default branch: confirm with the user before pushing.
- On merge conflict during push: stop, surface the conflict, ask how to proceed. Do not run `git pull` automatically; it can create unwanted merge commits or trigger an unintended rebase.

### 7. Open the PR (only if asked)

Use `gh pr create` per the harness instructions.

This skill adds one rule:

- **Do not put the `Prompts:` trailer in the PR body.** It belongs in the commit message only. PRs are for reviewers; commit messages are for archaeology.

### 8. Offer to run the PR's Test plan

After opening the PR, ask the user whether to run the checks listed under `## Test plan` in the PR body. Phrase the question simply, e.g.: "Should I run through the Test plan now?"

If **no**: stop. The PR is done.

If **yes**:

1. **Work through each checkbox in order.** For each item, pick the cheapest verification that actually proves the claim:
   - Rendering a Quarto / R Markdown / Jupyter document: invoke the local renderer (`quarto render <file>`, `Rscript -e 'rmarkdown::render(...)'`, etc.). On macOS where `quarto` isn't on PATH, try `/Applications/RStudio.app/Contents/Resources/app/quarto/bin/quarto` or `/Applications/Positron.app/Contents/Resources/app/quarto/bin/quarto` before giving up.
   - Content claims ("X appears in the output", "no Y in the directory"): use `grep` / `Read` against the rendered file or source.
   - Build/test claims: run the relevant `npm test`, `pytest`, `devtools::test()`, etc.
   - If a claim can only be verified by a human (visual inspection, "looks reasonable"): say so, leave the box unchecked, and report which items need the user's eyes.

2. **If any check fails:** stop. Report the failure, do not tick the box, do not delete anything yet. Ask how to proceed (fix and re-run, or accept and move on).

3. **If all checks pass and the run produced render artifacts** (`.html`, `.pdf`, `.docx`, `.knit.md`, Quarto `_files/` directories, `dist/`, `build/`):
   - List the artifacts.
   - Delete them. Default to deleting without re-asking: the user has already opted into this skill's workflow and asked for tests. Skip artifacts that were tracked in git before this run (check `git status` / `git ls-files` before deleting). Only clean up what the test run itself produced.

4. **Tick off the boxes.** Use `gh pr edit <PR>  --body-file -` (or `gh api`) to rewrite the PR body with `- [x]` for items that passed. Preserve everything else in the body verbatim. If some items required human verification, leave those as `- [ ]` and note which ones in your reply to the user.

5. **Report.** A one-line summary: "All N items passed, artifacts deleted, PR body updated." Or, if mixed: "M of N passed, K need your eyes: [list]."

---

## What this skill does NOT do

- **Force-push.** Never `git push --force` or `--force-with-lease` without explicit user approval per push.
- **Rewrite shared history.** No `rebase -i` or `commit --amend` on commits that have been pushed, unless the user is explicit.
- **Commit sensitive files.** Scan the diff and `git status` for things that look like secrets (API keys, `.env`, private keys, tokens, credential files). Respect `.gitignore`: if an ignored-looking file appears as untracked-but-about-to-be-staged, flag it. Stop and ask before committing anything suspicious.

---

## Edge cases

- **Pre-commit hooks fail:** show the hook output, ask the user how to proceed. Don't silently `--no-verify`. Per harness rules, fix the underlying issue and create a new commit; do not `--amend`.
- **Detached HEAD:** stop and explain; offer to create a branch from the current commit.
- **Empty diff after staging:** confirm there's something to commit. `git commit --allow-empty` only on explicit request.
- **Compaction dropped earlier prompts:** for the prompt-attachment step, only offer prompts you can still see verbatim. Don't reconstruct from summaries.

---

## Reference files

- `references/commit-conventions.md`: full Conventional Commits cheatsheet with type definitions and scope guidance.
- `references/examples.md`: worked examples of commit messages with and without prompt trailers.
