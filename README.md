# Agency OS

**File-based multi-agent orchestration for autonomous software development.**

Agency OS coordinates AI agents through the filesystem. No message queues, no databases, no complex infrastructure. Just directories as queues, file moves as locks, and naming conventions as state machines.

## Why Files?

Traditional coordination mechanisms (Redis, RabbitMQ, databases) solve distributed problems. But when your agents run sequentially on a single machine, they add complexity without benefit.

**Files give you:**
- **Instant debugging**: `ls inbox/` shows your queue
- **Natural persistence**: Survives crashes, state visible on restart
- **Atomic locking**: `mv` is atomic on POSIX systems
- **Human-readable state**: Every task is a markdown file you can read and edit
- **Zero dependencies**: Nothing to install, configure, or maintain

## Quick Start

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/agency-os.git
cd agency-os

# Make scripts executable
chmod +x dispatch supervisor

# Dispatch your first task
./dispatch dev "Add a hello world endpoint"

# Start the supervisor (requires Claude CLI)
./supervisor
```

## Architecture

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
├── dispatch         # CLI for creating tasks
├── supervisor       # Starts the supervisor agent
└── CLAUDE.md        # Full system documentation
```

## The Agent Team

| Agent | Role | Creates |
|-------|------|---------|
| **Supervisor** | Routes tasks, spawns agents, commits code | QA_REQUEST, BUG_FIX_REQUEST |
| **Developer** | Writes code, fixes bugs | READY_FOR_QA |
| **Tester** | Validates features, explores UI | QA_REPORT |
| **Regression** | Runs E2E tests, self-heals test bugs | E2E_REPORT, BUG_FIX_REQUEST |
| **Cartographer** | Documents features | SPEC_UPDATE, BUG_REPORT |

## Dispatching Tasks

```bash
# Document a feature (Cartographer)
./dispatch map "Document the user dashboard"

# Test a feature (Tester)
./dispatch qa "Verify login flow works"

# Implement something (Developer)
./dispatch dev "Add export button to reports"

# Run E2E tests (Regression)
./dispatch e2e "Run full regression suite"
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

1. **Edit `prompts/*.md`** to match your conventions
2. **Update `dispatch`** with your test commands
3. **Add project learnings** to `CLAUDE.md`

### Adding New Agent Types

1. Create `prompts/new-agent.md`
2. Add file pattern to Supervisor routing
3. Update `dispatch` with new command type

## Requirements

- [Claude CLI](https://claude.ai/code) - for running agents
- Bash - for dispatch script
- Git - for version control

## License

MIT

## Contributing

1. Fork the repo
2. Create a feature branch
3. Make your changes
4. Submit a PR

Issues and suggestions welcome!
