# autoskill

A Claude Code skill that scans your transcripts and tells you which skills to create and what to add to your `CLAUDE.md`.

```
autoskill scan complete.

Sources: 47 sessions across 6 projects
Prompts analyzed: 312

Skills to create: 4
  - pre-merge-review (12 occurrences)
  - draft-blog-post (8 occurrences)
  - run-perf-audit (6 occurrences)
  - prep-pr-description (4 occurrences)

CLAUDE.md rules to add: 7
  - Never use em dashes in text/UI (5 occurrences)
  - Always use Edit not Write for existing files (4 occurrences)
  - Don't squash-merge large branches (3 occurrences)

Drafts: ./autoskill-output/
```

## Install

```bash
git clone https://github.com/Ami3466/autoskill.git ~/.claude/skills/autoskill
```

Claude Code auto-discovers skills in `~/.claude/skills/`.

## Usage

```
/autoskill
```

Or: `scan my logs and tell me what skills I should have`

## Output

```
autoskill-output/
  CLAUDE.md.suggestions.md     # rules with verbatim evidence
  skills/
    pre-merge-review/SKILL.md
    draft-blog-post/SKILL.md
```

Each skill draft includes trigger phrases and the procedure, drawn from how you actually do the task in your transcripts.

## How it works

1. Reads `~/.claude/projects/*/*.jsonl` (your transcripts)
2. Extracts user prompts, filters system noise
3. Clusters repeated workflows and repeated corrections
4. Drafts skill files and CLAUDE.md rules

No external APIs. No telemetry. Pure prompt. Your transcripts never leave your machine.

## Why

You keep typing the same long prompts. You keep correcting Claude the same way. Each one is a missing skill or a missing rule. `autoskill` finds them for you.

## License

MIT
