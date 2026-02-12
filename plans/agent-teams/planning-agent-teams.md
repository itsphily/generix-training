# Planning Agent Teams - System Design

## Quick Overview

Three separate Claude Code skills that spawn agent teams to plan and implement new features:

| Command | Purpose | Output |
|---------|---------|--------|
| `/plan-overview` | Architecture overview of a proposed feature | `apps/plans/{feature}/overview.md` |
| `/plan-spec` | File-level detailed spec from the overview | `apps/plans/{feature}/spec.md` |
| `/plan-implement` | Implementation from the spec | Code changes to codebase |

Each command is a **separate team** with a separate context window. The plan file is the handoff artifact between phases. The user reviews and approves before proceeding to the next phase.

**Key design principles** (validated by external research):
- Manager orchestrates, does not execute (CrewAI lesson, Anthropic guidance)
- Narrow scope per agent - each gets a specific research mandate (Osmani, Lance Martin)
- **Fan-out/fan-in architecture**: Parallelize research (many agents), minimize synthesis (1-2 agents). See [Agent Scaling Evidence](#agent-scaling-evidence)
- No fixed agent count - team lead adapts based on task decomposability (Anthropic official docs: 2-3 for most tasks, scale up for parallelizable research)
- Constructive adversarial review, not maximal disagreement (debate research)
- Three-tier substantiation: Verified / Pattern-based / [NEEDS TESTING]
- Pause and wait on escalation - disputed items block until user decides

---

## Phase 1: `/plan-overview`

### Purpose
Analyze a feature request and produce a high-level architecture overview. The overview is written for the user to quickly understand the proposed approach, make decisions, and approve before detailed spec work begins.

### Internal Workflow (Three Stages)

The team uses a **three-stage workflow**: Interview → Research → Synthesis + Review.

#### Stage 1: Expert Interview (fan-out questions, fan-in to user)

Before research begins, the team lead spawns 2-3 lightweight **interview prep agents** that each analyze the feature request from a different perspective. Each agent reads relevant parts of the codebase and generates a list of deep, non-obvious questions the user should answer.

This integrates the `/improve-plan` interview pattern but with **codebase-informed questions from multiple expert perspectives** instead of a single generalist interviewer.

**How it works**:
1. Team lead receives the feature request
2. Team lead spawns interview prep agents in parallel (e.g., backend-focused, frontend-focused, database-focused - based on what the feature likely touches)
3. Each agent reads relevant code, then sends the team lead a list of 3-5 targeted questions with context for why each question matters
4. Team lead compiles, deduplicates, and prioritizes the best questions
5. Team lead interviews the user via AskUserQuestion (iterative, just like improve-plan)
6. Team lead writes the refined feature spec to `apps/plans/{feature-name}/interview.md`
7. Interview prep agents are shut down

**Why this is better than a single interviewer**: Questions are grounded in actual codebase knowledge. A backend-focused agent might ask "Does this need a new ContextSchema field?" because it read `configuration.py`. A frontend agent might ask "Should this be company-configurable?" because it read `company_table_features`. A single interviewer without codebase context would miss these.

**Interview prep agents are lightweight**: They only read code and generate questions - no research reports, no synthesis. They shut down before Stage 2 begins.

#### Stage 2: Research (fan-out)

Spawn multiple parallel researchers for independent data gathering. The refined feature spec from Stage 1 gives researchers much better direction than a raw feature request. Research tasks are embarrassingly parallel - each agent reads different files/docs independently. More agents = more coverage, no coordination overhead.

#### Stage 3: Synthesis + Review (fan-in)

The team lead alone synthesizes all findings into the plan document. Planning/synthesis is sequential reasoning that DEGRADES with more agents (Google DeepMind PlanCraft: -39% to -70% across all multi-agent variants). Single review agent then challenges the synthesized plan.

### Team Composition

**Minimum team**: Team Lead + 1 Codebase Researcher + 1 Review Agent
**Typical team**: Team Lead + 2-3 Codebase Researchers + 1 External Researcher + 1 Review Agent
**Complex cross-cutting features**: Team Lead + multiple Codebase Researchers + External Researchers + 1 Review Agent

No hardcoded maximum. The team lead spawns as many researchers as there are genuinely independent research areas. As the codebase grows and Anthropic improves agent team capabilities, the optimal count will naturally scale up.

The team lead decides the researcher count based on how many **independent research areas** the feature touches. Each codebase researcher gets a distinct domain to protect context windows and enable parallel coverage:

| Feature Type | Codebase Researchers | Example Scopes |
|-------------|---------------------|----------------|
| Backend-only change | 1-2 | Agent code + tools; Config + state |
| Frontend-only change | 1-2 | Components + registries; Providers + services |
| Cross-cutting feature | 3-4 | Backend/LangGraph; Frontend/components; Database/Supabase; Plans/history |
| New agent or graph | 2-3 | Existing agents (pattern reference); Tools + state; Configuration + deployment |

**Why multiple codebase researchers**: Each agent has its own context window. One researcher reading the entire codebase would hit context rot - models get worse as context grows (Lance Martin, Addy Osmani). Codebase research is embarrassingly parallel: reading backend code is independent from reading frontend code. The DeepMind study showed +81% improvement on parallelizable tasks. Our own validation session confirmed this - 3 codebase researchers (backend, frontend, supabase) produced detailed non-overlapping reports.

#### Team Lead (Architect-Orchestrator)
- **Role**: Decomposes the feature request, assigns research tasks, mediates disputes, synthesizes findings into the plan document
- **Model**: Inherit from parent (best available)
- **Permission mode**: `delegate` - restricted to coordination tools only (TaskCreate, TaskUpdate, SendMessage, Read). No Write/Edit except for the plan file and scratch directory
- **Skills**: All project skills available via progressive disclosure (but team lead should focus on orchestration, not research)
- **Behavior**:
  1. Receive feature request from user
  2. **Stage 1 - Interview**: Spawn interview prep agents to generate codebase-informed questions, then interview the user via AskUserQuestion. Write refined spec to `apps/plans/{feature-name}/interview.md`. Shut down interview agents.
  3. **Stage 2 - Research**: Analyze which researchers are needed based on the refined spec (auto-adapt). Create tasks and spawn researchers in parallel.
  4. Monitor progress, facilitate cross-domain communication between researchers
  5. **Stage 3 - Synthesis**: Synthesize all research into the overview document. Send draft to review agent for adversarial challenge. Incorporate valid challenges.
  6. Self-review against quality checklist before delivering
- **Quality Checklist** (applied before delivering plan):
  - [ ] Every architectural decision cites a source or marks [NEEDS TESTING]
  - [ ] Simpler alternatives considered for each component
  - [ ] No unnecessary coupling between components
  - [ ] Security implications addressed
  - [ ] Scope proportional to feature request
  - [ ] Existing codebase patterns respected
  - [ ] Decisions Log section is complete

#### Codebase Researchers (one per independent research domain)
- **Role**: Deep-dive into a specific domain of the NuevoDia codebase. Each researcher gets a non-overlapping scope to protect context windows and enable parallel coverage.
- **Model**: Inherit from parent
- **Permission mode**: `dontAsk` (read-only in practice - no Write/Edit to existing files)
- **Tools**: Read, Glob, Grep, LSP, Bash (read-only commands)
- **Skills**: All project skills are automatically available to every teammate through progressive disclosure (only descriptions loaded at ~100 tokens each; full skill body loads on-demand when relevant). No per-agent skill assignment is needed or possible in agent teams.
- **How many to spawn**: The team lead decides based on how many independent research domains the feature touches. No hardcoded limit - as the codebase grows and agent team capabilities improve, the optimal count will shift. The principle is: one researcher per genuinely independent research area.
- **Scope**: The team lead assigns each researcher a specific, non-overlapping domain. Examples:
  - **Backend researcher**: "Research how agents handle the runtime context pattern and tool filtering"
  - **Frontend researcher**: "Research the table feature registry system and how modals are triggered"
  - **Database researcher**: "Research the company_table_features schema and how features are toggled per company"
  - **Plans researcher**: "Read all plans in apps/plans/ related to {feature area} and summarize past decisions and lessons"
- **Output**: Structured research report with file paths, line references, and patterns found
- **Cross-domain queries**: Researchers can message each other directly via SendMessage to ask about code in another researcher's domain. This is preferred over reading files outside your assigned domain - it keeps each agent's context window clean while enabling cross-domain discovery. Examples:
  - Frontend researcher to backend researcher: "How does the `/api/crm/create-contact` endpoint work? I found the frontend calls it from AddContactModal."
  - Backend researcher to database researcher: "What columns does `company_table_features` have? I need to understand what config the table agent reads."
  - Database researcher to plans researcher: "Were there any past attempts at adding RLS to axxel_hr? I see it's missing."
- **Coordination**: Researchers also message each other to avoid reading overlapping files and to share discoveries that might be relevant to another domain

#### External Researcher
- **Role**: Search official documentation, existing plans, and external sources to find patterns, precedents, and best practices relevant to the feature
- **Model**: Inherit from parent
- **Permission mode**: `dontAsk`
- **Tools**: Read, Glob, Grep, WebSearch, WebFetch, Bash (read-only)
- **Skills**: All project skills available via progressive disclosure (will auto-trigger `researching-documentation` when searching external docs)
- **Scope**: Assigned by team lead. Examples:
  - "Search LangGraph documentation for the interrupt pattern"
  - "Read existing plans in apps/plans/ for similar features attempted before"
  - "Find official Supabase documentation on RLS policies for multi-tenant apps"
- **Substantiation rule**: Every finding must include the source URL or file path. Findings from unofficial sources must be flagged as [UNVERIFIED]
- **Output**: Structured research report with sources and credibility ratings

#### Review Agent
- **Role**: Constructive adversarial review of all research findings and the draft plan
- **Model**: Inherit from parent
- **Permission mode**: `dontAsk`
- **Tools**: Read, Glob, Grep, LSP (spot-checks claims against code)
- **Skills**: All project skills available via progressive disclosure (will auto-trigger relevant skills when spot-checking claims)
- **Mandate**: Challenge decisions that:
  - Lack substantiation (no source cited)
  - Add unnecessary complexity (YAGNI violation)
  - Deviate from established codebase patterns without justification
  - Introduce security risks
  - Miss simpler alternatives
  - Have cost/breaking-change implications
- **Does NOT challenge**: Style preferences, naming conventions, or decisions with clear substantiation
- **Output**: Critical review with CONFIRMED/CHALLENGED/GAPS sections
- **Steelman rule**: When challenging the team lead's decisions, the reviewer must first articulate the strongest argument FOR the decision before presenting the counter-argument

### Dynamic Specialist Spawning

The team lead can spawn additional specialist agents if the feature request requires domain expertise beyond what the core researchers cover. All agents automatically have access to all project skills via progressive disclosure - no manual skill assignment needed.

| Specialist | When to Spawn | Research Focus |
|-----------|--------------|----------------|
| LangGraph Specialist | Feature involves new graph nodes, agent patterns, or runtime context changes | LangGraph docs (MCP), existing agent code, langgraph.json |
| Supabase Specialist | Feature requires new tables, migrations, RLS policies, or schema changes | Database schema, migration patterns, RLS policies |
| Security Specialist | Feature touches auth, RLS, API keys, or data access patterns | Auth module, RLS status, data access patterns |
| Frontend Specialist | Feature requires new components, registries, or provider changes | Component patterns, registries, provider hierarchy |
| Deep Research Specialist | Feature modifies the deep research agent or batch processing | Orchestrator, researcher, batch processing code |

**Spawning rules**:
- Team lead decides autonomously (no user approval needed)
- No hard maximum, but each additional agent must have an independent research mandate (not overlapping with existing agents)
- Diminishing returns apply: each agent adds coordination overhead. Only spawn when the research area is genuinely independent from what other agents are already covering
- Each specialist gets a precise, scoped mandate
- Specialist reports back to team lead, not directly to other agents
- All research agents run in **parallel** (fan-out). The team lead waits for all to complete before starting synthesis (fan-in)

### Communication Guidelines

Agents communicate naturally using SendMessage, TaskCreate, and TaskUpdate. No rigid protocol is prescribed - agents will figure out effective communication patterns based on their roles and tasks. Two design choices matter:

1. **Review agent reviews the synthesized plan, not raw research reports** - the team lead synthesizes first, then sends the draft plan for review. This avoids polluting the reviewer's context with research details, keeping its focus on challenging the plan itself.
2. **Escalations to the user go through AskUserQuestion** - when the team lead needs a user decision, it uses AskUserQuestion with structured options and context.

### Escalation Rules

The team lead pauses work on the disputed item and asks the user when:

| Trigger | Example | Resolution |
|---------|---------|------------|
| **Changes to plan scope** | "This feature also requires changing the auth system" | User decides if scope expansion is acceptable |
| **Can't be done** | "The Supabase schema doesn't support this without a breaking migration" | User decides on alternative approach |
| **Security implications** | "This requires disabling RLS on a table" | User accepts risk or chooses safer alternative |
| **Breaking changes** | "This changes the company_config JSONB structure" | User decides if migration is acceptable |
| **Cost implications** | "This requires a new Supabase Edge Function" | User accepts cost |
| **Unresolved dispute** | Team lead and review agent disagree after steelman + counter-argument | User breaks the tie |

**Pause behavior**: Only the disputed item is paused. Non-dependent work continues. The team lead sends the user a structured question with context and options via AskUserQuestion.

### Plan Document Structure

The team lead produces the overview at `apps/plans/{feature-name}/overview.md`:

```markdown
# {Feature Name} - Architecture Overview

## Quick Summary
{2-3 sentences: what this feature does, why, and the recommended approach}

## Scope
- **In scope**: {bulleted list}
- **Out of scope**: {bulleted list}
- **Assumptions**: {bulleted list}

## Architecture
{High-level description of the approach. References to existing patterns in the codebase.}

### Components Affected
| Component | Change Type | Risk |
|-----------|------------|------|
| {file/module} | New / Modified / None | Low / Medium / High |

### Data Flow
{How data moves through the system for this feature}

## Decisions Log
| # | Decision | Alternatives Considered | Rationale | Substantiation |
|---|----------|------------------------|-----------|----------------|
| 1 | {decision} | {alt 1, alt 2} | {why chosen} | {source: file:line / doc URL / [NEEDS TESTING]} |

## Risks and Open Questions
{Inline throughout relevant sections, collected here as summary}
- {Risk 1} [severity: high/medium/low] - {mitigation}
- [NEEDS TESTING] {item} - {proposed test strategy}
- [USER DECISION NEEDED] {item} - {options}

## Phase 2 Requirements
{What the /plan-spec team needs to know}

### Recommended Specialist Agents for Phase 2
| Specialist | Why Needed | Key References |
|-----------|-----------|----------------|
| {e.g., LangGraph specialist} | {reason based on Phase 1 findings} | {docs/files to read} |

### Key Files to Read
| File | Why | Lines |
|------|-----|-------|
| {path} | {what it contains relevant to this feature} | {line range} |

### Substantiation Summary
- **Verified**: {count} decisions backed by docs/code
- **Pattern-based**: {count} decisions following established patterns
- **[NEEDS TESTING]**: {count} decisions requiring validation
```

### Scratch Directory

Artifacts are stored in `apps/plans/{feature-name}/`:
- `interview.md` - Refined feature spec from the Stage 1 expert interview (user's answers + context)
- `overview.md` - The final plan document (Stage 3 output)

Research artifacts in `apps/plans/{feature-name}/scratch/`:
- `backend-research.md` - Backend codebase researcher's findings
- `frontend-research.md` - Frontend codebase researcher's findings
- `database-research.md` - Database codebase researcher's findings
- `plans-research.md` - Historical plans researcher's findings
- `external-research.md` - External researcher's findings
- `review.md` - Review agent's critique
- `{specialist}-research.md` - Any additional specialist agent findings

Only the files relevant to the spawned agents are created (e.g., a backend-only feature won't have `frontend-research.md`).

These files persist after Phase 1 completes. Phase 2 agents can optionally reference them for deeper context beyond the plan document.

---

## Phase 2: `/plan-spec`

### Purpose
Read the Phase 1 overview and produce a file-level detailed specification. The spec tells the implementation team exactly which files to create, modify, what function signatures to use, what SQL migrations to write, and what tests to add.

### Input
- `apps/plans/{feature-name}/overview.md` (required - the Phase 1 output)
- `apps/plans/{feature-name}/scratch/` (optional - Phase 1 research artifacts)
- The user specifies which feature plan to spec (path to overview.md)

### Team Composition

The team composition is **guided by Phase 1's "Recommended Specialist Agents for Phase 2" section**. The team lead reads the overview and spawns the agents that Phase 1 identified as necessary.

**Minimum team (always present)**:
1. **Team Lead** (Architect-Orchestrator) - same role as Phase 1
2. **Spec Writer** - reads the overview, researches specific files, writes detailed specs per component
3. **Review Agent** - verifies specs against actual code, challenges feasibility

**Additional specialists** (spawned based on Phase 1 recommendations):
- LangGraph Specialist, Supabase Specialist, Frontend Specialist, etc.
- Each reads the overview + relevant scratch files + actual code
- Each produces component-level specs for their domain

### Spec Document Structure

Output at `apps/plans/{feature-name}/spec.md`:

```markdown
# {Feature Name} - Detailed Specification

## Overview Reference
Based on: `apps/plans/{feature-name}/overview.md`

## Files Changed

### New Files
| File | Purpose | Template/Pattern |
|------|---------|-----------------|
| {path} | {what it does} | {based on existing file pattern} |

### Modified Files
| File | Change | Lines Affected |
|------|--------|---------------|
| {path} | {what changes} | {current line range} |

## Component Specifications

### {Component 1}
**File**: `{path}`
**Change type**: New / Modified

#### Current State
{Current implementation summary with line references}

#### Proposed Changes
```{language}
{Code snippets showing key interfaces, signatures, types}
```

#### Dependencies
- Requires: {other components that must exist}
- Used by: {components that will consume this}

### {Component 2}
{... same structure ...}

## Database Migrations
```sql
-- Migration: {name}
{SQL for schema changes}
```

## Configuration Changes
{Changes to ContextSchema, company_config, company_table_features, etc.}

## Test Plan
| Test | Type | What It Verifies |
|------|------|-----------------|
| {test name} | Unit / Integration / Agent | {what it checks} |

## Implementation Order
{Numbered list of steps in dependency order}
1. {step 1} - creates {what}, enables {what}
2. {step 2} - depends on step 1
...

## Phase 3 Requirements
### Recommended Team for Implementation
| Agent | Why Needed | Files to Touch |
|-------|-----------|---------------|
| {specialist} | {reason} | {file list} |
```

---

## Phase 3: `/plan-implement`

### Purpose
Read the Phase 2 spec and implement the changes. This phase produces actual code changes.

**Note**: Phase 3 design is deferred. The spec document from Phase 2 provides enough structure for implementation. The team composition and workflow will be designed separately, informed by the patterns established in Phases 1 and 2.

### Key Constraints (established now)
- No git commit/push/destructive actions without explicit user approval
- Write permissions: plan files + new files only. Existing file modifications require the user's permission mode
- Implementation agents CAN modify existing files (unlike planning agents)
- Each agent follows the spec document strictly - deviations require escalation
- Phase 2's "Implementation Order" section defines the task sequence
- Phase 2's "Recommended Team for Implementation" section guides team composition

---

## Safety and Permissions

### Read-Only by Default (Phases 1 and 2)
Planning agents cannot modify existing source files. They can only:
- Read any file in the codebase
- Write to the plan file (`apps/plans/{feature-name}/`)
- Write to scratch directory (`apps/plans/{feature-name}/scratch/`)
- Run read-only bash commands (ls, git log, git diff, git blame)

### Forbidden Actions (All Phases)
- `git commit`, `git push`, `git checkout`, `git reset` - never
- `rm`, `rm -rf` on existing files - never
- Modifying `.env`, credentials, or secrets - never
- Running destructive database operations - never
- Force-pushing, amending commits - never

### Phase 3 Additional Permissions
- Can create new files anywhere in the project
- Can modify existing files (with user's configured permission mode)
- Can run tests (`uv run pytest`)
- Can run linting (`make lint`, `make format`)
- Cannot deploy or run production commands

---

## Substantiation Requirements

### Three-Tier System

**Tier 1 - Verified** (highest confidence):
- Backed by official documentation with URL
- Backed by working code in the NuevoDia codebase with file:line reference
- Backed by existing NuevoDia plan that documents a tested approach

**Tier 2 - Pattern-based** (good confidence):
- Follows an established general pattern (cite the pattern)
- Applied specifically to NuevoDia context
- Example: "Following the existing registry pattern (TableFeature/rowActions/index.ts:1-15), we add a new registry entry for {feature}"

**Tier 3 - [NEEDS TESTING]** (requires validation):
- Reasonable approach without direct precedent in docs or codebase
- Must include: (a) what needs testing, (b) proposed test strategy, (c) fallback if it doesn't work
- Example: "[NEEDS TESTING] Using runtime.context to pass custom tool configurations. Test: create a minimal agent with the pattern and verify tools are filtered correctly. Fallback: hardcode tool list in agent."

### Substantiation Rules
- Every architectural decision in the plan must have a Tier 1 or Tier 2 citation
- Tier 3 items are acceptable but must be explicitly flagged
- Review agents verify substantiation claims by spot-checking sources
- If an agent cannot find substantiation, it must say so explicitly rather than proceeding without evidence
- The plan's "Substantiation Summary" section tallies all three tiers

---

## Conflict Resolution Protocol

### Step 1: Initial Disagreement
When a review agent challenges a decision, both parties present their arguments to the team lead.

### Step 2: Steelman + Counter-argument
The team lead requires the challenger to first articulate the strongest argument FOR the challenged decision (steelman), then present the counter-argument. This ensures the challenge is substantive, not reflexive.

### Step 3: Team Lead Deliberation
The team lead evaluates both arguments against:
- Substantiation quality (which side has stronger evidence?)
- Complexity cost (which approach is simpler?)
- Codebase alignment (which approach matches existing patterns?)
- Risk profile (which approach has fewer unknowns?)

### Step 4: Resolution or Escalation
- If the team lead can resolve with clear reasoning: decision is made, documented in Decisions Log
- If either party still disagrees: they present one final argument
- If still unresolved: escalate to user via AskUserQuestion with full context

### Escalation Message Format
```
[ESCALATION] {topic}

Decision needed: {what the user must decide}

Option A: {description}
- Arguments for: {from specialist/researcher}
- Substantiation: {sources}

Option B: {description}
- Arguments for: {from reviewer}
- Substantiation: {sources}

Team lead assessment: {which option the team lead leans toward and why}
```

---

## Implementation as Claude Code Skills

Each phase is implemented as a Claude Code skill in `.claude/skills/`:

### `/plan-overview` Skill

```
.claude/skills/plan-overview/
  SKILL.md          # Skill definition with frontmatter
  references/       # Reference docs for the skill
```

**SKILL.md frontmatter**:
```yaml
---
name: plan-overview
description: >
  Spawn an agent team to analyze a feature request and produce an architecture overview.
  Use when: (1) planning a new feature, (2) designing a significant code change,
  (3) evaluating architectural decisions, (4) thinking about how to implement something,
  (5) asking how a feature should be built, (6) requesting a plan or architecture for
  any code change, (7) the user says /plan-overview.
  Stages: Expert interview with codebase-informed questions, parallel research by
  domain-specific agents, synthesis + adversarial review.
  Produces apps/plans/{feature}/overview.md.
args:
  - name: input
    description: Feature description (inline text) or path to a file containing the feature spec
---
```

**Input handling** (same pattern as `/improve-plan`):
- If input looks like a filepath (contains `/` or `\`, ends with `.md`/`.txt`, starts with `./` or `@`): read the file and use its contents as the feature description
- Otherwise: treat the input as an inline description of the feature
- If input is an existing overview.md: treat as a request to refine/rebuild the plan

**Invocation**: Both auto-triggered and explicit (`/plan-overview`). The description is broad to catch most planning-related requests. The user can tell Claude not to use it when they don't need a full planning session.

**Skill body**: Instructions for the team lead agent to:
1. Create the team via TeamCreate
2. Create the output directory (`apps/plans/{feature-name}/` and `scratch/`)
3. **Stage 1 - Interview**: Spawn 2-3 interview prep agents to analyze the feature from different perspectives. Each reads relevant code and generates questions. Team lead compiles best questions and interviews user via AskUserQuestion. Write refined spec to `interview.md`. Shut down interview agents.
4. **Stage 2 - Research**: Analyze the refined spec and determine which researchers to spawn. Create tasks and spawn researcher + reviewer agents via Task tool. Monitor progress, facilitate cross-domain queries.
5. **Stage 3 - Synthesis**: Synthesize all research into overview.md. Send to review agent. Incorporate valid challenges. Apply quality checklist.
6. Shut down all agents
7. Delete the team

### `/plan-spec` Skill

```
.claude/skills/plan-spec/
  SKILL.md
  references/
```

**SKILL.md frontmatter**:
```yaml
---
name: plan-spec
description: >
  Spawn an agent team to produce a file-level detailed specification from a plan overview.
  Use when: (1) an overview exists and user wants the detailed spec,
  (2) the user says /plan-spec. Requires a completed overview.md as input.
  Reads Phase 1 recommendations to determine which specialist agents to spawn.
args:
  - name: plan-path
    description: Path to the overview.md file (e.g., apps/plans/my-feature/overview.md)
---
```

### `/plan-implement` Skill (deferred)

To be designed after Phase 1 and Phase 2 are built and tested.

---

## Cost and Performance Expectations

### Token Usage (based on Anthropic data)
- Single agent: ~200K tokens per session
- 4-agent team: ~800K tokens per session (4x)
- Full planning cycle (Phase 1 + Phase 2): ~1.6M tokens estimated

### Time Estimates
- Phase 1 (`/plan-overview`): 5-15 minutes depending on feature complexity
- Phase 2 (`/plan-spec`): 5-15 minutes
- These are not fast commands - they are thorough planning sessions

### Cost Optimization
- Model inherits from parent (user controls cost via their model selection)
- Research agents can be assigned to Sonnet via the skill if cost is a concern (configurable, not default)
- Scratch files reduce duplicate research between phases
- Team lead as delegate-mode prevents it from doing redundant research

---

## Evidence and Sources

### Validated Patterns (Tier 1 sources)

| Pattern | Source | Credibility |
|---------|--------|-------------|
| Manager orchestrates, not executes | CrewAI failures (TDS), Anthropic guidance | HIGH |
| Narrow scope per agent | Osmani (Google), Lance Martin (LangChain) | HIGH |
| Fan-out research / fan-in synthesis | Google DeepMind scaling study + Anthropic C compiler | HIGHEST |
| Planning tasks degrade with multi-agent (-39 to -70%) | Google DeepMind PlanCraft benchmark (Dec 2025) | HIGHEST |
| Research tasks benefit from parallelism (+81%) | Google DeepMind Finance-Agent benchmark (Dec 2025) | HIGHEST |
| 2-3 teammates for most tasks, max 5 | Claude Code official docs (Feb 2026) | HIGHEST |
| Three-phase workflow | Thoughtworks, Augment Code, GitHub Spec-Kit | HIGH |
| Moderate disagreement > maximal | A-HMAD paper (Springer), DMAD (OpenReview) | HIGH |
| Context rot degrades performance | Lance Martin, Addy Osmani | HIGH |
| Centralized coordination > independent | Google DeepMind (80.9% improvement) | HIGHEST |
| >4 handoffs almost always fail | Microsoft Azure patterns | HIGH |
| 17x error amplification (independent) vs 4.4x (centralized) | Kim et al. via TDS | HIGH |
| Agent Teams coordination via TaskCreate/SendMessage | Claude Code official docs | HIGHEST |
| Skills auto-propagate to team teammates via progressive disclosure | Claude Code official docs | HIGHEST |
| Progressive disclosure: ~100 tokens per skill description, full body on-demand | Claude Code skills docs | HIGHEST |
| No per-agent skill assignment in teams (feature request #24316 open) | Claude Code docs + GitHub | HIGHEST |
| Boris Cherny: "You don't trust; you instrument" | Creator of Claude Code | HIGH |

### NuevoDia-Specific Patterns (from codebase research)

| Pattern | Location | Relevance |
|---------|----------|-----------|
| Runtime context: `runtime.context.{field}` (Pydantic) | `apps/backend/configuration.py` | Any new agent config |
| Agent node pattern: fresh agent per invocation | `apps/backend/agents/*/agent.py` | Adding new agents |
| Hub-and-spoke via Command(goto=Send()) | `apps/backend/agents/supervisor/graph.py` | Agent delegation |
| Message wrapper pattern (context control) | `apps/backend/agents/supervisor/graph.py:34-135` | Multi-agent message flow |
| 12-step agent addition process | Verified by backend reviewer | Adding agents to main graph |
| Registry pattern (sidebar, row actions, modals, columns) | `apps/frontend/src/registry/`, `TableFeature/` | Frontend extensibility |
| 4 nested providers: User > CompanyConfig > Thread > Stream | `apps/frontend/src/app/page.tsx` | Frontend architecture |
| Dual communication: LangGraph SDK + Supabase client | `apps/frontend/src/providers/` | Backend integration |
| Database-driven UI config | `company_config`, `company_table_features` | Multi-tenant features |
| Composite PK: company_assistants (company_id, graph_id) | `public.company_assistants` | Any assistant queries |
| axxel_hr has NO RLS (security gap) | Supabase schema | Security considerations |
| Historical plans in apps/plans/ | `apps/plans/*` | Past decisions context |

### Key Codebase Insights from Reviews

**Backend reviewer corrections**:
- Adding a new agent is 12 steps (not 9): includes wrapper function, retry_policy, destinations, initialize_state
- Deprecated `create_*_agent` factory functions are dead code (should be cleaned up)
- Three agent files reference `ContextSchema` without importing it (works only due to `from __future__ import annotations`)
- `Command.PARENT` is exception-based control flow (non-obvious)
- Deep research uses completely different framework (deepagents with middleware) vs main graph (create_react_agent)

**Frontend reviewer corrections**:
- Only 2 top-level registries (sidebar, quickEdit); 3 sub-registries inside TableFeature
- Missing from original report: AuthGuard, services layer, message rendering pipeline complexity, loading state management
- Code duplication in company slug verification across pages
- Login page uses inline styles (inconsistent with Tailwind)
- `ensureToolCallsHaveResponses` is a critical LangGraph SDK workaround

**Architecture reviewer key challenges**:
- Three separate teams means context loss - mitigated by self-contained plan docs + scratch files
- Team lead as architect + orchestrator has anchoring bias - mitigated by steelman rule
- Substantiation requirement could paralyze - mitigated by three-tier system
- Planning takes 5-15 minutes per phase - acceptable for thorough planning

---

## Agent Scaling Evidence

### The "4 Agent Threshold" - What It Actually Says

The commonly cited "gains saturate beyond 4 agents" is a **simplified distortion** of the Google DeepMind/MIT study ("Towards a Science of Scaling Agent Systems," arXiv 2512.08296, December 2025).

**What the paper actually found:**

| Finding | Detail | Implication |
|---------|--------|-------------|
| Budget-constrained ceiling | Under fixed 4,800-token budgets, per-agent reasoning becomes thin beyond 3-4 agents | NOT an absolute limit - with larger budgets (standard for Opus 4.6), the ceiling shifts |
| Task-type dependence | Parallelizable tasks: +81%. Sequential planning: -39% to -70% | The task type matters more than agent count |
| Capability saturation | Diminishing returns when single-agent baselines exceed ~45% accuracy | When the model is already good, extra agents help less |
| Error amplification | Independent: 17.2x. Centralized: 4.4x | Coordination topology matters more than agent count |
| Super-linear overhead | Communication turns follow power law: T = 2.72 x (n+0.5)^1.724 | Coordination cost accelerates with each additional agent |

**Models used**: GPT-5, Gemini 2.5, Claude Sonnet 4.5 - these are CURRENT models, making the study relevant to 2026.

### Why the C Compiler (16 Agents) Succeeded

Anthropic's 100,000-line C compiler project used 16 agents successfully, but the key was **task decomposability**:
- Each agent worked on independent subtasks (different source files, test cases)
- NO orchestration agent - agents used git-based task locking
- When agents hit the SAME bug, "16 agents running didn't help because each was stuck solving the same task"
- Success required restructuring work so agents operated on truly independent files

This validates the **fan-out for parallel independent work** pattern, not coordinated planning.

### The Critical Distinction for NuevoDia

| Activity | Multi-Agent Effect | Why |
|----------|-------------------|-----|
| **Research** (reading code, searching docs) | Positive (more coverage) | Tasks are independent and parallelizable |
| **Synthesis** (writing the plan) | Negative (-39 to -70%) | Sequential reasoning, interdependent decisions |
| **Review** (challenging the plan) | Positive (fresh perspective) | Independent evaluation of existing artifact |

This is why we use **fan-out/fan-in**: many researchers in parallel, single synthesizer, single reviewer.

### Source URLs
- Paper: https://arxiv.org/abs/2512.08296
- Google Blog: https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/
- C Compiler: https://www.anthropic.com/engineering/building-c-compiler
- Claude Code Teams docs: https://code.claude.com/docs/en/agent-teams
- Anthropic Multi-Agent Research: https://www.anthropic.com/engineering/multi-agent-research-system

---

## Open Items and Future Work

- [ ] **Build /plan-overview skill**: First skill to implement and test
- [ ] **Build /plan-spec skill**: After /plan-overview is proven
- [ ] **Design /plan-implement**: After Phases 1 and 2 are validated
- [ ] **Test with a real feature**: Use a medium-complexity feature (e.g., adding a new table column type) as the first test case
- [ ] **Validate fan-out/fan-in**: Test whether parallel research agents + single synthesizer produces better plans than fewer generalist agents
- [ ] **Refine escalation UX**: Test the AskUserQuestion-based escalation flow in practice
- [ ] **Cost tracking**: Monitor actual token usage across planning sessions to validate estimates
