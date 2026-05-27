# Examples

Worked examples of commit messages produced by `commit-push-pr`, with and without the prompt trailer.

## 1. Simple commit, no prompts attached

```
fix(parser): handle nested quotes in CSV cells

Previously, quoted strings containing commas inside other quoted
strings were split incorrectly. The parser now tracks nesting depth
with a small state machine.

Closes #142
```

User said no to prompt attachment. Standard Conventional Commits format.

## 2. Commit with one prompt attached

```
feat(abstract): tighten introduction to 150 words

The conference word limit is 300; the intro was eating 220 of them.
Trimmed redundant framing and moved the contribution statement to
the end of paragraph one.

Prompts:
- Can you cut the abstract intro to 150 words? Keep the contribution sentence but lose the lit-review framing.
```

The prompt explains the editorial decision better than any body text could. Good case for attaching.

## 3. Commit with several prompts attached, showing chronology

```
refactor(skill): restructure prompt-attachment menu

Switched from free-text selection to numbered options with range
support. The earlier free-text version was ambiguous when prompts
contained commas.

Prompts:
- The prompt menu is confusing - users have to type long strings. Can we make it numbered?
- Also let them pick ranges like 1-3, not just comma-separated.
- One more thing: add an "all" and "none" shortcut.
```

The sequence of prompts is the design history. Useful when revisiting "why does this look this way?" months later.

## 4. Prompt with code, trimmed

User's prompt was:
> "I'm getting an error: `TypeError: cannot read property 'name' of undefined at line 42`. Here's the function:
> ```js
> function getUser(id) {
>   return users.find(u => u.id === id).name;
> }
> ```
> Can you fix it?"

Resulting commit:

```
fix(users): guard against missing user in getUser

`users.find()` returns undefined when no match exists. Accessing
`.name` on undefined crashed the request handler. Now returns null
and lets callers decide how to handle a missing user.

Prompts:
- I'm getting an error: TypeError: cannot read property 'name' of undefined at line 42. Here's the function: [code omitted] Can you fix it?
```

Code fence stripped to `[code omitted]` — the diff captures the code, the prompt captures the intent.

## 5. Long prompt, truncated

```
docs(readme): rewrite quickstart for clarity

Restructured into three sections (install, first command, next steps)
and removed three outdated screenshots.

Prompts:
- The README quickstart is too dense. New users have told me they get lost halfway through the install section because we mix system requirements, install commands, and post-install verification all in one block. Can you restructure it so...
```

Truncated at ~200 chars with `...`. Full prompt is in the conversation history if anyone needs it.

## 6. PR description (separate from commit message)

For the PR body — not the commit message — use a simple structure:

```markdown
## What
Tightens the conference abstract intro to fit the 150-word target.

## Why
Word limit is 300; full abstract was at 380. Intro had the most cuttable
material.

## How
- Removed lit-review framing in paragraph one
- Moved contribution sentence to end of paragraph one
- Reworded transitions in paragraphs two and three
```

Don't put the Prompts trailer in the PR body — it goes in the commit message only. PRs are for reviewers; commit messages are for archaeology.

## Anti-examples

❌ Subject too long:
```
feat(parser): add support for nested quotes in CSV cells with mixed delimiter types and unicode
```

❌ Past tense:
```
feat(parser): added nested quote handling
```

❌ Not enough context, no body:
```
fix: bug
```

❌ Prompt trailer with raw newlines (breaks `git log --oneline`):
```
Prompts:
- Can you fix the parser?
  It's choking on the export file
  and I need to ship this today.
```

Fix: collapse to one line per prompt:
```
Prompts:
- Can you fix the parser? It's choking on the export file and I need to ship this today.
```

❌ Auto-added Claude attribution (unwanted by default):
```
fix(parser): handle nested quotes

Co-Authored-By: Claude <noreply@anthropic.com>
🤖 Generated with Claude Code
```

The default is no AI attribution. Only add if the user explicitly asks.
