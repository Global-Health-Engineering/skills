---
id: 2026-06-02-001-improve-skill-for-scientific-publication
timestamp: 2026-06-02T12:30:00+02:00
model: claude-opus-4-7
commit_sha: pending
files_touched:
  - commit-push-pr/README.md
  - commit-push-pr/SKILL.md
  - commit-push-pr/references/commit-conventions.md
  - commit-push-pr/references/examples.md
---

I am enjoying this skill but want to improve it further. Here is my use case that I need this for:

I am writing a scientific publication for a journal. The paper will be fully written under version control with regular commits. I need the skill to be able to cover when work was done by Claude vs. when I personally made changes and edits without Claude.

Also review if the current way of storing prompts in commits is the most efficient. I want to be able to extract the prompts at a later stage to be able to highlight how I instructed Claude. Is a prompts directory at root more useful to keep that audit trail? Does it make sense to combine the prompts into a CSV?

Think hard.
