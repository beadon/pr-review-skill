# pr-review-skill

A Claude Code `/pr-review` command that performs comprehensive GitHub PR reviews using the `gh` CLI. No Python dependencies — pure shell and `gh`.

## What it does

1. Fetches PR metadata, diff, inline comments, review history, and head SHA via `gh` CLI
2. Analyzes the diff against a standard review checklist (functionality, correctness, security, testing, style, PR quality)
3. Writes a detailed internal `review.md` and a clean `human.md` draft ready to post
4. Creates `/send` and `/send-decline` commands — nothing is posted to GitHub until you run one of them

## Installation

Copy `SKILL.md` to your global Claude Code commands directory:

```bash
curl -fsSL https://raw.githubusercontent.com/beadon/pr-review-skill/main/SKILL.md \
  -o ~/.claude/commands/pr-review.md
```

Or clone and symlink:

```bash
git clone git@github.com:beadon/pr-review-skill.git ~/skills/pr-review-skill
ln -s ~/skills/pr-review-skill/SKILL.md ~/.claude/commands/pr-review.md
```

## Usage

Inside Claude Code, pass a PR URL or number:

```
/pr-review https://github.com/owner/repo/pull/123
/pr-review owner/repo 123
```

After the review is prepared, run:

```
/send          # approve and post
/send-decline  # request changes and post
```

## Requirements

- [`gh` CLI](https://cli.github.com/) authenticated with access to the target repo
- Claude Code

## Two-stage workflow

All analysis is local. Nothing is posted to GitHub until you explicitly run `/send` or `/send-decline`. Edit `human.md` in the IDE before sending if needed.

Review files are written to `/tmp/pr-review/<owner>/<repo>/<pr-number>/pr/`.
