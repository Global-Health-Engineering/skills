# commit skill: human-only path is blocked by the auto-mode classifier

## Summary

On the human-only path, the skill instructs the agent to run
`git commit` with a `Human-authored: true` trailer. When the agent
does this, the Claude Code auto-mode classifier denies the commit as
"Content Integrity / Impersonation", on the grounds that an
agent-produced commit is falsely attributing edits as human-authored.

The result: a genuinely human-only commit cannot be committed by the
agent at all through this skill. The very case the trailer exists to
mark is the one case the skill cannot complete.

## How to reproduce

1. In a session, ask the agent to edit a file (any Claude-assisted work).
2. Then hand-edit a *different* change yourself, leaving it unstaged.
3. Ask the agent to `commit`.
4. The skill's Step 0 detects the human-only path (the diff contains
   only your hand edits) and drafts a message with `Human-authored: true`.
5. The agent runs `git commit`. The classifier denies it:

   > Permission for this action was denied by the Claude Code auto mode
   > classifier. Reason: The commit message trailer "Human-authored:
   > true" falsely attributes agent-produced edits as human-authored,
   > matching Content Integrity / Impersonation.

## Root cause

The block is not wrong in general. It is correct that an *agent*
asserting "a human, not an agent, wrote this" is unverifiable from a
diff and is the exact shape of an attribution wash. The two trailers
are asymmetric:

- `Assisted-by: Claude` is an agent admitting involvement. Safe for the
  agent to apply.
- `Human-authored: true` is an agent asserting *non*-involvement. Only
  the human can credibly make that claim. When the agent applies it, the
  classifier cannot distinguish a genuine hand-edit from laundering
  agent output as human work, so it blocks the category.

The skill's bug is that it tells the *agent* to apply a trailer that is
only trustworthy when applied by the *human*. Provenance the agent can
see in-session ("this diff arrived as an unstaged change I did not
produce") is not provenance the classifier can see; it only sees an
agent making a self-clearing claim.

## Affected skill text

`SKILL.md`:

- Line 17 (feature list): `Human-authored: true` for human-touched work.
- Line 49, line 70 (path table): human-only path -> `Human-authored: true`.
- Lines 209-215 (Step 5 trailer rules): human-only and mixed apply
  `Human-authored: true`.
- Lines 220-223 (counting queries): rely on the trailer being present.

All of these assume the agent can emit `Human-authored: true`. It
cannot, at least not on the human-only path where the agent is the one
running the commit.

## Proposed fix

The fix should keep the trailer's meaning (human-only commits stay
countable) while moving the *act of asserting it* to the human, who is
the only party the claim is trustworthy from.

Options, roughly in order of preference:

1. **Hand human-only commits to the user to run.** On the human-only
   path, the skill drafts the full message (including
   `Human-authored: true`) but does not run `git commit` itself.
   Instead it prints the exact command for the user to run via the
   session's `! <command>` prefix, so the human is literally the actor
   applying their own attribution. This is the cleanest fix: the trailer
   stays, and it is asserted by the party who can vouch for it. Update
   Step 5 and the "What this skill does NOT do" section to state that
   the skill never runs human-only commits on the user's behalf.

2. **Drop the trailer on the agent-run path; document the query change.**
   If the agent does run the human-only commit, omit
   `Human-authored: true` and accept that human-only commits are then
   only identifiable as "no `Assisted-by:` and no `Prompts:`". Update the
   counting section (lines 220-223) accordingly. Weaker, because it
   collides with legacy/harness commits, which is exactly the ambiguity
   the trailer was added to remove.

3. **Both:** default to option 1 (hand to the user); fall back to
   option 2 only if the user explicitly says "you commit it" and accepts
   the missing trailer.

The mixed path needs the same treatment: a mixed commit also carries
`Human-authored: true`, so an agent-run mixed commit will hit the same
block. Whatever fix is chosen for human-only must cover mixed too.

## Notes

- The `Assisted-by:` (Claude-assisted) path is unaffected; the agent can
  and should continue to run those commits itself.
- This is not a request to weaken the classifier. The classifier is
  doing the right thing. The skill is asking the agent to do something
  the agent should not do, and the skill is what should change.

## Environment

- Skill: `commit` (`<path to your clone>/commit`)
- Model: claude-opus-4-8[1m]
- Observed: agent-run human-only commit denied by auto-mode classifier.
