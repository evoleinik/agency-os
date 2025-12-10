# Pre-flight Check Protocol

**Required before:** Tester, Cartographer, Regression agents (anything using browser)

## Check Dev Server

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000
```

## Decision Tree

```
Response 200?
├── YES → Proceed with task
└── NO → Start dev server
         │
         ├── Run: npm run dev (or your start command)
         ├── Wait: up to 30 seconds
         ├── Recheck: curl localhost:3000
         │
         └── Response 200?
             ├── YES → Proceed with task
             └── NO → Escalate to STASIS
                      Create NOTIFICATION with:
                      - REASON: Dev server won't start
                      - ACTION: Check server logs manually
```

## Implementation

```bash
#!/bin/bash
check_server() {
  local status=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000)
  if [ "$status" = "200" ]; then
    echo "Server running"
    return 0
  else
    echo "Server not responding (status: $status)"
    return 1
  fi
}

# Check
if ! check_server; then
  echo "Starting dev server..."
  npm run dev &

  # Wait up to 30s
  for i in {1..30}; do
    sleep 1
    if check_server; then
      exit 0
    fi
  done

  echo "STASIS: Dev server failed to start"
  exit 1
fi
```

## Common Issues

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Connection refused | Server not running | Start with `npm run dev` |
| 500 error | Server crashed | Check logs, restart |
| Timeout | Port blocked | Check if port 3000 in use |
| 404 on root | Wrong port | Verify PORT env variable |

## Customization

Update the port (3000) and start command (`npm run dev`) for your project.
