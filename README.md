# pr-review-skill

A Claude Code `/pr-review` command that performs comprehensive GitHub PR reviews using the `gh` CLI. No Python dependencies — pure shell and `gh`.

## What it does

1. Fetches PR metadata, diff, inline comments, review history, and head SHA via `gh` CLI
2. Checks the project conventions file (`CLAUDE.md`, `CONTRIBUTING.md`) before reviewing
3. Reads test changes before implementation changes — the most common test quality failures show up first
4. Analyzes the diff against a standard review checklist (functionality, correctness, security, testing, style, PR quality)
5. Writes a detailed internal `review.md` and a clean `human.md` draft ready to post
6. Creates `/send` and `/send-decline` commands — nothing is posted to GitHub until you run one of them

## Skills

| File | Command | Purpose |
|---|---|---|
| `SKILL.md` | `/pr-review` | General-purpose PR review for any language |
| `SKILL-PERL.md` | `/pr-review-perl` | Perl-specific checks to run alongside `/pr-review` |

## Installation

Copy both skills to your global Claude Code commands directory:

```bash
# General skill
curl -fsSL https://raw.githubusercontent.com/beadon/pr-review-skill/main/SKILL.md \
  -o ~/.claude/commands/pr-review.md

# Perl sub-skill (add when reviewing Perl codebases)
curl -fsSL https://raw.githubusercontent.com/beadon/pr-review-skill/main/SKILL-PERL.md \
  -o ~/.claude/commands/pr-review-perl.md
```

Or clone and symlink:

```bash
git clone git@github.com:beadon/pr-review-skill.git ~/skills/pr-review-skill
ln -s ~/skills/pr-review-skill/SKILL.md ~/.claude/commands/pr-review.md
ln -s ~/skills/pr-review-skill/SKILL-PERL.md ~/.claude/commands/pr-review-perl.md
```

## Usage

For any PR:

```
/pr-review https://github.com/owner/repo/pull/123
/pr-review owner/repo 123
```

For Perl codebases, run the sub-skill first to load the additional checks, then invoke the main review:

```
/pr-review-perl
/pr-review https://github.com/owner/repo/pull/123
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

## License

AGPL-3.0-or-later. See [LICENSE](LICENSE).

## Acknowledgements

Checklist informed by [SpillwaveSolutions/pr-reviewer-skill](https://github.com/SpillwaveSolutions/pr-reviewer-skill), [ParadiseSS13/Paradise#21968](https://github.com/ParadiseSS13/Paradise/discussions/21968), practitioner discussion from r/rails, and [bhserna's review skill gist](https://gist.github.com/bhserna/831bc50ad38378813eee9d9407609cf7). See the References section in `SKILL.md` for full traceability.
