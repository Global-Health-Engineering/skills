---
name: open-pr
description: Open a GitHub pull request for the current branch and, on request, run through the PR's Test plan checklist. Assumes the branch is already committed and pushed; this skill never pushes on the user's behalf. Triggers on explicit verbs: "open a PR", "make a pull request", "pr", "pull request". Strips Claude attribution from the PR body (no Prompts: trailer, no Assisted-by: trailer, no Generated-with emoji line). Use this skill after the `commit` skill or after a manual commit; for committing itself, use `commit`.
---

# open-pr

A skill for taking an already-pushed branch and opening a clean GitHub pull request, then optionally walking the PR's Test plan checklist to verify the change end-to-end.

This skill picks up where the `commit` skill leaves off. It assumes:

1. The branch has at least one commit.
2. The branch is pushed to a remote with an upstream set.
3. The user has explicitly asked to open a PR.

If any of those are not true, stop and tell the user what to do first. This skill never pushes on the user's behalf.

What this skill adds on top of the harness's default `gh pr create` behavior:

- **A pre-flight check** that the branch is pushed and ahead of its base.
- **Two body rules**: no commit-level attribution trailers (`Prompts:`, `Assisted-by:`) in the PR body; no "Generated with Claude Code" emoji line.
- **A Test-plan runner** that walks `- [ ]` checkboxes in the PR body, runs the cheapest verification per item, ticks the boxes via `gh pr edit`, and cleans render artifacts.

---

## When to use this skill

Trigger only on explicit PR verbs from the user: "open a PR", "make a pull request", "pr", "pull request", "submit this for review".

Do NOT auto-trigger on:

- "ship it", "wrap this up", "I'm done", "let's submit this" (these often mean "summarize" or "move on", not "open a PR").
- A commit just landing (the `commit` skill explicitly hands off; the user must still say "open a PR").

If the user says "commit and open a PR", let the `commit` skill handle the commit first, then this skill handles the PR after they have pushed.

---

## Workflow

Follow these steps in order. Skip a step only if the user has already done it or explicitly opted out.

### 1. Verify branch state

Run these in one batch:

```bash
git status
git branch --show-current
git log @{u}.. --oneline 2>/dev/null || echo "NO_UPSTREAM"
git log @{u}..HEAD --oneline 2>/dev/null
```

From the output, confirm:

- **An upstream exists.** If `git log @{u}..` fails with "no upstream" or you see `NO_UPSTREAM`, stop. Tell the user the branch is not pushed and they need to run `git push -u origin <branch>` themselves. This skill does not push.
- **The branch is ahead of its upstream.** If `git log @{u}..HEAD` is empty, there is nothing to PR. Stop and ask whether the wrong branch is checked out.
- **The branch is not the default branch.** PRs from `main`/`master`/`trunk` into themselves are nonsense; stop and ask the user to switch to a feature branch.
- **Working tree is clean (or at least: the user is OK with uncommitted work being excluded).** If `git status` shows uncommitted or unpushed changes that look intentional, flag them: "You have uncommitted changes in X, Y. They will not be in the PR. Continue?"

### 2. Survey the commits on the branch

Detect the base branch (usually `main`; check `gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'` if unsure). Then:

```bash
git log <base>..HEAD --pretty=format:'%h %s'
git diff <base>...HEAD --stat
```

Use the commit list and the diff stat to understand what the PR will contain. A one-commit PR may not need a detailed body; a multi-commit refactor probably does.

### 3. Draft the PR title and body

**Title.** Under 70 characters. Prefer the first commit's subject if the branch has one commit. For multi-commit branches, pick a title that summarizes the arc, not any single commit.

**Body.** Follow the harness's default PR template:

```markdown
## Summary

<1-3 bullets describing the change and why>

## Test plan

- [ ] <verification item>
- [ ] <verification item>
```

**Two body rules specific to this skill:**

- Do **NOT** include `Prompts:` or `Assisted-by:` trailers in the PR body. They belong in commit messages only. The PR body is for reviewers; commit messages are for archaeology.
- Do **NOT** include the "Generated with Claude Code" emoji line (`🤖 Generated with [Claude Code](...)`) that the harness's default template suggests. The user's global guidance forbids emojis in any output.

Show the drafted title and body to the user before opening the PR.

For body and Test-plan examples, see `references/examples.md`.

### 4. Open the PR

Use `gh pr create` per the harness rules. Pass the body via heredoc to preserve formatting:

```bash
gh pr create --title "the pr title" --body "$(cat <<'EOF'
## Summary

- ...

## Test plan

- [ ] ...
EOF
)"
```

Return the PR URL to the user. Capture the PR number for the next step:

```bash
PR=$(gh pr view --json number --jq '.number')
```

### 5. Offer to run the Test plan

Ask the user simply: "Should I run through the Test plan now?"

If **no**: stop. The PR is done.

If **yes**:

1. **Work through each checkbox in order.** For each `- [ ]` item in the body, pick the cheapest verification that actually proves the claim:
   - **Rendering a Quarto / R Markdown / Jupyter document**: invoke the local renderer (`quarto render <file>`, `Rscript -e 'rmarkdown::render(...)'`, etc.). On macOS where `quarto` isn't on PATH, try `/Applications/RStudio.app/Contents/Resources/app/quarto/bin/quarto` or `/Applications/Positron.app/Contents/Resources/app/quarto/bin/quarto` before giving up.
   - **Content claims** ("X appears in the output", "no Y in the directory"): use `grep` / `Read` against the rendered file or source.
   - **Build/test claims**: run the relevant `npm test`, `pytest`, `devtools::test()`, etc.
   - **Human-eyes claims** (visual inspection, "looks reasonable", items prefixed with `(Visual)`): say so, leave the box unchecked, and report which items need the user's eyes.

2. **If any check fails:** stop. Report the failure, do not tick the box, do not delete anything yet. Ask how to proceed (fix and re-run, or accept and move on).

3. **If all checks pass and the run produced render artifacts** (`.html`, `.pdf`, `.docx`, `.knit.md`, Quarto `_files/` directories, `dist/`, `build/`):
   - List the artifacts.
   - Delete them. Default to deleting without re-asking: the user has already opted into the Test-plan runner. Skip artifacts that were tracked in git before this run (check `git status` / `git ls-files` before deleting). Only clean up what the test run itself produced.

4. **Tick off the boxes.** Use `gh pr edit "$PR" --body-file -` (or `gh api`) to rewrite the PR body with `- [x]` for items that passed. Preserve everything else in the body verbatim. If some items required human verification, leave those as `- [ ]` and note which ones in your reply to the user.

5. **Report.** A one-line summary: "All N items passed, artifacts deleted, PR body updated." Or, if mixed: "M of N passed, K need your eyes: [list]."

---

## What this skill does NOT do

- **Push.** This skill never runs `git push`. If the branch is not pushed, stop and tell the user.
- **Commit.** See the `commit` skill for crafting commits with Conventional Commits format and prompt archiving.
- **Force-push.** Never `git push --force` or `--force-with-lease`. Out of scope.
- **Echo commit-level attribution into the PR body.** `Prompts:` and `Assisted-by:` trailers stay in commit messages.
- **Add emoji to the PR body.** No "Generated with Claude Code" line, no decorative emoji anywhere. The user's global guidance forbids emojis in any output.
- **Delete tracked files.** The Test-plan runner only deletes artifacts the run itself produced; files tracked in git (`git ls-files`) are never touched.

---

## Edge cases

- **No GitHub remote (e.g. Codeberg or a private Gitea):** `gh` will not work. If `tea` is installed and logged in, fall back to `tea pulls create`. If neither is available, surface the situation to the user and stop.
- **PR template in `.github/pull_request_template.md`:** read it, prefer its sections over this skill's default scaffold, but still enforce the two body rules (no commit-level trailers, no emoji).
- **Test plan has no `- [ ]` items:** nothing to run. Tell the user the body has no checklist and stop.
- **Test plan item references a file that does not exist:** report the missing file, leave the box unchecked, continue with the next item.
- **`gh pr edit` rejects the body** (markdown size limit, unicode quirk): show the error, leave the original body in place, ask the user how to proceed.
- **PR opened earlier in the session, body already partially ticked:** read the current body before walking checkboxes; never overwrite `[x]` back to `[ ]`.

---

## Reference files

- `references/examples.md`: worked examples of PR bodies, Test-plan checklists, and runner output for passing, failing, and mixed runs.
