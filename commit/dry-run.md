# Dry-run plan: verifying the `commit` skill

A short, runnable script to verify each branch of the skill before pointing it at a real paper repo. Estimated time: about 8 minutes.

Run in any throwaway repo. None of these steps touch your actual paper.

## Setup (run once)

```bash
mkdir -p /tmp/commit-dryrun && cd /tmp/commit-dryrun
git init -q
git config user.email "you@example.com"
git config user.name "Your Name"
echo "# Scratch paper repo" > README.md
git add README.md
git commit -q -m "chore: initial commit"
```

## Check 1: Claude-assisted path, one prompt archived

Goal: verify that a `prompts/YYYY-MM-DD-001-*.md` file is written with proper YAML front matter, the commit message has `Prompts:` (ID-only) plus `Assisted-by:` trailers, no `Co-Authored-By:` appears, and the id round-trips between the archive file and the commit.

In a fresh Claude Code session inside `/tmp/commit-dryrun`:

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

Goal: verify the skill detects "no Claude tool calls" and writes a commit with `Human-authored: true` and no `Assisted-by:`, no `Prompts:`, no `prompts/` file.

```bash
# Edit by hand (no Claude tool call):
printf "\nA second paragraph added by hand.\n" >> manuscript/abstract.qmd
```

In the same session, tell Claude: *"Commit this."*

Verify:

```bash
git log -1 --format='%B'
git log -1 --format='%B' | grep '^Human-authored: true'              && echo GOOD || echo BAD
git log -1 --format='%B' | grep -E '^(Assisted-by|Prompts|Co-Authored-By):' && echo BAD || echo GOOD
ls prompts/ | wc -l                  # expect: still 1 (no new file)
```

## Check 2b: Mixed path

Goal: verify a diff that mixes Claude-touched and human-touched files, committed together (split declined), gets **both** authorship trailers and a `prompts/` archive scoped to the Claude-touched file only.

```bash
# Claude edits one file (via a tool call), human edits another by hand:
printf "\nHand-written changelog note.\n" >> CHANGELOG.md
```

Ask Claude to edit `manuscript/abstract.qmd` in the same session, then say: *"Commit both together as mixed, don't split."* Verify:

```bash
git log -1 --format='%B' | grep '^Human-authored: true'        && echo GOOD || echo BAD
git log -1 --format='%B' | grep '^Assisted-by:'                && echo GOOD || echo BAD
# The archive for this commit must NOT list CHANGELOG.md under files_touched:
grep -L 'CHANGELOG.md' prompts/*.md >/dev/null && echo "check files_touched scope by hand"
```

## Check 3: Override

Goal: confirm the user can force the human-only path even when Claude touched files this session.

In the same session, ask Claude to edit `manuscript/abstract.qmd` again, then say: *"Commit this as human-only."* Verify the resulting commit has `Human-authored: true` and no `Assisted-by:` trailer.

## Check 4: Claude-assisted commit without archive

Goal: confirm the "Claude did this but no archive" sub-case works. The commit gets `Assisted-by:` but no `Prompts:` trailer and no new file under `prompts/`.

In the same session, ask Claude to edit `manuscript/abstract.qmd` once more, then say: *"Commit this. No prompts archive."* Verify:

```bash
git log -1 --format='%B' | grep '^Assisted-by:' && echo GOOD || echo BAD
git log -1 --format='%B' | grep '^Prompts:'    && echo BAD  || echo GOOD
ls prompts/ | wc -l                  # expect: unchanged from Check 1
```

## Check 5: Extraction queries

Goal: confirm the Methods-section recipes in `references/examples.md` actually work against your log.

```bash
git log --grep='^Assisted-by:' --pretty=format:'%h  %s'      # Claude-touched (assisted + mixed)
git log --grep='^Human-authored:' --pretty=format:'%h  %s'   # human-touched (human-only + mixed)
ls prompts/ && cat prompts/*.md
```

A commit matching both greps is mixed; matching only one is that pure category; matching neither is an unmarked legacy/harness commit.

Also try the Markdown-table builder from example 7 in `references/examples.md`.

## Check 6: Skill stays in its lane

Goal: confirm the skill does not push and does not open PRs on its own.

Ask Claude: *"Now push and open a PR."* The skill should refuse to push or open a PR, and should tell you to run `git push` yourself and invoke the `open-pr` skill.

```bash
git log @{u}.. 2>/dev/null && echo "remote tracked" || echo "no remote, no push: GOOD"
```

## Clean up

```bash
rm -rf /tmp/commit-dryrun
```

## What "pass" looks like

All six checks meet the expected output, and `grep -i 'co-authored-by'` over the full log returns zero matches.
