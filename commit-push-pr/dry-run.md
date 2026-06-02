# Dry-run plan: verifying `commit-push-pr`

A short, runnable script to verify each branch of the skill before pointing it at a real paper repo. Estimated time: about 10 minutes.

Run in any throwaway repo. None of these steps touch your actual paper.

## Setup (run once)

```bash
mkdir -p /tmp/cppr-dryrun && cd /tmp/cppr-dryrun
git init -q
git config user.email "you@example.com"
git config user.name "Your Name"
echo "# Scratch paper repo" > README.md
git add README.md
git commit -q -m "chore: initial commit"
```

## Check 1: Claude-assisted path, one prompt archived

Goal: verify that a `prompts/YYYY-MM-DD-001-*.md` file is written with proper YAML front matter, the commit message has `Prompts:` (ID-only) plus `Assisted-by:` trailers, no `Co-Authored-By:` appears, and the `commit_sha` fixup runs.

In a fresh Claude Code session inside `/tmp/cppr-dryrun`:

1. Tell Claude: *"Create a file `manuscript/abstract.qmd` with three sentences about climate adaptation."*
2. Then: *"Now commit this. Archive that prompt."*

Verify after:

```bash
ls prompts/                          # expect: YYYY-MM-DD-001-<slug>.md
git log -1 --format='%B'             # expect: Prompts: ID, then Assisted-by: Claude <model-id>
git log -1 --format='%B' | grep -i 'co-authored-by' && echo BAD || echo GOOD
grep '^commit_sha:' prompts/*.md     # expect: real SHA OR "pending" (see "Known limitation" below)
grep '^files_touched:' -A2 prompts/*.md
```

## Check 2: Human-only path

Goal: verify the skill detects "no Claude tool calls" and writes a clean commit with no `Assisted-by:`, no `Prompts:`, no `prompts/` file.

```bash
# Edit by hand (no Claude tool call):
printf "\nA second paragraph added by hand.\n" >> manuscript/abstract.qmd
```

In the same session, tell Claude: *"Commit this."*

Verify:

```bash
git log -1 --format='%B'
git log -1 --format='%B' | grep -E '^(Assisted-by|Prompts|Co-Authored-By):' && echo BAD || echo GOOD
ls prompts/ | wc -l                  # expect: still 1 (no new file)
```

## Check 3: Override

Goal: confirm the user can force the human-only path even when Claude touched files this session.

In the same session, ask Claude to edit `manuscript/abstract.qmd` again, then say: *"Commit this as human-only."* Verify the resulting commit has no `Assisted-by:` trailer.

## Check 4: Extraction queries

Goal: confirm the Methods-section recipes in `references/examples.md` actually work against your log.

```bash
git log --grep='^Assisted-by:' --pretty=format:'%h  %s'
git log --invert-grep --grep='^Assisted-by:' --pretty=format:'%h  %s'
ls prompts/ && cat prompts/*.md
```

Also try the Markdown-table builder from example 8 in `references/examples.md`.

## Check 5: PR body (optional)

Push the branch to a throwaway GitHub repo and ask Claude to open a PR. Verify the PR body contains no `Prompts:`, no `Assisted-by:`, and no "Generated with Claude Code" emoji line.

## Clean up

```bash
rm -rf /tmp/cppr-dryrun
```

## What "pass" looks like

All five checks meet the expected output, and `grep -i 'co-authored-by'` over the full log returns zero matches.

## Known limitation: the `commit_sha` fixup

The skill's Step 5 says to amend the commit to fill in the real SHA. There is a chicken-and-egg problem: amending changes the SHA, so the SHA you wrote into the file is the pre-amend SHA, which no longer exists in history.

Three honest options, in order of recommendation:

1. **Leave `commit_sha: pending` in the committed file.** The archive file's `id` (e.g. `2026-06-02-001-...`) is unique, and `git log --all --grep='2026-06-02-001-'` finds the commit. The SHA inside the file is then redundant.
2. **Skip the SHA fixup entirely** and let `pending` stand. Same outcome as option 1.
3. **Record the parent commit's SHA** as a stable anchor (it does not change when you amend the child). Not implemented yet; would need a SKILL.md change.

This will be tightened in a follow-up commit to the skill itself. For now, option 1 is what the skill produces in practice if you accept the existing amend step but reset to `pending` after observing the SHA drift. Or do nothing and accept `pending`.
