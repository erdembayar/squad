# Squad — Design Document

> **What this document covers:** How Squad works, end‑to‑end. It is split into two halves:
> 1. **Abstract level** — the mental model. What the pieces are, how they relate.
> 2. **Implementation level** — the actual code paths, files, and runtime types.
>
> Throughout, we answer the central question: **What *is* a Squad agent?** (Spoiler: it’s a Copilot **subagent / session** driven by a markdown charter — not a separate Copilot install.)

---

## Part 1 — Abstract Level

### 1.1 The one‑sentence model

> A **Squad** is a folder of markdown files (`.squad/`) that defines a team of named specialists. A **Coordinator** (the GitHub Copilot agent loaded from `.github/agents/squad.agent.md`) reads those files at runtime and **spawns one Copilot subagent per specialist**, each booted with that specialist's charter as its system prompt.

There is **no separate process per agent**, no Copilot‑inside‑Copilot, no daemon. Agents are **prompted personalities** running on the host Copilot’s subagent / session machinery.

### 1.2 The three layers

```
┌──────────────────────────────────────────────────────────────────┐
│  Layer 3 — Team State (markdown in .squad/)                      │
│    team.md, routing.md, decisions.md, agents/<name>/charter.md   │
│    ↑ Human-readable, git-committed, persists across sessions     │
├──────────────────────────────────────────────────────────────────┤
│  Layer 2 — Coordinator (the Squad agent)                         │
│    A single Copilot agent definition (squad.agent.md)            │
│    Routes work, picks specialists, spawns subagents,             │
│    enforces handoffs and reviewer gates                          │
├──────────────────────────────────────────────────────────────────┤
│  Layer 1 — Host platform (GitHub Copilot)                        │
│    Provides:  task / runSubagent tools, models,                  │
│               sessions, tool execution, the LLM itself           │
└──────────────────────────────────────────────────────────────────┘
```

The user only ever talks to **Layer 2** (the Coordinator). The Coordinator is responsible for translating one user request into many parallel subagent invocations on Layer 1, using context from Layer 3.

### 1.3 What an "agent" actually is

| Concept                | What it really is                                                                                              |
|------------------------|-----------------------------------------------------------------------------------------------------------------|
| **Coordinator (Squad)**| A single Copilot agent definition (`squad.agent.md`) installed via `squad init`.                                |
| **Specialist agent**   | A *charter* (`.squad/agents/<name>/charter.md`) + *history* (`history.md`). Not a process. Not a Copilot install.|
| **Spawn**              | One call to `task` (CLI) or `runSubagent` (VS Code), passing the charter as the prompt. A fresh Copilot session.|
| **Subagent**           | The running session that resulted from a spawn. Lives only for the duration of the task. Its only memory is what it writes back to its `history.md`. |
| **@copilot**           | The official **GitHub Copilot Coding Agent** — a real, separate executor. Squad treats it as one team member it can route issues to. This is the *only* agent that is not a Copilot subagent. |
| **Ralph**              | A long‑running watcher (a Node process started by `squad watch`) that polls GitHub Issues and asks the Coordinator to dispatch work. Not an LLM session. |
| **Scribe**             | A specialist agent like any other, but invoked silently after every batch to merge `decisions/inbox/*` and write `orchestration-log/`. |

> **Mental rule:** *Every named team member except `@copilot` and `Ralph` is a markdown charter that becomes a Copilot subagent at spawn time.*

### 1.4 The lifecycle of a single user message

```
You: "Build the login page"

  ┌─ Coordinator receives message ──────────────────────────────┐
  │ 1. Capture directives (if any) → decisions/inbox/           │
  │ 2. Match message against routing.md                         │
  │    → Frontend, Backend, Tester, Lead, Scribe                │
  │ 3. Pick a model for each (per-spawn or session model)       │
  │ 4. Show "launch table" to user (CLI only)                   │
  │ 5. SPAWN all selected agents IN PARALLEL                    │
  │      task(name="frontend", prompt=charter+task, …)          │
  │      task(name="backend",  prompt=charter+task, …)          │
  │      task(name="tester",   prompt=charter+task, …)          │
  │      task(name="lead",     prompt=charter+task, …)          │
  └─────────────────────────────────────────────────────────────┘
                           │
                           ▼
  Each subagent runs in its own Copilot session, with its own
  context window, its own tool calls, its own model.
                           │
                           ▼
  ┌─ Coordinator collects results ──────────────────────────────┐
  │ 6. read_agent on each (CLI) or auto-return (VS Code)        │
  │ 7. Synthesize a reply for the user                          │
  │ 8. Spawn Scribe LAST with a "spawn manifest" so it can       │
  │    write orchestration-log/ + merge decisions inbox          │
  └─────────────────────────────────────────────────────────────┘
                           │
                           ▼
  Reply to user. .squad/ is updated. Knowledge persists in git.
```

### 1.5 Why this design

* **Memory in markdown, not RAM.** Every agent’s knowledge lives in `agents/<name>/history.md`. A clone of the repo gets the whole team, with all their accumulated context.
* **Parallelism is the whole point.** The host Copilot can run many subagents at once; the Coordinator’s job is to *maximize* that fan‑out per turn.
* **One human in the loop.** The Coordinator may not generate domain artifacts itself — it must spawn an agent. That keeps work attributable and reviewable.
* **Append‑only state.** `decisions.md`, `history.md`, `log/`, `orchestration-log/` are merged with `merge=union` (see `.gitattributes`), so concurrent branches/worktrees don’t conflict.

---

## Part 2 — Implementation Level

### 2.1 Repository shape

```
squad/
├─ packages/
│  ├─ squad-sdk/        ← runtime: coordinator, lifecycle, client, ralph, …
│  └─ squad-cli/        ← CLI wrapper: `squad init`, `squad watch`, …
├─ .squad-templates/    ← canonical templates (mirrored to 3 publish targets)
├─ .squad/              ← this repo's own team state (Squad uses Squad)
│  ├─ team.md
│  ├─ routing.md
│  ├─ decisions.md
│  ├─ agents/<name>/charter.md + history.md
│  ├─ orchestration-log/
│  └─ log/
└─ .github/agents/
   └─ squad.agent.md    ← the Coordinator definition (this is what Copilot loads)
```

### 2.2 The two npm packages

| Package                           | Purpose                                                                                              |
|-----------------------------------|------------------------------------------------------------------------------------------------------|
| `@bradygaster/squad-sdk`          | Programmable runtime. `SquadCoordinator`, `AgentLifecycleManager`, `SquadClientWithPool`, `RalphMonitor`, routing, charter compilation, model resolution, hooks. |
| `@bradygaster/squad-cli`          | Thin command‑line wrapper around the SDK. Implements `squad init`, `squad upgrade`, `squad watch`, etc. Depends on the SDK. |

The CLI is the user‑facing front door. The SDK is what runs inside the Copilot agent at runtime *and* what Ralph uses when it dispatches work in `--execute` mode.

### 2.3 How a Copilot session reaches Squad

```
copilot --agent squad --yolo
        │
        ├─ Copilot loads .github/agents/squad.agent.md
        │     (this file IS the Coordinator's system prompt)
        │
        ├─ User sends a message
        │
        ├─ Coordinator reasons through routing rules
        │  (driven by what's in `.squad/team.md`, `routing.md`, `decisions.md`)
        │
        └─ Coordinator calls the host's `task` tool (or `runSubagent` in VS Code)
              ├─ task(name="fido",    agent_type="general-purpose", prompt=…)
              ├─ task(name="eecom",   agent_type="general-purpose", prompt=…)
              └─ task(name="scribe",  agent_type="general-purpose", prompt=…)
```

The Coordinator does not call the SDK directly. The SDK exists for two cases:

1. **Programmable embedding** — a TypeScript app drives Squad through `SquadCoordinator` / `SquadClientWithPool` (e.g., custom CI integrations, Aspire dashboards, Ralph).
2. **Ralph’s execute path** — when `squad watch --execute` decides to run an issue, it spawns a Copilot CLI process and pipes a prompt file into it. Inside that process, `squad.agent.md` is the loaded agent and the loop above runs.

### 2.4 Code map (key SDK modules)

| Module                                | Role                                                                                                  |
|---------------------------------------|-------------------------------------------------------------------------------------------------------|
| `coordinator/coordinator.ts`          | `SquadCoordinator.handleMessage()` — direct response → routing → strategy → fan‑out → emit events.    |
| `coordinator/direct-response.ts`      | The "no spawn needed" fast path (status checks, factual answers).                                      |
| `coordinator/fan-out.ts`              | `spawnParallel()` — `Promise.allSettled` over agent spawns; isolates failures.                         |
| `agents/lifecycle.ts`                 | `AgentLifecycleManager.spawnAgent()` — read charter → compile → resolve model → create session → send.|
| `agents/charter-compiler.ts`          | Parses `charter.md` into a typed `CustomAgentConfig` (identity, expertise, model, tools).             |
| `agents/model-selector.ts`            | 4‑layer model resolution: user override → charter → task type → role default.                         |
| `client/index.ts` (`SquadClientWithPool`) | Wraps `@github/copilot-sdk` `CopilotClient`; adds a `SessionPool` and an `EventBus`.              |
| `adapter/client.ts`                   | `CopilotSessionAdapter` — maps Squad's session API onto the official Copilot SDK session.             |
| `runtime/event-bus.ts`                | Pub/sub for `coordinator:routing`, `session:created`, `agent:milestone`, etc. Drives Ralph + telemetry.|
| `ralph/triage.ts`                     | Parses `team.md` + `routing.md` to decide which team member should own a GitHub Issue.                 |
| `ralph/index.ts`                      | `RalphMonitor` — long‑lived watcher that subscribes to the EventBus, runs health checks.              |
| `config/routing.ts`                   | Compiles `routing.md` tables into `CompiledRouter`; `matchRoute(message, router)`.                    |

### 2.5 Anatomy of `spawnAgent` (the moneyshot)

From `packages/squad-sdk/src/agents/lifecycle.ts`:

```
spawnAgent({ agentName, task, taskType, modelOverride, … })
  │
  ├─ 1. Read charter.md         ← .squad/agents/<name>/charter.md
  ├─ 2. compileCharter(...)     ← turns markdown into CustomAgentConfig + system prompt
  ├─ 3. resolveModel(...)       ← user override → charter pref → task-type → role default
  ├─ 4. client.createSession({  ← SquadClientWithPool → @github/copilot-sdk CopilotClient
  │      model: resolvedModel.model,
  │      systemMessage: { content: agentConfig.prompt }   ← charter becomes the system prompt
  │    })
  ├─ 5. wrap session in AgentHandle (lifecycle status, idle timer, history persistence)
  └─ 6. handle.sendMessage(task) ← the actual user task is the first user-message in that session
```

Key insight: **the specialist's charter is injected as the *system message* of a fresh Copilot session**. That session is the agent. When it ends, only what it wrote back to disk (commits, history.md, decisions inbox) survives.

### 2.6 Coordinator dispatch (`SquadCoordinator.handleMessage`)

```
handleMessage(message, context):
  1. directHandler.shouldHandleDirectly(message)?    → return DirectResult
  2. matchRoute(message, compiledRouter)             → { agents: [...], confidence }
  3. determineStrategy(routing)                      → 'direct' | 'single' | 'multi' | 'fallback'
  4. buildSpawnConfigs(routing, message, context)
  5. spawnParallel(configs, fanOutDeps)              → Promise.allSettled
  6. emit 'coordinator:routing' events along the way (Ralph + OTel listen)
  7. return CoordinatorResult { strategy, routing, spawnResults, durationMs }
```

`spawnParallel` is intentionally *fire‑and‑isolate*: one bad spawn doesn’t take down the others.

### 2.7 Two surfaces, one design — CLI vs. VS Code

The Coordinator detects which Copilot surface it’s on by looking at available tools:

| Surface  | Spawn tool      | Parallelism trick                          | Model selection         |
|----------|-----------------|--------------------------------------------|-------------------------|
| CLI      | `task`          | `mode: "background"` + `read_agent`        | Per-spawn (4‑layer)     |
| VS Code  | `runSubagent`   | Multiple `runSubagent` calls in one turn   | Session model only      |
| Fallback | (none)          | Run inline                                 | n/a                     |

This is documented in `squad.agent.md` under **Client Compatibility**. The same charters and the same routing rules work on both surfaces; only the spawning syntax differs.

#### 2.7.1 Practical guidance regarding the number of subagents: Is it possible to spawn up to 10 subagents?

There are three layers between you and 10 simultaneous subagents:

┌───────────────┬─────────────────┬───────────────────────────────────────────────────────────┬──────────────────────────────────────┐
│ Layer         │ Default         │ Where                                                     │ What happens at 10                   │
├───────────────┼─────────────────┼───────────────────────────────────────────────────────────┼──────────────────────────────────────┤
│ Coordinator   │ maxAgents: 5    │ packages/squad-sdk/src/coordinator/response-tiers.ts:69   │ The Coordinator's heuristic stops at │
│ response-tier │ (Full mode)     │                                                           │ ~5. You'd need to ask explicitly:    │
│               │                 │                                                           │ "spawn 10 agents in parallel" to     │
│               │                 │                                                           │ override.                            │
├───────────────┼─────────────────┼───────────────────────────────────────────────────────────┼──────────────────────────────────────┤
│ SDK           │ maxConcurrent:  │ packages/squad-sdk/src/client/session-pool.ts:32          │ 10 fits exactly. The 11th throws     │
│ SessionPool   │ 10              │                                                           │ SessionPool at capacity (10). Bump   │
│               │                 │                                                           │ it: new SquadClientWithPool({ pool:  │
│               │                 │                                                           │ { maxConcurrent: 20 } }).            │
├───────────────┼─────────────────┼───────────────────────────────────────────────────────────┼──────────────────────────────────────┤
│ Host platform │ No documented   │ task (CLI) / runSubagent (VS Code)                        │ Real bottleneck → API rate limits,   │
│ (Copilot)     │ hard cap        │                                                           │ your model quota, machine RAM.       │
└───────────────┴─────────────────┴───────────────────────────────────────────────────────────┴──────────────────────────────────────┘

 1. CLI: Spawn all 10 in one turn with mode: "background", then collect via read_agent. The host runs them concurrently.
 2. VS Code: Issue 10 runSubagent calls in the same turn — they fan out automatically (no mode parameter exists there).
 3. You'll likely hit rate limits before 10 cleanly finish. Mix model tiers (Haiku for cheap ones, Sonnet for the heavy ones) to 
spread load — see Per-Agent Model Selection in squad.agent.md.
 4. If you're driving Squad programmatically (via the SDK, not Copilot), bump the pool: pool: { maxConcurrent: 15 } gives headroom 
over 10 to avoid edge-case throws.
 5. Avoid spawning 10 agents on the same files. Append-only files (history.md per agent, decisions/inbox/<agent>-*.md) are 
conflict-free; shared source files are not. Use worktree mode (SQUAD_WORKTREES=1) if multiple agents will edit the same code on the 
same issue.

So: can you? Yes — request it explicitly, raise the pool if you're using the SDK directly, and expect the upstream Copilot/model rate
limits to be the real ceiling.

### 2.8 What lives in `.squad/`

```
.squad/
├─ team.md              ← roster + capability profile + project context
├─ routing.md           ← Work Type → Agent + Module Ownership tables (parsed by triage.ts)
├─ decisions.md         ← merged team decisions (Scribe is the only writer)
├─ decisions/inbox/     ← agents drop {agent}-{slug}.md here; Scribe merges
├─ ceremonies.md        ← named multi-agent flows (sprint planning, retros, …)
├─ agents/<name>/
│   ├─ charter.md       ← identity / expertise / boundaries / model preference / voice
│   └─ history.md       ← what THIS agent learned about THIS project (append-only)
├─ casting/             ← persistent name registry (so cast names survive across sessions)
├─ orchestration-log/   ← Scribe writes one .md per agent per batch
├─ skills/              ← compressed reusable patterns (with confidence levels)
├─ identity/now.md      ← the team's current focus
└─ log/                 ← session transcripts (searchable)
```

**Append‑only files** (`decisions.md`, `history.md`, `log/`, `orchestration-log/`) declare `merge=union` in `.gitattributes` so concurrent branches/worktrees merge cleanly without manual conflict resolution.

### 2.9 Worktree mode (parallel issue work)

When `worktrees: true` (or `SQUAD_WORKTREES=1`), the Coordinator creates one git worktree per issue at `{repo-parent}/{repo-name}-{issue-number}`, junctions/symlinks `node_modules`, and spawns agents *inside that worktree*. Each worktree has its own branch‑local `.squad/` state; merges flow back to `dev`/`main` via normal PRs. Append‑only + `merge=union` makes this safe.

### 2.10 Ralph — the one piece that *isn’t* an LLM

`squad watch [--execute]` starts a Node process that:

```
loop every --interval minutes:
  1. gh issue list ...                       ← scan for triage-eligible issues
  2. parse team.md + routing.md              ← ralph/triage.ts
  3. for each issue: pick a TeamMember       ← module-ownership > routing-rule > role-keyword > lead-fallback
  4. write a context snapshot to a temp file
  5. (if --execute) gh copilot -p <file>     ← spawn a Copilot CLI; the Squad Coordinator inside picks the work
  6. monitor: subscribe to EventBus events emitted by the dispatched session
  7. write health, errors, observations; back off on failures (4-tier escalation)
```

Ralph is implemented in `packages/squad-sdk/src/ralph/` (deterministic Node code) and `packages/squad-cli/src/cli/` (the `watch` subcommand). It is the only persistent process in Squad.

---

## Part 3 — Workflows

### 3.1 First-time setup (`squad init`)

```
User                                CLI                          Coordinator (Copilot)
 │                                   │                                  │
 │  squad init                       │                                  │
 ├──────────────────────────────────►│                                  │
 │                                   │ scaffold .squad-templates/       │
 │                                   │ install .github/agents/squad.agent.md
 │                                   │ create empty team.md             │
 │                                   │                                  │
 │  copilot --agent squad --yolo     │                                  │
 ├──────────────────────────────────────────────────────────────────────►│
 │  "I'm building a recipe app …"    │                                  │
 ├──────────────────────────────────────────────────────────────────────►│
 │                                   │                                  │ Init Mode Phase 1:
 │                                   │                                  │ • casting picks names
 │                                   │                                  │ • proposes 4–5 + Scribe
 │ ◄────────────────────────────────────────────────────────────────────┤ ask_user "Look right?"
 │  "yes"                            │                                  │
 ├──────────────────────────────────────────────────────────────────────►│ Init Mode Phase 2:
 │                                   │                                  │ writes .squad/agents/<name>/charter.md, history.md
 │                                   │                                  │ writes team.md, routing.md, decisions.md, casting/
 │ ◄────────────────────────────────────────────────────────────────────┤ "Team ready."
```

Phase 1 NEVER writes files. Phase 2 only runs after explicit user confirmation.

### 3.2 Normal work request (multi-agent fan-out)

```
User: "Team, build the login page"
                │
                ▼
┌──────────── Coordinator (squad.agent.md) ─────────────┐
│  1. Acknowledge in plain text (REQUIRED — no blank screen)
│  2. Show launch table (CLI only)
│  3. PARALLEL spawns in a single turn:
│       task name=lead     prompt=<charter+task+TEAM_ROOT+CURRENT_DATETIME>
│       task name=frontend prompt=<charter+task+...>
│       task name=backend  prompt=<charter+task+...>
│       task name=tester   prompt=<charter+task+...>
└────────────────────────────────────────────────────────┘
        │           │           │           │
        ▼           ▼           ▼           ▼
   ┌────────┐ ┌──────────┐ ┌─────────┐ ┌────────┐
   │ Lead   │ │ Frontend │ │ Backend │ │ Tester │   ← each = a fresh Copilot subagent session
   │ session│ │ session  │ │ session │ │ session│      with its charter as system prompt
   └────────┘ └──────────┘ └─────────┘ └────────┘
        │           │           │           │
        └───────────┴────┬──────┴───────────┘
                         ▼
              Coordinator collects results
              (read_agent on CLI; auto-return on VS Code)
                         │
                         ▼
            Spawn Scribe LAST with the spawn manifest
                         │
                         ▼
   Scribe writes orchestration-log/<ts>-<agent>.md
   Scribe merges decisions/inbox/* → decisions.md
                         │
                         ▼
            Coordinator replies to the user
```

### 3.3 Reviewer rejection / lockout

When an agent reviews another agent’s work and **rejects** it, the Coordinator must route the revision to a **different** agent (not the original author). This is enforced at routing time and recorded in `decisions.md`. Charters declare it: *"On rejection, I may require a different agent to revise."*

### 3.4 Ralph automated triage (`squad watch --execute`)

```
Ralph (Node process)                  Copilot CLI (spawned)              Subagents
        │                                   │                                │
  poll  │                                   │                                │
  ──►   │                                   │                                │
  gh    │ select issues                     │                                │
  api   │                                   │                                │
        │ build prompt file (issues + decisions + success criteria)          │
        │                                   │                                │
        │ exec: gh copilot -p file --agent squad --yolo                      │
        │ ─────────────────────────────────►│                                │
        │                                   │ load squad.agent.md            │
        │                                   │ pick an issue                  │
        │                                   │ spawn specialists ─────────────►│
        │                                   │                          (work happens)
        │ ◄──── EventBus events ────────────│ ◄──────────────────────────────│
        │ update .squad/triage state        │                                │
        │ on failure: 4-tier escalation     │                                │
        │ (reset CB → reauth → git pull →   │                                │
        │  pause 30m)                       │                                │
        │                                   │                                │
  next  │                                   │                                │
  poll  │                                   │                                │
```

### 3.5 The @copilot exception

`@copilot` (GitHub Copilot Coding Agent) is the **only** team member that is *not* a subagent of the local Coordinator. When the Coordinator routes work to `@copilot`, it does so by **assigning a GitHub issue** to the `@copilot` user. GitHub’s coding-agent infrastructure then opens its own session on its own runner and produces a PR. Squad’s `team.md` declares its capability profile (🟢 / 🟡 / 🔴) so the Coordinator can decide whether an issue is a good fit.

---

## Part 3.6 — State hygiene & history compaction (`squad nap`)

Squad's whole memory model relies on **append-only files** in `.squad/` (history.md, decisions.md, log/, orchestration-log/). That's great for git merges and worktree safety — but it grows unboundedly, which means: bigger `history.md` → bigger system prompt for the next spawn → more tokens, slower starts, eventual context blowout.

The fix is **`squad nap`** — a deterministic, file-only hygiene pass. **There is no LLM involved.** It's pure markdown surgery, implemented in `packages/squad-cli/src/cli/core/nap.ts` (~560 lines, no SDK calls, no network).

Quick recap of what it covers:

 - No LLM involved — nap is pure deterministic markdown surgery (packages/squad-cli/src/cli/core/nap.ts).
 - Pipeline: journal-on → compress histories → prune logs → drain inbox → archive decisions → journal-off.
 - history.md compaction: when > 15 KB, keep the last 5 (or 3 with --deep) ##  learning entries inline; move the rest verbatim to 
<name>/history-archive.md. Nothing is deleted — older entries are one file over.
 - decisions.md archival: age-based (>30 days) first, count-based fallback if still over 20 KB; undated entries are never touched. 
Documented invariant: entries_before === entries_kept + entries_archived.
 - Crash safety: .squad/.nap-journal sentinel; every step is idempotent and threshold-gated, so re-running is safe.
 - Manual trigger today (no auto-nap scheduler). Ralph's watch loop has a separate cleanup pass for its own scratch/log files — it 
doesn't touch history.md.
 - What nap deliberately doesn't do: no LLM summarization, no charter/team.md edits, no commits.

Also clarified the contract: growth happens during work; compaction happens between sessions, by command, deterministically, with
archives.

### What `nap` does, in order

```
squad nap [--deep] [--dry-run]
   │
   ├─ 1. Journal-on  (.squad/.nap-journal — survives crashes)
   ├─ 2. Snapshot "before" metrics (totalBytes, historyBytes, …)
   │
   ├─ 3. compressHistory()    ← per agent: .squad/agents/<name>/history.md
   ├─ 4. pruneLogs()          ← .squad/orchestration-log/  + .squad/log/
   ├─ 5. cleanInbox()         ← .squad/decisions/inbox/* → decisions.md
   ├─ 6. archiveDecisions()   ← decisions.md → decisions-archive.md
   │
   ├─ 7. Snapshot "after" metrics
   └─ 8. Journal-off, return NapResult { before, after, actions[] }
```

Each step is independent — failure of one doesn't stop the others.

### Step 3 — How `history.md` is compacted

`history.md` files are structured as `## Core Context` (kept forever) followed by a stream of `## …` learning entries. The compactor:

```
if size(history.md) <= 15 KB                           → skip (HISTORY_THRESHOLD)
parse all `## ...` headings into sections
keep the LAST N sections inline                        (N = 5 normal, 3 with --deep)
move the rest verbatim to `<name>/history-archive.md`  (append-only)
rewrite history.md = header + Core Context + last N sections
```

So the agent's working memory stays small, but **nothing is ever destroyed** — older learnings are just one file over in `history-archive.md`. The relevant constants live at the top of `nap.ts`:

```
HISTORY_THRESHOLD     = 15 KB         // don't bother under this
DECISION_THRESHOLD    = 20 KB
LOG_MAX_AGE_DAYS      = 7             // log/ + orchestration-log/ pruning
DECISION_MAX_AGE_DAYS = 30
KEEP_ENTRIES_DEFAULT  = 5             // last N ## sections kept inline
KEEP_ENTRIES_DEEP     = 3             // --deep keeps fewer
```

### Step 6 — How `decisions.md` is archived

`decisions.md` entries are dated `### YYYY-MM-DD: Topic`. The strategy is:

1. **Age‑based first.** Anything older than 30 days → `decisions-archive.md`.
2. **Count‑based fallback.** If nothing is old enough but the file is still over 20 KB, archive the **oldest dated** entries until what remains fits the budget.
3. **Undated entries are never touched** — they're treated as foundational directives.

A documented invariant guarantees no data loss: `entries_before === entries_kept + entries_archived`.

### Step 5 — Inbox cleanup overlap with Scribe

Scribe normally drains `decisions/inbox/` after every batch (see §3.2). `nap`'s `cleanInbox()` is the **belt‑and‑braces** version: if Scribe ever missed a file (crash, interrupted session, manual drop), `nap` appends it to `decisions.md` and deletes the inbox file. Same merge logic, idempotent.

### Crash safety: the journal file

```
.squad/.nap-journal     ← created at start, removed at end
```

If `nap` is interrupted, the journal stays on disk. The next run sees it, prints a warning ("Previous nap was interrupted. Continuing anyway."), and proceeds — every step is idempotent and threshold‑gated, so re-running is safe.

### Two tiers: normal vs `--deep`

| Mode               | What changes                                                                  |
|--------------------|-------------------------------------------------------------------------------|
| `squad nap`        | Keep last 5 `## ` entries per `history.md`. Standard prune/archive thresholds.|
| `squad nap --deep` | Keep last 3 entries per `history.md`. More aggressive history compression.    |
| `squad nap --dry-run` | Compute `NapResult` (before/after/actions) without touching disk.          |

There are also **two callable surfaces** of the same engine:

* `runNap(options)` — async entry, used by the `squad nap` CLI command.
* `runNapSync(options)` — synchronous twin, used inside the interactive shell where `executeCommand` is sync.

Both share the same threshold constants and produce the same `NapResult` shape.

### When `nap` runs

`nap` is **manual** today — there is no scheduler that auto-naps. Typical triggers:

1. The user sees prompts getting slow / `history.md` files large → runs `squad nap`.
2. CI / a pre-commit hook can shell out to `squad nap --dry-run` to detect bloat.
3. Ralph's watch loop has its own *cleanup* pass (`packages/squad-cli/src/cli/commands/watch/capabilities/cleanup.ts`) for *its own* scratch dirs and log files — that is **separate** from `nap` (it doesn't touch `history.md`).

### What `nap` deliberately does NOT do

* It does not call an LLM. No summarization, no semantic compression. Compaction = pure structural rewrite.
* It does not delete charters, team.md, routing.md, ceremonies.md, casting state, or skills.
* It does not commit. The user (or CI) decides when to commit the resulting diff.
* It does not touch `agents/*/charter.md` — only `history.md`.

### How compaction interacts with the agent lifecycle

```
Spawn time:
  charter-compiler reads charter.md  ← stable, never compacted
  (optionally seeds agent context with relevant slice of history.md)
                                      ↑
                                      this file IS what nap compacts

Work time:
  agent appends to history.md (own folder, no contention)
  agent drops decision files into decisions/inbox/
  Scribe drains inbox into decisions.md
  Scribe writes orchestration-log/<ts>-<agent>.md

Maintenance time (manual):
  squad nap → compress history.md, prune logs/, archive decisions.md
```

The contract is: **growth happens during work; compaction happens between sessions, by command, deterministically, with archives**.

---

## Part 4 — Quick reference

### 4.1 "How is each agent created?" — final answer

| Member type        | Created how?                                                                                  |
|--------------------|-----------------------------------------------------------------------------------------------|
| Coordinator (Squad)| Loaded once, by Copilot, from `.github/agents/squad.agent.md`. One Copilot agent definition. |
| Specialists        | **Copilot subagents.** One per `task` / `runSubagent` call. Charter = system message. Live for one task, then the session ends. |
| Scribe             | Same as specialists — but always spawned last in a batch with a spawn manifest.               |
| Ralph              | A Node process started by `squad watch`. Not an LLM. Drives the Coordinator from the outside.|
| @copilot           | The official GitHub Copilot Coding Agent. Squad routes to it by assigning a GitHub issue.     |

### 4.2 "Where does an agent's memory live?"

* **In‑session memory:** the Copilot context window of that subagent. Discarded on session end.
* **Cross‑session memory:** `.squad/agents/<name>/history.md`, written by the agent at end of work. Committed to git. Read back as part of the charter on next spawn.
* **Team‑shared memory:** `.squad/decisions.md` (merged by Scribe from `decisions/inbox/`), `.squad/skills/`, `.squad/identity/now.md`.

### 4.3 "Can two agents run at once?"

Yes — that is the default. The Coordinator’s mindset, in its own prompt, is literally *"What can I launch RIGHT NOW?"*. Parallel fan‑out uses `Promise.allSettled` (`spawnParallel` in `coordinator/fan-out.ts`); on the CLI surface it uses `task` + `mode: "background"`; on VS Code it spawns multiple `runSubagent` calls in one turn.

### 4.4 "Is Squad an agent framework like LangChain?"

Not quite. Squad is **a thin orchestration layer + a markdown convention** that runs *on top of* GitHub Copilot. It does not host an LLM, it does not implement tool calling, it does not implement subagents — those are all provided by Copilot. Squad supplies (a) the Coordinator prompt that knows how to use those facilities and (b) the SDK / CLI / file conventions that make a *team* of charters reproducible across machines and sessions.
