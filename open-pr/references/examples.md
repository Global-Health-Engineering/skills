# Examples

Worked examples of PR bodies and Test-plan runs produced by the `open-pr` skill.

The skill never echoes commit-level attribution (`Prompts:`, `Assisted-by:`) into the PR body; those trailers belong in commit messages only. PRs are for reviewers; commit messages are for archaeology.

## 1. PR body for a small content change

For a single-commit PR tightening an abstract:

```markdown
## Summary

- Tightens the conference abstract intro to fit the 150-word target.
- Word limit is 300; full abstract was at 380. Intro had the most cuttable
  material.

## Test plan

- [ ] `quarto render manuscript/abstract.qmd` succeeds.
- [ ] Rendered abstract is at or under 150 words in paragraph one.
- [ ] Contribution sentence appears at the end of paragraph one.
```

No `Prompts:` trailer. No `Assisted-by:` trailer. No "Generated with Claude Code" emoji line. The body is for reviewers.

## 2. PR body for a multi-commit refactor

For a branch with several related commits, summarize the arc of the work, not the per-commit diffs.

```markdown
## Summary

- Restructures the prompt-attachment menu from free-text input to numbered
  options with range support.
- Earlier free-text input was ambiguous when prompts contained commas.
- Three commits on this branch; see `git log` for the breakdown.

## Test plan

- [ ] Numbered menu renders without errors on a session with 5+ prompts.
- [ ] Range selection (`1-3`) selects three contiguous items.
- [ ] `all` and `none` shortcuts behave as documented in SKILL.md Step 4.
```

## 3. Test plan with a mix of automated and human-eyes checks

When some boxes require visual inspection, mark them clearly so the runner does not falsely tick them.

```markdown
## Test plan

- [ ] `pytest tests/parser/` passes.
- [ ] `grep -r 'TypeError' src/` returns no hits.
- [ ] (Visual) Rendered abstract reads naturally; no awkward transitions.
- [ ] (Visual) Figure captions match in-text references.
```

The Test-plan runner tickets the first two on success and leaves the `(Visual)` items unchecked, reporting them back to the user in the wrap-up.

## 4. Test plan runner: a passing run

After opening the PR, the runner walks each `- [ ]` box, picks the cheapest verification, and ticks on success. Sample wrap-up message:

> "All 3 items passed, artifacts deleted, PR body updated."

Behind the scenes:

```bash
# Render check
quarto render manuscript/abstract.qmd
# Content check
grep -c 'climate adaptation' manuscript/abstract.html
# Word-count check
wc -w manuscript/abstract.qmd

# Tick the boxes
gh pr edit 42 --body-file - <<'EOF'
## Summary
...
## Test plan
- [x] `quarto render manuscript/abstract.qmd` succeeds.
- [x] Rendered abstract is at or under 150 words in paragraph one.
- [x] Contribution sentence appears at the end of paragraph one.
EOF

# Clean render artifacts produced by this run
rm -rf manuscript/abstract.html manuscript/abstract_files/
```

The runner only deletes artifacts the test run itself produced. Files that were tracked in git before the run (`git ls-files`) are never deleted.

## 5. Test plan runner: a failing run

Stop on the first failure. Do not tick the box. Report the failure verbatim, ask how to proceed.

> "Box 2 failed: word count is 162, target was 150. Stopping. Fix the abstract and re-run, or accept and move on?"

No `gh pr edit` until the user decides. No artifacts deleted yet.

## 6. Test plan runner: mixed results

When some boxes pass and others need human eyes, tick the passing ones and report the rest.

> "2 of 4 items passed, 2 need your eyes: 'Rendered abstract reads naturally', 'Figure captions match in-text references'."

The PR body now has `- [x]` for the two automated checks and `- [ ]` for the two visual ones. Body otherwise verbatim.

## Anti-examples

Bad: PR body carrying commit-level attribution.

```markdown
## Summary

Tightens abstract.

Assisted-by: Claude claude-opus-4-7
Prompts:
- 2026-06-02-001-tighten-abstract-intro
```

Fix: leave the trailers in the commit message. The PR body explains the change to a reviewer; the commit message keeps the audit trail.

Bad: PR body with a "Generated with Claude Code" emoji line.

```markdown
## Summary

Tightens abstract.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

Fix: remove it. The skill keeps PR bodies free of emoji and of tool advertising.

Bad: Test-plan runner ticking boxes that were never verified.

Fix: only `- [x]` boxes that the runner positively confirmed. Leave human-only and skipped items as `- [ ]` and report them in the wrap-up.

Bad: Test-plan runner deleting tracked files because they happened to be rendered output.

```bash
rm manuscript/abstract.html   # but this file is tracked in git!
```

Fix: check `git ls-files` before deleting. Only clean up artifacts the run itself produced.
