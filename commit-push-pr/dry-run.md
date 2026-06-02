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

Goal: verify that a `prompts/YYYY-MM-DD-001-*.md` file is written with proper YAML front matter, the commit message has `Prompts:` (ID-only) plus `Assisted-by:` trailers, no `Co-Authored-By:` appears, and the id round-trips between the archive file and the commit.

In a fresh Claude Code session inside `/tmp/cppr-dryrun`:

1. Tell Claude: *"Create a file `manuscript/abstract.qmd` with three sentences about climate adaptation."*
2. Then: *"Now commit this. Archive that prompt."*

Verify after:

```bash
ls prompts/                                       # expect: YYYY-MM-DD-001-<slug>.md
git log -1 --format='%B'                          # expect: Prompts: ID, then Assisted-by: Claude <model-id>
git log -1 --format='%B' | grep -i 'co-authored-by' && echo BAD || echo GOOD
grep '^commit_sha:' prompts/*.md && echo BAD || echo GOOD   # expect: no commit_sha field at all
grep '^files_touched:' -A2 prompts/*.md

# Round-trip: id in file should appear verbatim in the commit's Prompts: trailer
id=$(awk '/^id:/{print $2; exit}' prompts/*.md | head -1)
git log --all --grep="$id" --pretty=format:'%h %s'   # expect: one hit
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
