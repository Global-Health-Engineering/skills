# Examples

Worked examples of commits produced by the `commit` skill, covering all three paths (Claude-assisted, human-only, mixed) and showing the `prompts/` archive file alongside the commit message where relevant.

The skill overrides the harness default `Co-Authored-By: Claude ...` trailer. Claude-touched commits get `Assisted-by: Claude <model-id>` as the last trailer; human-touched commits get a `Human-authored: true` trailer. A mixed commit carries both, so every commit is positively countable by one query or the other.

## 1. Simple Claude-assisted commit, no prompts archived

Commit message:

```
fix(parser): handle nested quotes in CSV cells

Previously, quoted strings containing commas inside other quoted
strings were split incorrectly. The parser now tracks nesting depth
with a small state machine.

Closes #142
Assisted-by: Claude claude-opus-4-7
```

No `prompts/` file written. The user declined the archive step (or did not engage). The `Assisted-by:` trailer still goes in: Claude wrote the fix, even though the prompts that produced it are not archived.

## 2. Claude-assisted commit with one prompt archived

Commit message:

```
feat(abstract): tighten introduction to 150 words

The conference word limit is 300; the intro was eating 220 of them.
Trimmed redundant framing and moved the contribution statement to
the end of paragraph one.

Prompts:
- 2026-06-02-001-tighten-abstract-intro
Assisted-by: Claude claude-opus-4-7
```

Companion archive file `prompts/2026-06-02-001-tighten-abstract-intro.md`:

```markdown
---
id: 2026-06-02-001-tighten-abstract-intro
timestamp: 2026-06-02T14:32:11+02:00
model: claude-opus-4-7
files_touched:
  - manuscript/abstract.qmd
---

Can you cut the abstract intro to 150 words? Keep the contribution sentence but lose the lit-review framing.
```

The commit message stays tidy; the archive file holds the full prompt for later citation in a Methods section. The archive file does not record a `commit_sha`; the `id` is unique and the commit is found via `git log --all --grep='2026-06-02-001-tighten-abstract-intro'`.

## 3. Claude-assisted commit with several prompts archived

Commit message:

```
refactor(skill): restructure prompt-attachment menu

Switched from free-text selection to numbered options with range
support. The earlier free-text version was ambiguous when prompts
contained commas.

Prompts:
- 2026-06-02-002-numbered-prompt-menu
- 2026-06-02-003-add-range-selection
- 2026-06-02-004-add-all-none-shortcuts
Assisted-by: Claude claude-opus-4-7
```

Three companion files under `prompts/`, one per prompt, in the order the prompts were given. The sequence is the design history.

## 4. Prompt with code, archived verbatim

User's prompt was:

> "I'm getting an error: `TypeError: cannot read property 'name' of undefined at line 42`. Here's the function:
> ```js
> function getUser(id) {
>   return users.find(u => u.id === id).name;
> }
> ```
> Can you fix it?"

Commit message:

```
fix(users): guard against missing user in getUser

`users.find()` returns undefined when no match exists. Accessing
`.name` on undefined crashed the request handler. Now returns null
and lets callers decide how to handle a missing user.

Prompts:
- 2026-06-02-005-fix-getuser-typeerror
Assisted-by: Claude claude-opus-4-7
```

Companion archive file `prompts/2026-06-02-005-fix-getuser-typeerror.md`:

```markdown
---
id: 2026-06-02-005-fix-getuser-typeerror
timestamp: 2026-06-02T15:07:43+02:00
model: claude-opus-4-7
files_touched:
  - src/users/getUser.js
---

I'm getting an error: `TypeError: cannot read property 'name' of undefined at line 42`. Here's the function:

```js
function getUser(id) {
  return users.find(u => u.id === id).name;
}
```

Can you fix it?
```

Code fences are preserved verbatim. The archive is lossless because a reader of the published paper may need to see exactly what was asked.

## 5. Long prompt, archived without truncation

Commit message:

```
docs(readme): rewrite quickstart for clarity

Restructured into three sections (install, first command, next steps)
and removed three outdated screenshots.

Prompts:
- 2026-06-02-006-rewrite-quickstart-section
Assisted-by: Claude claude-opus-4-7
```

The archive file contains the full prompt body (multiple paragraphs, code blocks, the works). No truncation. If the trailer line `- 2026-06-02-006-rewrite-quickstart-section` is the only thing in `git log --oneline`, that is by design: the commit log stays short, the archive carries the long text.

## 6. Human-only commit

Commit message:

```
docs(abstract): fix typo in conclusion sentence

s/effected/affected/

Human-authored: true
```

No `Assisted-by:`, no `Co-Authored-By:`, no `Prompts:`. No file written to `prompts/`. The human typed this fix in their editor; Claude was not involved. The `Human-authored: true` trailer is the positive signal.

`git log --grep='^Human-authored:'` lists every commit where the human had a hand (human-only + mixed). Relying on the absence of `Assisted-by:` instead would also catch unmarked legacy commits and plain harness commits, so the explicit trailer is what makes the human bucket countable.

## 6b. Mixed commit (human + Claude in one commit)

The user hand-edited `CHANGELOG.md` while Claude rewrote `src/parser.js` in the same session, and they chose to commit both together rather than split.

Commit message:

```
feat(parser): support nested quotes; note in changelog

Claude rewrote the quote-nesting state machine in the parser. The
changelog entry was written by hand.

Prompts:
- 2026-06-24-001-nested-quote-state-machine
Human-authored: true
Assisted-by: Claude claude-opus-4-8
```

Companion archive file `prompts/2026-06-24-001-nested-quote-state-machine.md` lists **only** the Claude-touched file under `files_touched:`:

```markdown
---
id: 2026-06-24-001-nested-quote-state-machine
timestamp: 2026-06-24T11:20:05+02:00
model: claude-opus-4-8
files_touched:
  - src/parser.js
---

Rewrite the CSV quote handling as a small state machine that tracks nesting depth.
```

`CHANGELOG.md` is absent from `files_touched:` because Claude did not write it; recording it there would make the audit trail claim authorship Claude does not hold. The commit matches both `^Assisted-by:` and `^Human-authored:`, which is exactly what identifies it as mixed. When a clean split is feasible, prefer two commits (one per path) over one mixed commit; reach for mixed when the changes are genuinely intertwined or the user declines the split.

## 7. Extracting prompts for a Methods section or supplementary material

The archive is designed to make the Methods section trivial to produce.

**List every Claude-assisted commit on the branch:**

```bash
git log --grep='^Assisted-by:' --pretty=format:'%h  %s'
```

**List every prompt referenced in the log, in commit order:**

```bash
git log --grep='^Prompts:' --pretty=format:'%H' \
  | while read sha; do
      git show --no-patch --format='%B' "$sha" \
        | awk '/^Prompts:/{f=1;next} /^[A-Z][a-zA-Z-]+:/{f=0} f && /^- /{print substr($0,3)}'
    done
```

**Produce a Markdown table for supplementary material** (commit, model, prompt). Looks up each prompt's commit by grepping the log for the prompt's `id`:

```bash
{
  echo "| Commit | Model | Prompt |"
  echo "|---|---|---|"
  for f in prompts/*.md; do
    id=$(awk '/^id:/{print $2; exit}' "$f")
    sha=$(git log --all --grep="$id" --pretty=format:'%h' | head -1)
    model=$(awk '/^model:/{print $2; exit}' "$f")
    prompt=$(awk '/^---$/{c++; next} c==2' "$f" | tr '\n' ' ' | sed 's/|/\\|/g')
    echo "| $sha | $model | $prompt |"
  done
} > methods/supplementary-prompts.md
```

**Count prompts by model** (useful when a paper spans more than one model version):

```bash
grep -h '^model:' prompts/*.md | sort | uniq -c
```

**Find the commit for a single archive file:**

```bash
id=$(awk '/^id:/{print $2; exit}' prompts/2026-06-02-001-tighten-abstract-intro.md)
git log --all --grep="$id" --pretty=format:'%H %s'
```

The archive files are RO-Crate-compatible if you later wrap the repo for archival. Each file has stable identifiers (`id`, `timestamp`) that a `ro-crate-metadata.json` can reference. The commit SHA is derived on demand from `git log --grep` rather than stored in the file, since the SHA cannot be known at archive-write time (it depends on the file's own contents).

## Anti-examples

Bad: subject too long.

```
feat(parser): add support for nested quotes in CSV cells with mixed delimiter types and unicode
```

Bad: past tense.

```
feat(parser): added nested quote handling
```

Bad: not enough context, no body.

```
fix: bug
```

Bad: `Prompts:` trailer with full text instead of IDs (this is the old format; the archive file should hold the full text).

```
Prompts:
- Can you fix the parser? It's choking on the export file and I need to ship this today.
```

Fix: archive the prompt to `prompts/YYYY-MM-DD-NNN-slug.md` and reference its ID.

```
Prompts:
- 2026-06-02-007-fix-parser-export-choke
```

Bad: `Co-Authored-By: Claude` in a commit made through this skill. The skill replaces that trailer with `Assisted-by:`.

```
Co-Authored-By: Claude Opus 4.7 <noreply@anthropic.com>
```

Fix:

```
Assisted-by: Claude claude-opus-4-7
```

Bad: `Assisted-by:` trailer on a human-only commit. Miscounts human work as Claude-touched.

```
docs(abstract): fix typo in conclusion sentence

Assisted-by: Claude claude-opus-4-7
```

Fix: use `Human-authored: true` on the human-only path; reserve `Assisted-by:` for commits Claude actually touched.

```
docs(abstract): fix typo in conclusion sentence

Human-authored: true
```

Bad: human-only commit with no authorship trailer at all (the pre-`Human-authored:` behaviour). The commit is then indistinguishable from an unmarked legacy or harness commit and is not countable as human work.

Fix: add `Human-authored: true`. Only genuinely Claude-touched commits omit it (they carry `Assisted-by:` instead).

Bad: `Assisted-by:` trailer placed before issue references (out of order).

```
Assisted-by: Claude claude-opus-4-7
Closes #142
```

Fix: `Assisted-by:` goes last. See `commit-conventions.md` for the ordering convention.

Bad: archive file carries a `commit_sha:` field. The schema does not include one. The SHA cannot be known at write time (it depends on the file's contents) and is derived on demand from `git log --grep='<id>'`.

Fix: remove the field. Use the `id` as the sole link between archive file and commit.

Bad: archive file `id` does not match the corresponding `Prompts:` trailer entry in the commit message. Breaks the round-trip lookup.

Fix: copy the `id` from the YAML front matter into the trailer verbatim, no rename, no slug edit.
