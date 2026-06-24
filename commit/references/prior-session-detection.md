# Detecting Claude-assisted work from a prior session

The Step 0 path detection in `SKILL.md` defaults to scanning the **current** session's tool calls. That misses a real case: the user did Claude-assisted work in a previous session, exited, came back, and now invokes `/commit` in a fresh session that has no in-context history of those edits.

Symptom: `git status` shows staged or untracked files that the current session never touched. On a brand-new repo (no commits yet), the entire diff was almost certainly produced by a prior session.

This reference documents how to recover the missing context from the on-disk session transcripts under `~/.claude/projects/`. It supplements Step 0 (path detection), Step 4 (prompt archive), and the model-attribution line in Step 5.

## When to invoke this recipe

Run it when **both** of these hold:

1. The current session has no `Edit`, `Write`, `MultiEdit`, `NotebookEdit`, or working-tree-mutating `Bash` tool calls in its visible history.
2. `git diff --staged --name-only` and `git ls-files --others --exclude-standard` together return at least one file.

Also run it when the repo has zero commits and there are untracked files: a fresh repo with Claude-touched files is almost always a prior-session artifact.

Do **not** run it if the current session already explains the diff. The current session is authoritative when it has the relevant tool calls.

## Where the transcripts live

For a project at `/Users/alice/code/widgets`, sessions are at:

```
~/.claude/projects/-Users-alice-code-widgets/<session-uuid>.jsonl
```

The directory name is the absolute working directory with `/` replaced by `-` and a leading `-`. To compute it from the current `cwd`:

```bash
sanitized="$(pwd | sed 's|/|-|g')"
proj_dir="$HOME/.claude/projects/${sanitized}"
```

Each `.jsonl` file is one session. Lines are JSON objects. Records relevant here have shape:

```json
{"type": "user",      "message": {"role": "user",      "content": "<string OR list>"}, "cwd": "...", "timestamp": "..."}
{"type": "assistant", "message": {"role": "assistant", "content": [...], "model": "claude-opus-4-7"}, "timestamp": "..."}
```

Key field rules (verified against current session files):

- `message.content` is a **string** when the user typed a real prompt. It is a **list** when the entry is a tool result (`type: tool_result` inside the list).
- Assistant tool calls are list-form `content` with blocks of `type: tool_use` and a `name` field (`Write`, `Edit`, `MultiEdit`, `NotebookEdit`, `Bash`, etc.). The `input.file_path` field holds the absolute path the tool acted on.
- `message.model` is set on assistant turns and is the model ID that produced that turn (e.g. `claude-opus-4-7`).
- `cwd` is set on most records and is the absolute working directory at the time the entry was written.

## Candidate-session search

List session files for this project, newest first, and inspect each until you find one (or more) that touched files currently in the diff:

```bash
ls -t "$proj_dir"/*.jsonl 2>/dev/null
```

For each session file, scan its tool_use blocks. A session is a **candidate source** for the current diff if any of these is true:

- It has a `Write`, `Edit`, `MultiEdit`, or `NotebookEdit` tool call whose `input.file_path` matches a path in the staged or unstaged diff.
- It has a `Bash` tool call whose `input.command` plausibly mutated a file in the diff (`mv`, `rm`, `>`, `>>`, `sed -i`, format/codegen tools).

The most recent matching session is usually the one to use. If two sessions touched overlapping files, treat them as a single Claude-assisted history and merge their prompts in timestamp order during Step 4.

### Inspection one-liner

This Python snippet, run via `Bash`, summarizes one session's tool_use activity and reports which sessions touched any of a given set of paths. Pass the target paths as positional args.

```bash
python3 - "$@" << 'PY'
import json, os, sys, glob
targets = set(os.path.abspath(p) for p in sys.argv[1:])
proj = os.path.expanduser("~/.claude/projects/" + os.getcwd().replace("/", "-"))
sessions = sorted(glob.glob(os.path.join(proj, "*.jsonl")),
                  key=os.path.getmtime, reverse=True)
for s in sessions:
    hits, model, first_ts, last_ts = set(), None, None, None
    with open(s) as f:
        for line in f:
            try: o = json.loads(line)
            except: continue
            ts = o.get("timestamp")
            if ts:
                first_ts = first_ts or ts
                last_ts = ts
            msg = o.get("message")
            if not isinstance(msg, dict): continue
            if msg.get("role") == "assistant" and model is None:
                model = msg.get("model")
            c = msg.get("content")
            if isinstance(c, list):
                for b in c:
                    if not isinstance(b, dict) or b.get("type") != "tool_use":
                        continue
                    if b.get("name") in ("Write","Edit","MultiEdit","NotebookEdit"):
                        fp = (b.get("input") or {}).get("file_path")
                        if fp and (not targets or os.path.abspath(fp) in targets):
                            hits.add(fp)
    if hits:
        print(os.path.basename(s), "model=", model,
              "first=", first_ts, "last=", last_ts,
              "hits=", len(hits))
        for h in sorted(hits): print("  ", h)
PY
```

Invoke with the paths from `git diff --name-only HEAD` (or `git diff --staged --name-only` plus `git ls-files --others --exclude-standard`).

## Prompt recovery

Once a candidate session is identified, harvest its real user prompts. A prompt is a `user`-role message whose `content` is a **string** (not a list). Exclude:

- Wrappers from local commands: anything that opens with `<local-command-stdout>`, `<local-command-caveat>`, `<command-name>`, `<command-args>`, `<command-message>`, or `<bash-input>`.
- Slash-command invocations: leading `/` followed by a single word with no further substance ("/exit", "/clear", "/compact").
- System-injected reminders: blocks wrapped in `<system-reminder>` only.

Each surviving turn is a candidate for the Step 4 archive list, in `timestamp` order. Preserve text verbatim (do not trim, paraphrase, or escape).

Extraction snippet:

```bash
python3 - "$1" << 'PY'
import json, sys, re
path = sys.argv[1]
SKIP_PREFIXES = ("<local-command-", "<command-name>", "<command-args>",
                 "<command-message>", "<bash-input>", "<bash-stdout>",
                 "<bash-stderr>")
def is_real_prompt(s):
    s2 = s.lstrip()
    if not s2: return False
    if s2.startswith(SKIP_PREFIXES): return False
    if re.match(r"^/[A-Za-z][\w:-]*\s*$", s2): return False
    if s2.startswith("<system-reminder>") and s2.rstrip().endswith("</system-reminder>"):
        return False
    return True
with open(path) as f:
    for line in f:
        try: o = json.loads(line)
        except: continue
        msg = o.get("message")
        if not isinstance(msg, dict): continue
        if msg.get("role") != "user": continue
        c = msg.get("content")
        if not isinstance(c, str): continue
        if not is_real_prompt(c): continue
        ts = o.get("timestamp", "")
        print("=== " + ts + " ===")
        print(c)
        print()
PY
```

## Model attribution

The `Assisted-by:` trailer must name the model that **actually produced the work**, not the current session's model. Pull the model ID from the candidate session's first assistant turn:

```python
# inside the same scan that finds candidate sessions
if msg.get("role") == "assistant" and model is None:
    model = msg.get("model")  # e.g. "claude-opus-4-7"
```

If the candidate session is multi-model (rare but possible after `/fast` toggles or model switches), report the **first** model used. If the diff spans two candidate sessions with different models, list both in the trailer separated by a comma: `Assisted-by: Claude claude-opus-4-7, claude-sonnet-4-6`.

If `model` is missing or empty (truncated or corrupt session file), do not guess. Ask the user.

## Honesty about scope

A prior-session candidate may not account for every file in the diff. The user could have hand-edited files in their editor (e.g. an IDE workspace file, a `.Rproj`, an `.idea/` change, a typo fix in `README.md`) between the prior session and the current `/commit` invocation.

Reconcile before writing the archive:

1. Compute `claude_touched = {file_path from candidate session tool_use blocks}`.
2. Compute `diff_paths = git diff --name-only HEAD` (and for new repos, `git ls-files --others --exclude-standard`).
3. Pick one of these strategies and apply it consistently:
   - **Strategy A (preferred): split.** Commit Claude-touched files first (Claude-assisted path with trailers and `prompts/` archive), then a separate commit for the human-touched-only files (human-only path with `Human-authored: true`). Tell the user which two commits you intend before doing it.
   - **Strategy B: commit as mixed.** Commit everything together on the **mixed path**: both `Human-authored: true` and `Assisted-by:` trailers, and a `prompts/` archive whose `files_touched:` lists **only** the intersection of `claude_touched` and the staged set. The two trailers make the commit countable on both sides and identifiable as mixed.

Do not silently list human-only files under `files_touched:` in the archive YAML. That is the line that, if it lies, breaks the audit purpose of the archive.

## The fresh-repo case

A repo with zero commits and untracked Claude-touched files is the strongest signal of a prior session. Default to the Claude-assisted path and confirm in one sentence ("This looks like a prior-session repo, treating as Claude-assisted; say 'human-only' to override"). Do not block on the confirmation.

For a fresh repo, `git diff --name-only HEAD` does not work (no HEAD). Use:

```bash
git ls-files --others --exclude-standard   # untracked files
git diff --cached --name-only              # whatever is already staged, if anything
```

## What this recipe does NOT do

- It does not replace the in-session detection. If the current session has Claude tool calls that explain the diff, use those; the on-disk transcripts are a fallback.
- It does not promise a complete history. Sessions older than the user's transcript retention (or sessions started from a different `cwd` and later moved into this directory) are not reachable. State that explicitly if you cannot account for every file.
- It does not run for repos outside `~/.claude/projects/`. If the sanitized-cwd directory does not exist, there is no prior-session data to recover; fall back to asking the user once.
- It does not modify or delete session files. Read-only.
