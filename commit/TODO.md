# TODO: open items for the `commit` skill

Open follow-ups discovered while self-applying the skill.

(None currently open.)

## Resolved

- ~~`commit_sha` fixup is unworkable.~~ Fixed: dropped the field from the schema entirely. The archive file's `id` is the unique link between file and commit; `git log --all --grep='<id>'` retrieves the commit on demand. See `SKILL.md` Step 4 notes, `references/examples.md` example 7 ("Find the commit for a single archive file"), and `dry-run.md` Check 1 round-trip step.
- ~~Open PR for `bd86c0f docs(commit-push-pr): add dry-run plan`.~~ Merged as PR #3.
- ~~Skill bundled commit + push + PR.~~ Split: commit logic lives here; PR opening and Test-plan running moved to the `open-pr` skill. Push is left to the user. See repo root `README.md` skills table. (The `tea` login item moved to `open-pr/TODO.md`.)
