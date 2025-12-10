# Agency OS

**File-based multi-agent orchestration for autonomous software development.**

Agency OS coordinates AI agents through the filesystem. No message queues, no databases, no complex infrastructure. Just directories as queues, file moves as locks, and naming conventions as state machines.

## Project Integration

Agency OS lives **inside your project** as a subdirectory:

```
my-project/
├── src/                    # Your code
├── tests/                  # Your tests
├── package.json
├── CLAUDE.md               # Project rules (agents read this)
│
└── agency-os/              # ← Clone here
    ├── inbox/              # Task queue
    ├── prompts/            # Agent definitions
    ├── shared/             # Common protocols
    └── ...
```

**How agents access your code:** They reference the parent directory (`../src`, `../tests`, `../package.json`). Your project's `CLAUDE.md` is read by all agents for project-specific rules.

**What to customize:**
- `shared/pre-flight.md` - Dev server port (default: 3000) and start command (`npm run dev`)
- `prompts/*.md` - Test commands, file conventions for your stack

## Quick Start

```bash
# From your project root
cd my-project
git clone https://github.com/evoleinik/agency-os.git
cd agency-os

# Start the supervisor (requires Claude CLI)
./supervisor

# Or just talk to Claude directly:
# "Read CLAUDE.md and prompts/supervisor.md, then begin your supervisor loop"
```

That's it. Just talk to the Supervisor in natural language:

```
You: "Add a user profile page with avatar upload"
You: "Test the login flow"
You: "Run the E2E tests and fix any failures"
You: "Document the dashboard features"
```

The Supervisor decomposes your request into tasks, creates the appropriate files in `inbox/`, and spawns specialized agents to execute them.

## Architecture

```
                              ┌─────────┐
                              │  Human  │
                              └────┬────┘
                                   │
                        conversational request
                                   │
                                   ▼
                        ┌─────────────────────┐
                        │     SUPERVISOR      │
                        │ (decomposes request │
                        │  into task files)   │
                        └──────────┬──────────┘
                                   │
                   Creates task files in inbox/
                                   │
                                   ▼
                           ┌───────────────┐
                           │    inbox/     │
                           │ (task queue)  │
                           └───────┬───────┘
                                   │
                                   ▼
                  ┌────────────────────────────────┐
                  │   EXECUTION PHASE (SUPERVISOR) │
                  │  (process tasks sequentially)  │
                  └──┬─────┬─────┬─────┬─────┬────┘
                     │     │     │     │     │
                     ▼     ▼     ▼     ▼     ▼
              ┌──────────────────────────────────────┐
              │           SUBAGENTS                   │
              │  Developer | Tester | Regression |   │
              │            Cartographer              │
              └──────────────────────────────────────┘
```

## Why Files?

Traditional coordination mechanisms (Redis, RabbitMQ, databases) solve distributed problems. But when your agents run sequentially on a single machine, they add complexity without benefit.

**Files give you:**
- **Instant debugging**: `ls inbox/` shows your queue
- **Natural persistence**: Survives crashes, state visible on restart
- **Atomic locking**: `mv` is atomic on POSIX systems
- **Human-readable state**: Every task is a markdown file you can read and edit
- **Zero dependencies**: Nothing to install, configure, or maintain

## Directory Structure

```
agency-os/
├── inbox/           # Tasks waiting to be processed
├── processing/      # Active workspace (agent's lock)
├── reports/         # Output from completed work
├── archive/         # Completed tasks and reports
│   ├── inputs/      # Archived task files
│   └── reports/     # Archived report files
├── stasis/          # Human intervention needed
├── prompts/         # System prompts for each agent
├── supervisor       # Starts the supervisor agent
└── CLAUDE.md        # Full system documentation
```

## The Agent Team

| Agent | Role | Creates |
|-------|------|---------|
| **Supervisor** | Decomposes requests, routes tasks, spawns agents, commits code | DEV_TASK, QA_REQUEST, BUG_FIX_REQUEST |
| **Developer** | Writes code, fixes bugs | READY_FOR_QA |
| **Tester** | Validates features, explores UI | QA_REPORT |
| **Regression** | Runs E2E tests, self-heals test bugs | E2E_REPORT, BUG_FIX_REQUEST |
| **Cartographer** | Documents features | SPEC_UPDATE, BUG_REPORT |

## How It Works

1. **You talk to the Supervisor** in natural language
2. **Supervisor creates task files** in `inbox/` based on your request
3. **Supervisor spawns the right agent** for each task
4. **Agent executes**, creates output, returns control
5. **Supervisor routes results** and continues until done

Example conversation:
```
You: "Fix the bug where users can submit empty forms, then verify it works"

Supervisor creates:
  inbox/20251210_100000_BUG_FIX_REQUEST_v1.md  (fix the bug)
  inbox/20251210_100001_QA_REQUEST.md          (verify the fix)

Then processes them sequentially:
  1. Spawns Developer → fixes bug → creates READY_FOR_QA
  2. Spawns Tester → verifies fix → creates QA_REPORT (PASS)
  3. Commits the verified code
```

## File Naming Convention

All files follow: `YYYYMMDD_HHMMSS_TYPE[_vN].md`

Examples:
- `20251205_143022_DEV_TASK.md`
- `20251205_143500_READY_FOR_QA_v1.md`
- `20251205_144000_QA_REPORT_v1.md`
- `20251205_144500_BUG_FIX_REQUEST_v2.md`

Timestamps ensure chronological processing. Version suffixes track retry attempts.

## Safety Mechanisms

### Infinite Loop Prevention
```
BUG_FIX_REQUEST_v1 → fails → v2 → fails → v3 → stasis/
```
After 3 failed attempts, tasks go to `stasis/` for human review. The system auto-pauses.

### Regression Detection
If v2 fails with the same bug as v1, it's immediately escalated to stasis. The system recognizes it's not making progress.

### Zombie Recovery
Files stuck in `processing/` for >30 minutes get moved back to `inbox/` with a `RETRY_` prefix.

### Pause/Resume
```bash
# Pause the system
touch PAUSED

# Resume
rm PAUSED
```

## Role Separation

Hard boundaries prevent common AI agent pitfalls:

| Agent | Does | Cannot |
|-------|------|--------|
| **Supervisor** | Routes, spawns, commits | Write code, use browser |
| **Developer** | Writes code, runs build | Commit (QA must pass first) |
| **Tester** | Validates, explores | Run full test suite |
| **Regression** | Runs tests, self-heals | Fix application code |

The **"Supervisor Never Works"** rule is critical. When tempted to "just quickly fix this," it must create a task file and spawn an agent instead.

## Sequential Processing

The system processes **one task at a time**. No parallelism.

**Why:**
- Developer and Cartographer share the codebase - concurrent edits conflict
- Tester uses a single browser instance
- Simplicity eliminates race conditions

## Customization

### Adapting for Your Project

1. **`shared/pre-flight.md`** - Change dev server settings:
   - Port (default: `3000`)
   - Start command (default: `npm run dev`)

2. **`prompts/*.md`** - Match your stack:
   - Test runner commands
   - Build commands
   - File path conventions

3. **Project root `CLAUDE.md`** - Add project-specific rules that all agents must follow

### Adding New Agent Types

1. Create `prompts/new-agent.md` with role definition
2. Add file pattern to Supervisor routing in `prompts/supervisor.md`

## Requirements

- [Claude Code](https://claude.ai/code) - for running agents

## License

MIT

## Contributing

1. Fork the repo
2. Create a feature branch
3. Make your changes
4. Submit a PR

Issues and suggestions welcome!
