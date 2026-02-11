# Skill Change Evaluation — Design Document

## 1. Problem Statement

When the main agent edits skill files during a task, we want those changes evaluated and fixed automatically against skill-creator best practices. The main agent's context must be protected — it should not handle evaluation or fixing.

## 2. Decisions

| Question | Decision |
|----------|----------|
| Model | Hardcode `claude-opus-4-6` |
| Timeout | None (accept cost of Opus per Stop) |
| Architecture | Single agent (evaluate + fix in one pass) |
| Cost | Accept Opus firing every Stop |
| Notification | Yes — tell main agent to re-read updated skill |
| Fixes | Subagent handles fixes directly |

## 3. Architecture

```
Main agent edits skill file(s) as part of its task
        │
        ▼
PostToolUse COMMAND hook (mark-skill-change.sh)
  → Checks: is file in .claude/skills/?
  → Checks: is .skill-fix-in-progress set? (loop guard)
  → If both pass: appends path to .skill-changes-pending
        │
        ▼
Main agent finishes responding → Stop hook fires
        │
        ▼
Stop AGENT hook (Opus)
  → Reads .skill-changes-pending
  → If not found: return {"ok": true} immediately
  → If found:
    1. Read changed skill files
    2. Read skill-creator guidelines
    3. Evaluate each skill
    4. If issues: set .skill-fix-in-progress → fix files → delete marker
    5. Write audit log to .claude/skill-feedback/[skill-name].md
    6. Delete .skill-changes-pending
    7. Return {"ok": true} with notification
        │
        ▼
Main agent's next turn sees notification:
  "Skills [X, Y] were evaluated and updated by the skill evaluator.
   Re-read these skill files before using them again:
   - .claude/skills/X/SKILL.md
   - .claude/skills/Y/SKILL.md
   Audit log: .claude/skill-feedback/"
```

### Loop Prevention

1. Subagent sets `.skill-fix-in-progress` BEFORE editing skill files
2. Its edits trigger PostToolUse → command hook sees the guard marker → skips
3. Subagent deletes marker after editing
4. No `.skill-changes-pending` created → next Stop finds nothing → done

### Why Notification Matters

If the subagent fixes a skill file and the main agent uses that skill later in the conversation, the main agent needs to re-read the updated file. Without notification, it would work with stale skill content. The notification tells the main agent which files changed so it can reload them.

## 4. Implementation

### Files

- `.claude/settings.json` — Hook configuration (PostToolUse command + Stop agent)
- `.claude/hooks/mark-skill-change.sh` — Lightweight script to mark skill changes
- `.claude/skill-feedback/` — Directory for evaluation audit logs
- `.gitignore` — Excludes temporary marker files

### Hook Configuration

See `.claude/settings.json` for the full configuration.

### Stop Agent Prompt

The Stop agent prompt instructs the subagent to:
1. Check for `.skill-changes-pending`
2. Read changed skill files + skill-creator guidelines
3. Evaluate against: frontmatter, description completeness, progressive disclosure, references, no extraneous files
4. Fix issues (with loop guard marker)
5. Write audit log
6. Return notification for main agent to re-read updated files

## 5. Verification

1. **Non-skill edit:** Edit a Python file → PostToolUse fires → command hook exits silently → Stop fires → agent finds no pending → exits immediately
2. **Valid skill edit:** Edit a well-formed SKILL.md → marker created → agent evaluates → passes → notification says "all passed"
3. **Broken skill edit:** Remove description from a SKILL.md → agent evaluates → fails → fixes → notification says "updated, re-read"
4. **Loop prevention:** Agent's fixes don't create new marker → next Stop finds nothing
5. **Notification:** Main agent sees which files changed and knows to re-read them
6. **Audit log:** Check `.claude/skill-feedback/` for evaluation records
