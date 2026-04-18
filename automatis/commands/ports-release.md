---
description: Release port conflicts on macOS by killing the offending process
argument-hint: "[port] [port...]"
allowed-tools: Bash
---

# Release Ports

Detect, diagnose, and release port conflicts on macOS. Works for any project by using the current working directory to identify project ownership.

## When to Use

- Service fails to start due to port conflict ("address already in use")
- Need to restart services cleanly
- Want to check what's using specific ports
- Before starting any service that uses a known port

## Arguments

The user can provide port number(s) as arguments:
- `/automatis:ports-release 8000` - Check single port
- `/automatis:ports-release 8000 8001 8080` - Check multiple ports
- `/automatis:ports-release` - Ask user which port(s) to check

## Procedure

### Step 1: Get Port(s) to Check

If user provided port(s) as arguments, use those. Otherwise, ask:
```
Which port(s) do you want to check? (e.g., 8000 or 8000 8001 8080)
```

### Step 2: Check Each Port

For each port, run:
```bash
lsof -i :PORT
```

If no output, the port is free. Report and move to next port.

### Step 3: Identify Process Owner

For occupied ports, determine if the process belongs to current project:

```bash
# Get PID
PID=$(lsof -i :PORT -t)

# Get process working directory
lsof -p $PID | grep cwd

# Get process command line
ps -p $PID -o pid,args=
```

Compare the process CWD and command against the current working directory.

**Project process criteria**: The process CWD or command path contains the current working directory path.

### Step 4: Take Action

**Always try graceful termination first, then escalate to SIGKILL if needed.** SIGKILL prevents process cleanup (buffer flush, lock release, connection close) and can corrupt state for processes holding DB connections, open files, or network sessions.

```bash
# Graceful: SIGTERM. Give the process a few seconds to exit cleanly.
kill "$PID" && sleep 3

# Escalate only if the process is still alive:
if kill -0 "$PID" 2>/dev/null; then
  kill -9 "$PID"
fi
```

**If process is from current project:** run the block above without asking.

**If process is NOT from current project:** show the user the process details, ask before running the block, proceed only on confirmation.

**Safety check**: Never kill processes with PID < 100 (system processes).

### Step 5: Verify Port is Free

```bash
lsof -i :PORT
```

Should return empty if port is now free.

### Step 6: Restart Service (Optional)

Ask user if they want to restart the service. If yes, ask for the restart command or look for common patterns:
- `make dev` in project directory
- `npm start` / `yarn start` for Node.js
- `python -m module` for Python

## Quick Reference Commands

| Task | Command |
|------|---------|
| Check port | `lsof -i :PORT` |
| Get PID only | `lsof -i :PORT -t` |
| Get process CWD | `lsof -p PID \| grep cwd` |
| Get process details | `ps -p PID -o pid,args=` |
| Graceful kill by PID | `kill PID` (SIGTERM) |
| Force-kill by PID | `kill -9 PID` (SIGKILL — last resort) |
| Check PID still alive | `kill -0 PID` (exit 0 = alive, 1 = gone) |

## Safety Rules

1. Never kill system processes (PID < 100)
2. Always warn and ask before killing non-project processes
3. Always verify port is free after killing
4. Show process details before any kill action
