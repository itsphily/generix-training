# Skill Change Evaluation — Design Document

## 1. Problem Statement

When the main agent edits skill files during a task, we want those changes evaluated and fixed automatically against skill-creator best practices. The main agent's context must be protected — it should not handle evaluation or fixing.

## 2. Decisions

| Question | Decision |
|----------|----------|
| Model | Inherit from main agent |
| Timeout | None |
| Architecture | One foreground subagent per modified skill |
| Cost | Zero latency when no skills edited (bash check only) |
| Notification | Yes — tell main agent to re-read updated skill |
| Fixes | Each subagent handles fixes for its skill |
| Permissions | Subagent can edit files in `.claude/skills/` only, user manually approves each edit |
| Background | No — foreground so user gets approval prompts (background auto-denies) |
| Audit logs | None — subagent prompts user to accept/edit changes directly |
| Skill evaluation | Each subagent must use `example-skills:skill-creator` (Anthropic plugin) |

## 3. Architecture (v2 — Command Hook + Per-Skill Subagents)

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
Main agent finishes responding → Stop COMMAND hook fires
        │
        ▼
Stop COMMAND hook (check-skill-changes.sh)
  → Checks: does .skill-changes-pending exist?
  → If NOT: exit silently (zero latency, zero cost)
  → If YES:
    1. Read file, extract unique skill directory names
    2. Rename .skill-changes-pending → .skill-changes-evaluating (prevents duplicates)
    3. Return {"decision": "block", "reason": "...list of skills to evaluate..."}
        │
        ▼
Main agent receives instruction to spawn subagents
  → Spawns ONE foreground Task subagent per unique skill
  → Each subagent:
    1. Invokes the example-skills:skill-creator skill
    2. Evaluates its assigned skill against skill-creator guidelines
    3. Proposes fixes (user approves/denies each edit)
    4. Manages .skill-fix-in-progress loop guard via Bash
    5. Returns result to main agent
  → Main agent resumes its primary task after subagent completes
        │
        ▼
Main agent's next turn sees notification:
  "Skill [X] was evaluated and updated.
   Re-read .claude/skills/X/SKILL.md before using it again."
```

### Loop Prevention

1. Subagent sets `.skill-fix-in-progress` BEFORE editing skill files
2. Its edits trigger PostToolUse → command hook sees the guard marker → skips
3. Subagent deletes marker after editing
4. No `.skill-changes-pending` created → next Stop finds nothing → done

### Rename Prevents Duplicate Subagents

1. Stop command hook renames `.skill-changes-pending` → `.skill-changes-evaluating`
2. Next Stop: checks for `.skill-changes-pending` → gone → exits silently
3. If new edits happen during evaluation, they create a fresh `.skill-changes-pending`
4. That new batch gets picked up on a future Stop — separate subagents

### Why Notification Matters

If a subagent fixes a skill file and the main agent uses that skill later in the conversation, the main agent needs to re-read the updated file. Without notification, it would work with stale skill content.

## 4. Implementation

### Files

- `.claude/settings.json` — Hook configuration (PostToolUse command + Stop command)
- `.claude/hooks/mark-skill-change.sh` — Marks skill file edits (PostToolUse)
- `.claude/hooks/check-skill-changes.sh` — Checks for pending changes (Stop)
- `.gitignore` — Excludes temporary marker files

**Important:** The `env` block (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`) must always be preserved when modifying settings.json.

## 5. Tests & Observations

### Test 1: Harmless edit (Stop agent hook — v1 architecture)
- **What:** Added "(requires user approval)" to analyzing-sessions/SKILL.md git integration section
- **Result:** Stop agent hook fired. Spinner showed "Creating... (running stop hook · 39s · ↓ 1.9k tokens)"
- **Observation:** 39s latency on every response is too much. The Opus agent hook fires on EVERY Stop, even when no skills were edited. Most of the time it just checks the marker and exits, but that's still an API call.
- **Decision:** Replace Stop agent hook with Stop command hook (bash check = instant, free). Only spawn subagents when there's actual work.

### Test 2: Command hook gate + background subagent (v2 architecture)
- **What:** Edited analyzing-sessions/SKILL.md to trigger the pipeline
- **Result — Command hook:** Worked instantly. Blocked the stop with JSON instructions listing the modified skills. Zero latency when no skills edited.
- **Result — Background subagent:** Spawned successfully, invoked skill-creator, evaluated the skill, found 4 issues. BUT all Write/Edit/Bash permissions were denied — background subagents cannot prompt the user for permission approval.
- **Observation:** `run_in_background: true` prevents user approval prompts. The subagent could read and evaluate but couldn't apply any fixes or manage marker files.
- **Decision:** Run subagents in foreground (no `run_in_background`). Trade-off: main agent waits during evaluation, but user retains manual approval of each edit. Subagent permissions scoped to skill folder files only.

### Test 3: Foreground subagent with user approval (v2 architecture, fixed)
- **What:** Edited analyzing-sessions/SKILL.md, spawned foreground subagent (no `run_in_background`)
- **Result:** Subagent successfully: invoked skill-creator, evaluated the skill, found 7 issues, applied all fixes with user approval, cleaned up both marker files.
- **Issues fixed:** (1) expanded frontmatter description with 5 trigger scenarios, (2) removed "When to Use" body section, (3) removed Rollback section, (4) condensed Git Integration to 4-line Rules section, (5) moved reflect.sh to scripts/, (6-7) fixed path references in reflect.sh
- **Observation:** Foreground subagents work correctly — user gets prompted for each edit. Main agent waits during evaluation (~4 min for Opus) but resumes after.
- **Loop guard:** Worked — subagent's edits to SKILL.md did NOT trigger new .skill-changes-pending entries because .skill-fix-in-progress was set.
- **Note:** This test was semi-manual (subagent spawned before Stop hook fired). In real usage, the Stop hook handles the .skill-changes-pending → .skill-changes-evaluating rename before the main agent spawns subagents.

### Test 4: Full end-to-end pipeline (v2 architecture, foreground)
- **What:** Edited analyzing-sessions/SKILL.md (minor heading change). Let the Stop hook fire naturally.
- **Result:** Full pipeline worked end-to-end:
  1. PostToolUse hook created `.skill-changes-pending` ✓
  2. Stop hook detected it, renamed to `.skill-changes-evaluating`, blocked with JSON instructions ✓
  3. Main agent spawned foreground subagent per instructions ✓
  4. Subagent invoked skill-creator, evaluated skill, found 1 issue (dead `scripts/reflect.sh`), removed it with user approval ✓
  5. Subagent cleaned up both marker files ✓
- **Observation:** ~84s total for evaluation (Opus). Acceptable since it only fires when skills are actually edited.
- **Loop guard:** Confirmed working — subagent's file deletions did not trigger new `.skill-changes-pending`.
- **Status:** PASSING — v2 architecture validated end-to-end.

---

## 6. Detailed Implementation Guide

This section contains everything needed to implement the skill auto-evaluation pipeline from scratch. All code is production-tested.

### 6.1 Prerequisites

- **Claude Code CLI** (v2.1+)
- **`jq`** installed on the system (used by hooks to parse JSON input)
- **`example-skills` plugin** enabled (provides the `skill-creator` skill used for evaluation)
- At least one skill in `.claude/skills/` to evaluate

### 6.2 Directory Structure

After implementation, your project will have these files:

```
your-project/
├── .claude/
│   ├── settings.json              ← Hook configuration (project-level)
│   ├── settings.local.json        ← Permission rules (user-level, not committed)
│   ├── hooks/
│   │   ├── mark-skill-change.sh   ← PostToolUse hook (detects skill edits)
│   │   └── check-skill-changes.sh ← Stop hook (gates subagent spawning)
│   ├── skills/
│   │   └── your-skill/
│   │       └── SKILL.md           ← The skill files being evaluated
│   │
│   │  (Temporary marker files — gitignored)
│   ├── .skill-changes-pending     ← Created by mark-skill-change.sh
│   ├── .skill-changes-evaluating  ← Created by check-skill-changes.sh (renamed from pending)
│   └── .skill-fix-in-progress     ← Created by subagent as loop guard
│
└── .gitignore                     ← Must exclude the 3 marker files
```

### 6.3 File 1: `.claude/settings.json`

This is the central configuration file. It wires up both hooks and enables the required plugin.

**Location:** `.claude/settings.json` (committed to git — shared across the team)

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/mark-skill-change.sh"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/check-skill-changes.sh"
          }
        ]
      }
    ]
  },
  "enabledPlugins": {
    "example-skills@anthropic-agent-skills": true
  }
}
```

**Key details:**

| Field | Purpose |
|-------|---------|
| `hooks.PostToolUse` | Fires AFTER every Write or Edit tool call |
| `hooks.PostToolUse[].matcher` | `"Write\|Edit"` — only triggers on file-modifying tools |
| `hooks.PostToolUse[].hooks[].type` | `"command"` — runs a shell script (zero-latency, free) |
| `hooks.Stop` | Fires when the agent finishes a response and is about to stop |
| `hooks.Stop[].hooks[].type` | `"command"` — bash check, not an agent call (avoids 39s Opus latency) |
| `$CLAUDE_PROJECT_DIR` | Built-in variable — resolves to the project root at runtime |
| `enabledPlugins` | `example-skills@anthropic-agent-skills` provides the `skill-creator` skill |

**Why `command` type and NOT `agent` type for Stop:**

We tested a Stop `agent` hook first (v1 architecture). It used Opus to check if skills were modified. Problem: the agent hook fires on EVERY Stop — even when no skills were edited. That added ~39 seconds of latency to every single response. A `command` hook (bash script) checks in <1ms and costs nothing.

### 6.4 File 2: `.claude/hooks/mark-skill-change.sh`

This PostToolUse hook detects when a file inside `.claude/skills/` is modified and appends it to a pending list.

**Location:** `.claude/hooks/mark-skill-change.sh`
**Trigger:** Every time Claude uses the `Write` or `Edit` tool
**Must be executable:** `chmod +x .claude/hooks/mark-skill-change.sh`

```bash
#!/bin/bash
# Hook: PostToolUse for Write|Edit
# Detects when a file inside .claude/skills/ is created or modified,
# and marks it for evaluation by the Stop agent hook.

INPUT=$(cat)
FILE_PATH=$(jq -r '.tool_input.file_path // empty' <<< "$INPUT")

# Exit if no file path or not a skill file
if [[ -z "$FILE_PATH" ]] || [[ "$FILE_PATH" != *"/.claude/skills/"* ]]; then
  exit 0
fi

# Exit if fixer is currently working (loop prevention)
PROJECT_DIR=$(jq -r '.cwd // empty' <<< "$INPUT")
if [[ -z "$PROJECT_DIR" ]]; then
  PROJECT_DIR="$CLAUDE_PROJECT_DIR"
fi

if [[ -f "$PROJECT_DIR/.claude/.skill-fix-in-progress" ]]; then
  exit 0
fi

# Append the changed file path to pending list (deduplicated later by agent)
echo "$FILE_PATH" >> "$PROJECT_DIR/.claude/.skill-changes-pending"
exit 0
```

**How it works, line by line:**

1. **`INPUT=$(cat)`** — PostToolUse hooks receive a JSON payload on stdin containing the tool name, input, output, and current working directory.

2. **`FILE_PATH=$(jq -r '.tool_input.file_path // empty' <<< "$INPUT")`** — Extracts the file path from the tool input. Both `Write` and `Edit` tools have a `file_path` parameter.

3. **First guard — Is it a skill file?**
   ```bash
   if [[ "$FILE_PATH" != *"/.claude/skills/"* ]]; then exit 0; fi
   ```
   If the edited file is NOT inside `.claude/skills/`, exit silently. This means editing Python files, configs, etc. never triggers anything.

4. **Second guard — Is the fixer already working? (Loop prevention)**
   ```bash
   if [[ -f "$PROJECT_DIR/.claude/.skill-fix-in-progress" ]]; then exit 0; fi
   ```
   When the evaluator subagent is fixing a skill, it sets this marker file BEFORE editing. This prevents the subagent's own edits from creating new pending entries — which would cause an infinite loop.

5. **Append to pending list**
   ```bash
   echo "$FILE_PATH" >> "$PROJECT_DIR/.claude/.skill-changes-pending"
   ```
   Appends the full file path. Multiple edits to the same skill (or different skills) in one response all get logged. Deduplication happens in the Stop hook.

**PostToolUse JSON input format** (what the hook receives on stdin):

```json
{
  "tool_name": "Edit",
  "tool_input": {
    "file_path": "/absolute/path/to/.claude/skills/my-skill/SKILL.md",
    "old_string": "...",
    "new_string": "..."
  },
  "tool_output": "The file has been updated successfully.",
  "cwd": "/absolute/path/to/project"
}
```

### 6.5 File 3: `.claude/hooks/check-skill-changes.sh`

This Stop hook checks if any skills were modified during the response. If yes, it blocks the stop and tells the main agent to spawn evaluator subagents.

**Location:** `.claude/hooks/check-skill-changes.sh`
**Trigger:** Every time the agent finishes a response
**Must be executable:** `chmod +x .claude/hooks/check-skill-changes.sh`

```bash
#!/bin/bash
# Hook: Stop
# Checks if any skill files were modified during the response.
# If yes, blocks the stop and tells the main agent to spawn
# one background subagent per modified skill for evaluation.

PENDING="$CLAUDE_PROJECT_DIR/.claude/.skill-changes-pending"

# If no pending changes, exit silently (zero cost, zero latency)
if [ ! -f "$PENDING" ]; then
  exit 0
fi

# Extract unique skill directory names from the pending list
SKILLS=$(sort -u "$PENDING" | sed -n 's|.*/.claude/skills/\([^/]*\).*|\1|p' | sort -u | tr '\n' ', ' | sed 's/,$//')

# Rename to prevent duplicate subagents on next Stop
mv "$PENDING" "$CLAUDE_PROJECT_DIR/.claude/.skill-changes-evaluating"

# Block the stop — tell main agent to spawn per-skill subagents
printf '{"decision":"block","reason":"Skill files were modified: [%s]. For EACH skill listed, spawn a separate Task subagent (subagent_type: general-purpose, do NOT use run_in_background). Each subagent must: (1) invoke the example-skills:skill-creator skill, (2) evaluate the skill at .claude/skills/[skill-name]/ using the skill-creator guidelines, (3) if fixes needed, create .claude/.skill-fix-in-progress before editing and delete it after — edits ONLY to files in .claude/skills/[skill-name]/, (4) delete .claude/.skill-changes-evaluating when all evaluations complete. After all subagents finish, continue your current task."}' "$SKILLS"
```

**How it works, line by line:**

1. **Fast exit when nothing to do:**
   ```bash
   if [ ! -f "$PENDING" ]; then exit 0; fi
   ```
   If `.skill-changes-pending` doesn't exist, no skills were edited → exit immediately. This runs on EVERY response and completes in <1ms.

2. **Extract unique skill names:**
   ```bash
   SKILLS=$(sort -u "$PENDING" | sed -n 's|.*/.claude/skills/\([^/]*\).*|\1|p' | sort -u | tr '\n' ', ' | sed 's/,$//')
   ```
   The pending file might contain:
   ```
   /path/.claude/skills/my-skill/SKILL.md
   /path/.claude/skills/my-skill/references/schema.md
   /path/.claude/skills/other-skill/SKILL.md
   ```
   This extracts → `my-skill, other-skill` (unique skill directory names, comma-separated).

3. **Rename to prevent duplicates:**
   ```bash
   mv "$PENDING" "$CLAUDE_PROJECT_DIR/.claude/.skill-changes-evaluating"
   ```
   Atomic rename. The next Stop hook will not find `.skill-changes-pending` → exits silently. If new skill edits happen during evaluation, they create a fresh `.skill-changes-pending` that gets picked up on a future Stop.

4. **Block the stop with JSON:**
   ```bash
   printf '{"decision":"block","reason":"..."}'
   ```
   The `{"decision":"block","reason":"..."}` JSON format is the Claude Code hook protocol for blocking a stop. The `reason` string becomes the instruction to the main agent. We embed the full subagent prompt here.

**Stop hook output protocol:**

| Output | Effect |
|--------|--------|
| Empty / exit 0 | Agent stops normally |
| `{"decision":"block","reason":"..."}` | Agent does NOT stop — receives the reason as a system message and continues |
| Any non-JSON text to stderr | Displayed as error to user |

### 6.6 File 4: `.gitignore` additions

Add these lines to your `.gitignore` to exclude the temporary marker files:

```gitignore
# Claude Code skill evaluation temp files
.claude/.skill-changes-pending
.claude/.skill-changes-evaluating
.claude/.skill-fix-in-progress
```

These files are transient — created and deleted during evaluation. They must not be committed.

### 6.7 Permissions Model

#### Project-level permissions (`.claude/settings.json`)

The settings.json hooks fire for ALL users of the project. No special permissions are needed for the hooks themselves — they are `command` type (bash scripts), which always execute.

#### User-level permissions (`.claude/settings.local.json`)

This file is per-user and NOT committed to git. It controls what tools Claude can use without prompting.

**Critical for this pipeline:** The subagent needs `Write`, `Edit`, and `Bash` permissions to fix skill files and manage marker files. There are two approaches:

**Approach A — Manual approval (recommended):**
Do NOT pre-approve Write/Edit/Bash in settings.local.json. The subagent runs in the foreground, and the user is prompted to approve or deny each edit individually. This gives full control over what changes are applied.

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Grep",
      "Bash(ls *)",
      "Bash(touch *)",
      "Bash(rm *)"
    ],
    "deny": [],
    "ask": []
  }
}
```

**Approach B — Auto-approve (faster, less control):**
Pre-approve Write and Edit for skill files. The subagent applies fixes without prompting.

> Note: Claude Code permissions do not support path-scoped Write/Edit rules (as of v2.1). Auto-approving `Write` or `Edit` approves them for ALL files, not just skill files. Use Approach A if you want edits scoped to skill folders only.

**Why NOT `run_in_background: true`:**

Background subagents cannot prompt the user for permissions. Any tool call that requires approval (Write, Edit, Bash) is automatically denied. We tested this (Test 2) and confirmed all write operations failed silently. Foreground subagents block the main agent but allow user interaction.

### 6.8 The Subagent Prompt

When the Stop hook blocks, the main agent reads the JSON reason and spawns a Task subagent. Here is the prompt template the main agent should use (the hook's reason string tells it to do this):

```
You are a skill evaluator subagent. Your job is to evaluate a Claude Code
skill against the skill-creator best practices.

**IMPORTANT SETUP STEPS — do these first:**
1. Invoke the `example-skills:skill-creator` skill using the Skill tool
   (skill: "example-skills:skill-creator")
2. Create the loop guard file:
   touch "$PROJECT_DIR/.claude/.skill-fix-in-progress"

**Skill to evaluate:** `.claude/skills/[SKILL-NAME]/`
- Read ALL files in the skill directory
- Evaluate the skill against the skill-creator guidelines (from step 1)

**Evaluation criteria:**
- Frontmatter has `name` and comprehensive `description` that covers ALL
  trigger scenarios
- Description is the primary triggering mechanism — all "when to use" info
  must be in the description, NOT in the body
- Body is concise (<500 lines), follows progressive disclosure
- No extraneous files (no README, CHANGELOG, etc.)
- References properly linked from SKILL.md
- Claude already knows basic things — don't document what's obvious

**If fixes are needed:**
- Edit ONLY files within `.claude/skills/[SKILL-NAME]/`
- Show each proposed change clearly before applying
- After all fixes: delete the loop guard
  rm "$PROJECT_DIR/.claude/.skill-fix-in-progress"

**If NO fixes are needed:**
- Still delete the loop guard:
  rm "$PROJECT_DIR/.claude/.skill-fix-in-progress"

**When done:**
- Delete the evaluating marker:
  rm "$PROJECT_DIR/.claude/.skill-changes-evaluating"
- Return a summary of what was evaluated and what was fixed
```

**Key aspects of the prompt:**

1. **Invokes the skill-creator skill first** — this loads the evaluation guidelines into the subagent's context. Without this, the subagent would not know the skill-creator best practices.

2. **Creates `.skill-fix-in-progress` before any edits** — this is the loop guard. The PostToolUse hook checks for this file and skips if present, preventing the subagent's own edits from triggering another evaluation cycle.

3. **Scoped to one skill** — each subagent evaluates only its assigned skill. If 3 skills were modified, 3 separate subagents are spawned (sequentially in foreground mode).

4. **Cleans up marker files** — the subagent is responsible for deleting both `.skill-fix-in-progress` (its own guard) and `.skill-changes-evaluating` (the batch marker).

### 6.9 Loop Prevention — Complete Flow

This is the most critical safety mechanism. Without it, the system enters an infinite loop:

```
Edit skill → hook marks change → Stop blocks → spawn subagent →
subagent edits skill → hook marks change → Stop blocks → spawn subagent → ...
```

**How the loop is broken:**

```
1. Main agent edits .claude/skills/my-skill/SKILL.md
   → mark-skill-change.sh fires
   → Checks: does .skill-fix-in-progress exist? NO
   → Appends to .skill-changes-pending ✓

2. Main agent's response ends → Stop hook fires
   → check-skill-changes.sh finds .skill-changes-pending
   → Renames to .skill-changes-evaluating
   → Returns {"decision":"block"} with subagent instructions

3. Main agent spawns evaluator subagent
   → Subagent runs: touch .skill-fix-in-progress    ← GUARD SET

4. Subagent edits .claude/skills/my-skill/SKILL.md (fixing issues)
   → mark-skill-change.sh fires
   → Checks: does .skill-fix-in-progress exist? YES
   → EXIT SILENTLY (no .skill-changes-pending created) ← LOOP BROKEN

5. Subagent finishes
   → Deletes .skill-fix-in-progress                  ← GUARD CLEARED
   → Deletes .skill-changes-evaluating                ← BATCH CLEARED

6. Subagent returns to main agent
   → Next Stop: check-skill-changes.sh
   → .skill-changes-pending does NOT exist
   → Exit silently → agent stops normally              ← CLEAN EXIT
```

### 6.10 How to Set Up From Scratch (Step by Step)

#### Step 1: Create the hooks directory

```bash
mkdir -p .claude/hooks
```

#### Step 2: Create `mark-skill-change.sh`

Copy the code from Section 6.4 into `.claude/hooks/mark-skill-change.sh`.

```bash
chmod +x .claude/hooks/mark-skill-change.sh
```

#### Step 3: Create `check-skill-changes.sh`

Copy the code from Section 6.5 into `.claude/hooks/check-skill-changes.sh`.

```bash
chmod +x .claude/hooks/check-skill-changes.sh
```

#### Step 4: Configure hooks in `settings.json`

Copy the JSON from Section 6.3 into `.claude/settings.json`. If you already have a settings.json, merge the `hooks` block into it. Preserve any existing `env` or `enabledPlugins` entries.

#### Step 5: Enable the skill-creator plugin

Make sure `example-skills@anthropic-agent-skills` is in the `enabledPlugins` block. This provides the evaluation guidelines the subagent uses.

#### Step 6: Update `.gitignore`

Add the three marker file entries from Section 6.6.

#### Step 7: Verify `jq` is installed

```bash
which jq || echo "Install jq: brew install jq (macOS) or apt install jq (Linux)"
```

The PostToolUse hook uses `jq` to parse JSON input from Claude Code.

#### Step 8: Test

1. Edit any file inside `.claude/skills/` (e.g., change a word in a SKILL.md)
2. Let the agent finish responding
3. The Stop hook should block with a JSON message listing the modified skills
4. The agent should spawn a foreground subagent
5. The subagent evaluates the skill and proposes fixes (you approve/deny each one)
6. Both marker files should be cleaned up after

### 6.11 Customization Points

| What | How to customize |
|------|-----------------|
| **Which tools trigger detection** | Change the `matcher` in PostToolUse — e.g., `"Write\|Edit\|NotebookEdit"` |
| **Which files are tracked** | Change the path check in mark-skill-change.sh — e.g., `*"/.claude/commands/"*` |
| **Evaluation criteria** | Modify the subagent prompt in check-skill-changes.sh's `printf` statement |
| **Subagent model** | Add `model` parameter to the Task tool call — e.g., `"model": "sonnet"` for faster/cheaper evaluation |
| **Auto-approve edits** | Spawn with `mode: "acceptEdits"` (still requires foreground, but skips per-edit approval) |
| **Multiple skills per subagent** | Change the hook to pass all skills to one subagent instead of one-per-skill (less isolation, more context-efficient) |
| **Different evaluation plugin** | Replace `example-skills:skill-creator` with your own evaluation skill |

### 6.12 Gotchas and Lessons Learned

**1. Stop `agent` hooks add latency to EVERY response.**
Even if the agent just checks a file and exits, it's still an API call (~39s for Opus). Use `command` hooks for fast checks, and only spawn agents when there's actual work.

**2. Background subagents cannot prompt for user permissions.**
Any tool call requiring approval (Write, Edit, Bash) is automatically denied. Use foreground subagents if user approval is needed.

**3. Always preserve existing `settings.json` fields when editing.**
It's easy to accidentally delete the `env` block or `enabledPlugins` when rewriting the file. Read-then-edit, never overwrite.

**4. The `$CLAUDE_PROJECT_DIR` variable resolves at runtime.**
Don't hardcode absolute paths in settings.json. Use `$CLAUDE_PROJECT_DIR` so the hooks work on any machine.

**5. Marker files must be gitignored.**
If committed, they would persist across sessions and trigger false evaluations. They are transient — created and deleted within a single conversation.

**6. The `chmod +x` step is required.**
Hook scripts must be executable. Claude Code runs them directly, not via `bash script.sh`.

**7. The subagent's Skill invocation is essential.**
Without `Skill("example-skills:skill-creator")`, the subagent does not have the evaluation guidelines in context. It would evaluate based on general knowledge, not the specific skill-creator best practices.

**8. Hook scripts must exit 0 on success.**
A non-zero exit code is treated as an error and displayed to the user. Even the "skip" path should `exit 0`.

**9. PostToolUse hooks receive JSON on stdin, not as arguments.**
Use `INPUT=$(cat)` to capture it, then parse with `jq`. The `$CLAUDE_PROJECT_DIR` environment variable is available as a fallback for the project directory.

**10. The Stop hook protocol uses `{"decision":"block"}` to prevent stopping.**
Any other output (or no output) allows the agent to stop normally. The `reason` field becomes a system message that the agent acts on.
