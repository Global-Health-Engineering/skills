# Dry-run plan: verifying the `open-pr` skill

A short, runnable script to verify each branch of the skill before using it on a real paper repo. Estimated time: about 10 minutes. Requires a throwaway GitHub repo you can push to and run `gh pr create` against.

## Setup (run once)

```bash
# Pick an empty throwaway repo on GitHub, e.g. <you>/scratch-open-pr.
# Replace REMOTE with the SSH or HTTPS URL.
REMOTE="git@github.com:<you>/scratch-open-pr.git"

mkdir -p /tmp/openpr-dryrun && cd /tmp/openpr-dryrun
git init -q
git config user.email "you@example.com"
git config user.name "Your Name"
echo "# Scratch repo" > README.md
git add README.md
git commit -q -m "chore: initial commit"
git branch -M main
git remote add origin "$REMOTE"
git push -u origin main
git checkout -b dev
echo "A first line." > note.md
git add note.md
git commit -q -m "docs: add a scratch note"
git push -u origin dev
```

## Check 1: Refuse to run from a non-`dev` branch

Goal: confirm the skill stops when the current branch is anything other than `dev`.

```bash
git checkout -b feature/not-dev
echo "off-branch work" > off.md
git add off.md
git commit -q -m "docs: off-branch work"
git push -u origin feature/not-dev
```

In a Claude Code session inside the repo, say: *"Open a PR for this branch."* The skill should refuse: it only opens PRs from `dev` into `main`. No PR should be created.

```bash
gh pr list --state open --head feature/not-dev
# expect: no rows
```

Switch back when done:

```bash
git checkout dev
```

## Check 1b: Refuse to run without a pushed branch

Goal: confirm the skill stops when `dev` has no upstream. Reset the upstream temporarily:

```bash
git branch --unset-upstream dev
```

Ask Claude: *"Open a PR."* The skill should detect the missing upstream and tell you to run `git push -u origin dev` yourself. No PR should be created. Restore the upstream:

```bash
git push -u origin dev
```

## Check 2: Open a PR with a clean body

Goal: confirm the PR body has no `Prompts:`, no `Assisted-by:`, no "Generated with Claude Code" emoji line.

Ask Claude: *"Open a PR."* Verify after (the skill should open with `--base main --head dev` automatically):

```bash
PR=$(gh pr list --state open --head dev --json number --jq '.[0].number')
gh pr view "$PR" --json body --jq '.body' | grep -E '^(Assisted-by|Prompts):' && echo BAD || echo GOOD
gh pr view "$PR" --json body --jq '.body' | grep -F 'Generated with' && echo BAD || echo GOOD
gh pr view "$PR" --json body --jq '.body' | grep -E '🤖|✨|🎉' && echo BAD || echo GOOD
```

## Check 3: Test plan runner ticks passing boxes

Goal: confirm the runner walks `- [ ]` boxes, runs the cheapest verification, ticks on success, and updates the PR body via `gh pr edit`.

Edit the PR body to include a trivial Test plan:

```bash
gh pr edit "$PR" --body "$(cat <<'EOF'
## Summary

- Scratch note for dry-run.

## Test plan

- [ ] `note.md` exists at the repo root.
- [ ] `note.md` contains the word "first".
- [ ] (Visual) The note reads clearly.
EOF
)"
```

In Claude, say: *"Run the Test plan for this PR."* Verify after:

```bash
gh pr view "$PR" --json body --jq '.body'
# expect: first two boxes ticked [x], third remains [ ]
```

## Check 4: Test plan runner stops on failure

Goal: confirm the runner halts on the first failure and does not tick the failed box.

Edit the PR body again:

```bash
gh pr edit "$PR" --body "$(cat <<'EOF'
## Summary

- Scratch note for dry-run.

## Test plan

- [ ] `note.md` contains the word "never-appears-in-the-file".
- [ ] `note.md` exists at the repo root.
EOF
)"
```

In Claude, say: *"Run the Test plan again."* Verify after: neither box should be ticked, and Claude should report which one failed.

## Check 5: Artifact cleanup

Goal: confirm the runner deletes render artifacts it produced but leaves tracked files alone.

```bash
# Pre-populate a tracked file that looks like an artifact:
echo "<html>tracked</html>" > tracked-artifact.html
git add tracked-artifact.html
git commit -q -m "docs: add tracked-looking artifact"
git push
```

Update the PR body to include a render-style check (you can use a noop renderer, e.g. `touch tmp-output.html`):

```bash
gh pr edit "$PR" --body "$(cat <<'EOF'
## Summary

- Scratch note for dry-run.

## Test plan

- [ ] `touch tmp-output.html` succeeds.
EOF
)"
```

In Claude, say: *"Run the Test plan."* Verify after:

```bash
ls tmp-output.html 2>/dev/null   && echo BAD  || echo GOOD   # expect: deleted
ls tracked-artifact.html         && echo GOOD || echo BAD    # expect: present (tracked, must not delete)
```

## Clean up

```bash
gh pr close "$PR" --delete-branch
cd / && rm -rf /tmp/openpr-dryrun
# Optionally delete the throwaway GitHub repo: gh repo delete <you>/scratch-open-pr --yes
```

## What "pass" looks like

All six checks (1, 1b, 2, 3, 4, 5) meet the expected output, and the throwaway repo's PR list shows no leftover open PRs after cleanup.
