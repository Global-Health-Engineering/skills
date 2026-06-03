---
id: 2026-06-03-002-found-another-gap-prior-session-commit
timestamp: 2026-06-03T09:56:34+02:00
model: claude-opus-4-7
files_touched:
  - commit/SKILL.md
  - commit/references/prior-session-detection.md
---

I have found another gap: exiting the session and then using commit slash command:

On your gap point

  You're right, and this matters enough to bake into the skill. The current commit skill's Step 0 says to "scan the current session
  for Claude tool calls," which silently misses everything from a prior, exited session in the same project directory, even though the
   JSONL transcripts are sitting at ~/.claude/projects/<sanitized-cwd>/*.jsonl (plus ~/.claude/projects/<sanitized-cwd>/memory/).
  Untracked Claude-written files in the working tree are the obvious symptom: they had to come from some session, and on a fresh repo
  (no commits yet) it's almost always a prior one.

  What the teaching material (and ideally the skill itself) should add:

  1. Path detection should look beyond the current session when staged/untracked files exist that the current session did not touch.
  Recipe: list ~/.claude/projects/<sanitized-cwd>/*.jsonl by mtime, grep each for Write/Edit/MultiEdit/NotebookEdit tool_use blocks,
  and check whether any file_path matches a file in the diff. If yes, that session is a candidate source.
  2. Prompt recovery from prior sessions. Same JSONL files contain role: user turns with content as a string (real user prompts)
  versus list (tool results). Filter to strings, drop slash-command and local-command wrappers. Each surviving turn is an archive
  candidate.
  3. Model attribution. Pull from the prior session's first assistant turn (message.model), not from the current session's
  environment, otherwise the Assisted-by: line lies.
  4. Honesty about scope. If the diff also contains files Claude did not touch (here, skill-tests.Rproj from RStudio), either exclude
  them from files_touched in the archive or call it out. The skill should be explicit about which to do.
  5. Edge case worth naming. A fresh repo with no commits but untracked Claude-written files is a strong signal of a prior session;
  treat as Claude-assisted by default and confirm.

  Want me to draft an addition to the skill's references/ covering this, or open an issue against the skill source?
