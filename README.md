# skills

<!-- badges: start -->
[![Lifecycle: experimental](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://lifecycle.r-lib.org/articles/stages.html#experimental)
<!-- badges: end -->

Claude Code skills from [Global Health Engineering](https://ghe.ethz.ch) at ETH Zurich.

Experimental. The skills are being trialled in the [agentsforsci-ghe](https://github.com/agentsforsci-ghe) workshop, and anything here can change without notice. Feedback is welcome as an [issue](https://github.com/Global-Health-Engineering/skills/issues).

## Skills

| Skill | What it does |
|-------|--------------|
| [`commit`](commit/) | Turns your working changes into a commit with a Conventional Commits message. Marks each commit as Claude-assisted, human-only, or mixed with trailers you can count in `git log`, and can archive the prompts behind a Claude-assisted commit under `prompts/`. Does not push. |
| [`open-pr`](open-pr/) | Opens a GitHub pull request from `dev` into `main` with a clean body (no attribution trailers, no emoji), and can run through the pull request's test plan checklist. Does not push. |

Each skill's `SKILL.md` is its full documentation. The `references/` folder holds worked examples, and `dry-run.md` is a script for trying the skill in a throwaway repository.

## Install

You need [Claude Code](https://code.claude.com/docs/en/overview).

Option 1, inside Claude Code. The skills are then called `/ghe-skills:commit` and `/ghe-skills:open-pr`, and Claude Code keeps them up to date.

```
/plugin marketplace add Global-Health-Engineering/skills
/plugin install ghe-skills@ghe-skills
```

Option 2, in a terminal. The skills are then called `/commit` and `/open-pr`, and you update them with `git pull`.

```bash
git clone https://github.com/Global-Health-Engineering/skills.git
cd skills
ln -s "$PWD/commit" ~/.claude/skills/commit
ln -s "$PWD/open-pr" ~/.claude/skills/open-pr
```

Copy the two folders instead of linking them if you do not want to keep the clone. To install for one project only, use that project's `.claude/skills/` folder instead of `~/.claude/skills/`. Start a new Claude Code session afterwards.

## What the skills assume

- `open-pr` uses the GitHub CLI (`gh`) and only opens pull requests from a branch named `dev` into `main`.
- `commit` replaces the default `Co-Authored-By: Claude` trailer with `Assisted-by: Claude <model-id>`. Commits that carry `Human-authored: true` are drafted by the skill and run by you, because only you can vouch for that claim.
- A change that you instructed and Claude executed counts as Claude-assisted. Whether that is the right classification is an open question.

## Where the skills came from

Both skills were developed in a personal skill collection between May and July 2026 and moved here with their git history. The `prompts/` folder holds the prompts behind the Claude-assisted commits in that history, written by the `commit` skill itself.

## License

MIT, see [LICENSE](LICENSE).
