# TODO: open items for `commit-push-pr`

Open follow-ups discovered while self-applying the skill.

## 1. `tea` is installed but not logged in

`tea logins list` returned an empty table. If you want CLI-driven PR creation on Codeberg in future sessions, run `tea login add` once. The web UI works fine in the meantime.

## Resolved

- ~~`commit_sha` fixup is unworkable.~~ Fixed: dropped the field from the schema entirely. The archive file's `id` is the unique link between file and commit; `git log --all --grep='<id>'` retrieves the commit on demand. See `SKILL.md` Step 4 notes, `references/examples.md` example 8 ("Find the commit for a single archive file"), and `dry-run.md` Check 1 round-trip step.
- ~~Open PR for `bd86c0f docs(commit-push-pr): add dry-run plan`.~~ Merged as PR #3.
