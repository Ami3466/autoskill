---
name: autoskill
description: Reads the user's Claude Code transcripts and recommends (1) skills to create from repeated workflows and (2) CLAUDE.md rules to add from repeated requests/corrections. Use when the user invokes /autoskill, says "scan my logs", "what skills should I have", "what should be in my CLAUDE.md", or asks autoskill to analyze their behavior.
---

# autoskill

Scans the user's Claude Code transcripts at `~/.claude/projects/*/*.jsonl` and produces two recommendations:

1. **Skills to create** - repeated multi-step *workflows* the user runs by hand.
2. **CLAUDE.md rules to add** - repeated *requests, corrections, or preferences* the user states across sessions.

Output goes to `./autoskill-output/` in the current working directory. The user reviews drafts and copies what they want into `~/.claude/skills/` or their `CLAUDE.md`.

The skill itself does the analysis - no external API, no Node deps. Claude reads the JSONL, clusters by reasoning, and writes drafts.

---

## When to use

Trigger on any of:
- `/autoskill`
- "scan my logs"
- "what skills should I have"
- "what should be in my CLAUDE.md"
- "find patterns in my Claude Code usage"
- "look at how I use Claude Code"

## When NOT to use

- One-off requests ("scan this file") - that's normal Read.
- Skill discovery for installing (use `find-skills` skill).
- Single-session analysis ("what did I do today") - that's a transcript read, not a behavioral scan.

---

## The workflow (execute in order)

### Step 1: Locate transcripts

Run:
```bash
ls ~/.claude/projects/
```

Each subdirectory is one project (path-encoded). Each `.jsonl` file inside is one session. There are usually dozens of sessions per project.

If the user says "scan project X", filter to that project's directory. Otherwise scan ALL projects.

### Step 2: Extract user prompts

For each `.jsonl` file, extract entries where `type == "user"` and the message has plain user content (not tool results).

Use `jq` for speed:

```bash
# Extract all user prompts across all projects, with session ID
for f in ~/.claude/projects/*/*.jsonl; do
  jq -r 'select(.type == "user" and (.message.content | type == "string")) | "\(.sessionId)\t\(.cwd // "")\t\(.message.content)"' "$f" 2>/dev/null
done > /tmp/autoskill-prompts.tsv
```

Some user content is an array (tool_result blocks). Filter to string-only content - those are the real human prompts.

If `/tmp/autoskill-prompts.tsv` is empty, fall back to:
```bash
jq -r 'select(.type == "user") | .message.content | if type == "string" then . else empty end' "$f"
```

### Step 3: Sample for analysis

Reading every prompt across hundreds of sessions blows the context window. Sample:

- Take the most recent ~500 prompts (last 30 days is usually enough).
- If a single prompt is over 2000 chars (a long instruction block), keep only the first 500 chars + last 200 chars - the middle is usually file content.
- Dedupe exact duplicates before analysis (same prompt run 10 times = 1 entry with count).

```bash
head -500 /tmp/autoskill-prompts.tsv | sort | uniq -c | sort -rn | head -200 > /tmp/autoskill-sample.tsv
```

### Step 4: Cluster (this is the LLM step - YOU do it)

Read the sample. Group prompts into clusters by intent. Two cluster types:

**Type A - Repeated workflows (-> skill candidates):**
A multi-step task the user runs more than once. Signals:
- Long prompt with steps/checklist (e.g. "1. do X, 2. do Y, 3. commit").
- Same opening phrase repeated ("Review the last commit", "Draft a blog post about", "Run pre-merge checks").
- Same tool sequence implied (git diff -> read files -> commit).
- 3+ occurrences across sessions.

**Type B - Repeated preferences/corrections (-> CLAUDE.md candidates):**
Short imperative statements about HOW to behave. Signals:
- Negation: "don't X", "never X", "stop Xing".
- Style rules: "use X", "always Y", "prefer Z".
- Tool restrictions: "don't run X", "never push without asking".
- 2+ occurrences (lower bar than skills - rules are higher signal per occurrence).

Ignore one-off requests, debugging chatter, and project-specific data.

### Step 5: Cross-check existing CLAUDE.md and skills

Before drafting, read what's already there so you don't propose duplicates:

```bash
cat ~/.claude/CLAUDE.md 2>/dev/null
ls ~/.claude/skills/
# Plus the project's own CLAUDE.md if scanning a single project
```

Skip any cluster that's already covered.

### Step 6: Write drafts

Create `./autoskill-output/` in the current working directory.

#### For each skill candidate

Write `./autoskill-output/skills/<kebab-name>/SKILL.md`:

```markdown
---
name: <kebab-name>
description: <one sentence: what it does + when to invoke. Be specific about trigger phrases.>
---

# <Title Case Name>

<One paragraph: what this skill produces and when to use it.>

## When to use
- <trigger phrase 1>
- <trigger phrase 2>

## Workflow

### Step 1: <action>
<Concrete commands or actions, drawn from how the user actually does this in their transcripts.>

### Step 2: ...

## Inputs
<What user must provide, if anything.>

## Output
<What the skill produces.>
```

The procedure should mirror what the user actually does in their transcripts - cite specific tools/commands they used. Don't invent steps.

#### For CLAUDE.md additions

Write a single file: `./autoskill-output/CLAUDE.md.suggestions.md`:

```markdown
# Suggested CLAUDE.md additions

Based on patterns found across <N> sessions. Copy any of these into your `~/.claude/CLAUDE.md` (global) or project `CLAUDE.md`.

## Rules to add

### <Short rule title>
<Imperative one-liner. E.g. "Never use em dashes (-) in any text or UI copy.">

**Evidence:** <count> occurrences across <count> sessions. Examples:
- "<verbatim user quote>"
- "<verbatim user quote>"

**Suggested location:** global / project (`<path>`)

---

### <Next rule>
...
```

Always include the verbatim evidence so the user can verify.

### Step 7: Summary report

After writing files, output to the user:

```
autoskill scan complete.

Sources: <N> sessions across <M> projects
Sample analyzed: <N> prompts

Skill candidates: <count>
  - <name> (<count> occurrences)
  - <name> (<count> occurrences)

CLAUDE.md rules: <count>
  - <rule title> (<count> occurrences)
  - <rule title> (<count> occurrences)

Drafts written to: ./autoskill-output/
Review and copy what you want.
```

Don't auto-install anything. The user moves files manually - safer, and they get to see what they're agreeing to.

---

## Quality bar (read before drafting)

- **Skills must be runnable** - the procedure should be concrete enough that Claude can execute it without asking the user follow-up questions.
- **Rules must be universal-ish** - if a "rule" only applies to one project, it's a project memory, not a CLAUDE.md rule. Note that in the suggestion (`Suggested location: project`).
- **Quote evidence** - never paraphrase the user's words when justifying a rule. Verbatim or skip.
- **Skip the obvious** - "user prefers concise responses" is not a useful rule. Look for non-default, surprising, project-shaping patterns.
- **No hallucination** - if you can't find 3+ workflow occurrences or 2+ rule occurrences, don't draft. Empty output is fine.

---

## Privacy note

Transcripts contain everything the user has ever typed to Claude Code, including secrets, credentials, and internal info. Do NOT:
- Send transcripts to any external API.
- Write transcript content to any file outside `./autoskill-output/`.
- Include raw prompts in the summary beyond short verbatim quotes used as evidence.

All analysis happens in this Claude session. The drafts you write are reviewed by the user before they go anywhere.
