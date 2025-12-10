# Agency OS: File-Based Multi-Agent Orchestration

**NOTICE TO AGENTS:** This directory is your physical workspace. All state is managed via the file system. Agents communicate only through files.

---

## System Overview

**Architecture:** Single Supervisor spawns subagents on demand. One terminal runs everything.

**Two-Phase Execution:** Planning Phase (create ALL tasks) → Execution Phase (process sequentially)

```
                                    ┌─────────┐
                                    │  Human  │
                                    └────┬────┘
                                         │
                         ./dispatch OR conversational request
                                         │
                                         ▼
                              ┌─────────────────────┐
                              │   PLANNING PHASE    │
                              │ (decompose → tasks) │
                              └──────────┬──────────┘
                                         │
                         Creates ALL tasks in inbox/
                                         │
                                         ▼
                                 ┌───────────────┐
                                 │    inbox/     │
                                 │ (full queue)  │
                                 └───────┬───────┘
                                         │
                                         ▼
                        ┌────────────────────────────────┐
                        │   EXECUTION PHASE (SUPERVISOR) │
                        │  (process tasks sequentially)  │
                        └──┬─────┬─────┬─────┬─────┬─────┘
                           │     │     │     │     │
     ┌─────────────────────┼─────┼─────┼─────┼─────┼─────────────────────┐
     │                     │     │     │     │     │                     │
     ▼                     ▼     ▼     ▼     ▼     ▼                     ▼
┌───────────┐       ┌──────────┐│┌──────┐┌──────────┐┌───────────┐ ┌───────────┐
│MAP_REQUEST│       │ DEV_TASK ││READY  ││QA_REQUEST││E2E_RUN_REQ│ │BUG_REPORT │
└─────┬─────┘       │BUG_FIX_* ││FOR_QA │└────┬─────┘└─────┬─────┘ └─────┬─────┘
      │             └────┬─────┘└──┬────┘     │            │             │
      │                  │         │          │            │             │
      ▼                  ▼         │          ▼            ▼             │
┌───────────┐      ┌───────────┐  │    ┌───────────┐ ┌───────────┐       │
│CARTOGRAPH-│      │ DEVELOPER │  │    │  TESTER   │ │REGRESSION │       │
│    ER     │      │ (subagent)│  │    │ (subagent)│ │ (subagent)│       │
│ (subagent)│      └─────┬─────┘  │    └─────┬─────┘ └─────┬─────┘       │
└─────┬─────┘            │        │          │             │             │
      │                  ▼        │          ▼             ▼             │
      │            READY_FOR_QA ──┘     QA_REPORT     E2E_REPORT         │
      │             (inbox/)            (reports/)    (reports/)         │
      │                                     │             │              │
      ▼                    ┌────────────────┴─────┐       │              │
 SPEC_UPDATE               ▼                      ▼       ▼              │
 BUG_REPORT              PASS                   FAIL   BUG_FIX_*         │
 (reports/)                │                      │   (if code bugs)    │
      │                    ▼                      ▼       │              │
      │             archive/reports/      BUG_FIX_v[n+1]◀─┴──────────────┘
      │                                   or stasis/ (v3+)
      ▼
archive/reports/
BUG_FIX_REQUEST_v1 (if bugs found)
```

**Flow:**
1. Human dispatches task or makes conversational request
2. **PLANNING PHASE:** Supervisor decomposes request into ALL tasks, creates files in `inbox/`
3. **EXECUTION PHASE:** Supervisor picks up oldest task, spawns appropriate subagent
4. Subagent executes, creates output, returns control
5. Supervisor routes results (discovered work → append to queue)
6. Loop until all queues empty

---

## Processing Model: Sequential Only

**CRITICAL:** The system processes ONE task at a time. No parallel agent execution.

**Why:**
| Agent | Constraint |
|-------|------------|
| **Developer** | Shared codebase - concurrent edits conflict |
| **Tester** | Single browser instance |
| **Cartographer** | Reads codebase state - conflicts with Developer writes |

**Supervisor Loop:**
```
while inbox/ or processing/ or reports/ has files:
  pick ONE task (priority: processing/ → reports/ → inbox/)
  spawn agent
  wait for completion
  route result
  repeat
```

**No exceptions.** Even if multiple tasks are queued, process them one at a time.

---

## Directory Structure

| Directory | Purpose | Access Rules |
|:---|:---|:---|
| **`inbox/`** | Input triggers. Tasks waiting to be picked up. | **Read:** All. **Write:** Supervisor & Human. |
| **`processing/`** | Active workspace. Tasks being executed. | **Write:** Working agent only. |
| **`reports/`** | Output zone. Results from QA/Cartographer. | **Write:** Tester/Cartographer/Regression. |
| **`archive/inputs/`** | Completed input files (tasks). | **Write:** All agents after task completion. |
| **`archive/reports/`** | Completed output files (reports). | **Write:** Supervisor after routing. |
| **`stasis/`** | Human intervention needed (v3+ failures). | **Write:** Supervisor. **Read:** Human. |
| **`prompts/`** | System prompts for roles. | **Read Only.** |

### Archive Strategy

**What goes where:**
- **archive/inputs/**: All task files after completion
  - `*_DEV_TASK.md`, `*_MAP_REQUEST.md`, `*_BUG_FIX_REQUEST_*.md`
  - `*_QA_REQUEST_*.md`, `*_E2E_RUN_REQUEST.md`, `*_READY_FOR_QA_*.md`
- **archive/reports/**: All output files after processing
  - `*_QA_REPORT_*.md` (after routing pass/fail)
  - `*_E2E_REPORT.md`, `*_SPEC_UPDATE.md`, `*_BUG_REPORT.md`

**Rule:** Agent archives its own input. Supervisor archives reports after routing.

---

## The Roster

### 1. The Supervisor (Orchestrator)
- **Function:** Central controller. Spawns subagents. Does NOT write code.
- **Loop:** Checks `PAUSED` → `processing/` (zombies) → `reports/` (results) → `inbox/` (tasks).
- **Output:** Routes files, creates `BUG_FIX_REQUEST_*` and `QA_REQUEST_*`.
- **Light Work Allowed:** Shell checks (curl, ls), file moves, task file creation, **git commit/push**.
- **Heavy Work Forbidden:** Browser/MCP tools, code edits, running tests.
- **Special:** Moves v3+ failures to `stasis/` with NOTIFICATION. Manages dev server pre-flight.
- **Commits:** After QA_REPORT (Pass), commits verified changes and pushes to remote.

### 2. The Developer (Subagent)
- **Function:** Writes code, fixes bugs, implements features.
- **Spawned by:** Supervisor when `DEV_TASK_*` or `BUG_FIX_REQUEST_*` found.
- **Output:** Code changes, `READY_FOR_QA_*`, and CHANGELOG updates.
- **Returns:** Control to Supervisor after completion.

### 3. The Cartographer (Subagent)
- **Function:** Maps application behavior into documentation.
- **Spawned by:** Supervisor when `MAP_REQUEST_*` found.
- **Output:** `docs/specs/*.md`, `SPEC_UPDATE_*`, optionally `BUG_REPORT_*`.
- **Returns:** Control to Supervisor after completion.

### 4. The Tester (Subagent)
- **Function:** Validates code against specs. Be ruthless.
- **Spawned by:** Supervisor when `QA_REQUEST_*` found.
- **Output:** `QA_REPORT_*` (Pass/Fail).
- **Returns:** Control to Supervisor after completion.

### 5. The Regression Agent (Subagent)
- **Function:** Runs E2E tests, self-heals test bugs, routes code bugs.
- **Spawned by:** Supervisor when `E2E_RUN_REQUEST_*` found.
- **Output:** `E2E_REPORT_*`, optionally `BUG_FIX_REQUEST_*` (for code bugs).
- **Self-Healing:** Fixes selectors, timing, expected values directly without Developer.
- **Returns:** Control to Supervisor after completion.

---

## How to Run

### Starting the System
```bash
# Single terminal - Supervisor handles everything
./supervisor
# Or manually: Tell Claude to read prompts/supervisor.md and begin the loop
```

### Dispatching Tasks
```bash
# Map a feature (Cartographer)
./dispatch map "Document the user dashboard"

# Request QA testing (Tester)
./dispatch qa "Verify login works correctly"

# Request development (Developer)
./dispatch dev "Add export button to reports page"

# Run E2E regression tests (Regression)
./dispatch e2e "Run full regression suite"
```

### Supervisor Spawns Subagents
When Supervisor finds a task, it uses the Task tool:
```
Task: "You are the Developer agent. Read prompts/developer.md for your instructions. Your task file is: inbox/DEV_TASK_143022.md. Execute it."
```

Each subagent:
1. Reads its prompt file
2. Executes the single task
3. Creates output in `reports/` or `inbox/`
4. Returns control to Supervisor

---

## Protocols

### Protocol A: Map then Gap
*Used when working on legacy/undocumented features.*

```
Human ──MAP_REQUEST──▶ Cartographer ──▶ docs/specs/*.md
                             │
                 ┌───────────┴───────────┐
                 ▼                       ▼
           SPEC_UPDATE              BUG_REPORT
           (reports/)               (reports/)
                 │                       │
                 ▼                       ▼
         archive/reports/       BUG_FIX_REQUEST_v1
                                    (inbox/)
```

### Protocol B: Bug Fix Loop
*Used when a bug is found or a feature is requested.*

```
BUG_FIX_REQUEST_v1 ──▶ Developer ──▶ READY_FOR_QA_* ──▶ Supervisor ──▶ QA_REQUEST
     (inbox/)                           (inbox/)                         (inbox/)
                                                                            │
                                                                            ▼
                                                                         Tester
                                                                            │
                                                                       QA_REPORT
                                                                       (reports/)
                                                                            │
                                         ┌──────────────────────────────────┴──────┐
                                         ▼                                         ▼
                                       PASS                                      FAIL
                                         │                                         │
                                         ▼                                         ▼
                                  archive/reports/                      BUG_FIX_REQUEST_v[n+1]
                                                                        or stasis/ (if v3+)
```

### Protocol C: E2E Regression Loop
*Used to run E2E tests and fix failures.*

```
Human ──./dispatch e2e──▶ E2E_RUN_REQUEST (inbox/)
                                 │
                                 ▼
                            SUPERVISOR
                        (spawns Regression)
                                 │
                                 ▼
                          ┌────────────┐
                          │ REGRESSION │
                          └──────┬─────┘
                                 │
                ┌────────────────┴────────────────────┐
                ▼                                     ▼
           ALL PASS                               FAILURES
                │                                     │
                ▼                                     ▼
        E2E_REPORT (PASS)              Regression triages each:
         (reports/)                    - "not visible" → TEST_BUG
                │                      - "500 error" → CODE_BUG
                ▼                               │
       Supervisor archives     ┌────────────────┴────────────────┐
                               ▼                                 ▼
                          TEST_BUG                           CODE_BUG
                               │                                 │
                               ▼                                 ▼
                    Regression FIXES IT           BUG_FIX_REQUEST_v1 (inbox/)
                   (edits test files)
```

**Self-Healing:** Regression agent fixes simple test bugs (selectors, timing) directly without involving Developer. Only code bugs route to the full Developer → Tester loop.

---

## File Naming Conventions

**Format:** `YYYYMMDD_HHMMSS_TYPE[_v[n]].md` (timestamp first for chronological sorting)

| File Suffix | Created By | Picked Up By | Location |
|:---|:---|:---|:---|
| `_MAP_REQUEST` | Human | Cartographer | `inbox/` |
| `_DEV_TASK` | Human | Developer | `inbox/` |
| `_BUG_FIX_REQUEST_v[n]` | Supervisor/Regression | Developer | `inbox/` |
| `_TEST_FIX_REQUEST_v[n]` | Regression | Developer | `inbox/` |
| `_READY_FOR_QA[_v[n]]` | Developer | Supervisor | `inbox/` |
| `_QA_REQUEST[_v[n]]` | Supervisor | Tester | `inbox/` |
| `_QA_REPORT[_v[n]]` | Tester | Supervisor | `reports/` |
| `_E2E_RUN_REQUEST` | Human | Regression | `inbox/` |
| `_E2E_REPORT` | Regression | Supervisor | `reports/` |
| `_SPEC_UPDATE` | Cartographer | Supervisor | `reports/` |
| `_BUG_REPORT` | Cartographer | Supervisor | `reports/` |

**Version Propagation (CRITICAL for infinite loop prevention):**

```
BUG_FIX_REQUEST_v1 → READY_FOR_QA_v1 → QA_REQUEST_v1 → QA_REPORT_v1 (Fail) → BUG_FIX_REQUEST_v2
  (20251205_100000)   (20251205_100500)  (20251205_100501)  (20251205_101000)   (20251205_101001)
                                                                         (Pass) → archive/
```

**Version Format:** `v1`, `v2`, `v3` suffix for retry tracking. v3+ fails go to `stasis/`.

---

## Prime Directives

1. **Move First:** Your FIRST action must be `mv inbox/file processing/file`. If it fails (file not found), abort. This is your lock.

2. **File Hygiene:** Never leave a file in `inbox/` if you are working on it.

3. **Context Awareness:** Always check `docs/specs/` before asking "how should this work?".

4. **Communication:** Agents do NOT talk directly. They communicate only by creating files.

5. **Atomic Work:** One task, one report, sleep.

6. **Strict Naming:** Use EXACT file prefixes. Never hallucinate variations.

7. **Supervisor Never Works:** Supervisor is a ROUTER, not a WORKER.
   - NEVER use browser/playwright/MCP tools
   - NEVER edit code or test files
   - NEVER run tests directly
   - If tempted to do work → STOP → create task file → spawn agent

---

## Safety Mechanisms

### Pause/Resume Protocol
- **Pause:** Create `PAUSED` marker file to halt system
  ```bash
  touch PAUSED
  ```
- **Supervisor behavior:** At loop start, if `PAUSED` exists → log "System paused" → sleep 30s → recheck
- **Resume:** Remove the marker file
  ```bash
  rm PAUSED
  ```

### Zombie File Recovery
- **Trigger:** File in `processing/` for > 30 minutes.
- **Action:** Supervisor moves it back to `inbox/` with `RETRY_` prefix.
- **Naming:** `RETRY_[original-filename]_attempt[n].md`

### Infinite Loop Prevention
- **Trigger:** `BUG_FIX_REQUEST_v3` or higher.
- **Action:** Supervisor moves to `stasis/` instead of creating v4.
- **Resolution:** Human reviews and decides next steps.

### Regression Detection
**Trigger:** QA_REPORT_v2 fails with same bug as v1.
**Action:** Immediately escalate to stasis (don't create next version).
**Rationale:** If fix A breaks, fix B re-breaks with same issue, the problem is deeper than code.

### Stasis Protocol (Auto-Pause)

**When ANY file enters stasis/, Supervisor MUST:**
1. Create notification file
2. **Create `PAUSED`** - auto-pause the system
3. Log: "STASIS: Human intervention required. System paused."

**Notification File:** `stasis/NOTIFICATION_[timestamp].md`
```markdown
# STASIS NOTIFICATION
TIMESTAMP: YYYYMMDD_HHMMSS
REASON: v3+ failure | timeout | blocked | regression

## Task
- File: [original task filename]
- Type: [BUG_FIX_REQUEST | QA_REPORT | E2E_REPORT]

## Why Escalated
[Brief explanation]

## Recommended Action
[fix manually | split task | investigate root cause]
```

---

## Customization

### Adapting for Your Project

1. **Edit `prompts/*.md`** to match your project's conventions
2. **Update `dispatch`** script with your test commands and paths
3. **Add project-specific learnings** to the Learnings section below
4. **Configure credentials** in a `.env` file or auth JSON files

### Adding New Agent Types

1. Create `prompts/new-agent.md` with role definition
2. Add file pattern to Supervisor routing table
3. Update `dispatch` script with new command type

---

## Learnings

*Agents append learnings here. 1 line per item, imperative style. MAX 30 items.*

- Never spawn multiple agents in parallel - shared resources cause conflicts
- Sub-agents for role transitions preserve "fresh eyes" - Tester who just "was" Developer has same blind spots
- Decompose multi-page QA tasks into 1-2 pages per request - agents work better with focused scope
- Require evidence table in QA reports - no table means incomplete testing
- Reuse browser state between sequential tests - don't close browser, stay logged in
- Supervisor must NEVER use browser/MCP tools or edit files - always spawn agent first
- Developer must NOT commit - explicit policy prevents bypassing QA verification
- E2E multi-bug priority: CODE_BUGs first, TEST_FIXes second
