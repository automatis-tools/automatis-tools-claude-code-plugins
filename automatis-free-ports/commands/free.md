# Free Ports

Universal skill to detect, diagnose, and manage port conflicts on macOS. Works for any project by using the current working directory to identify project ownership.

## When to Use

- Service fails to start due to port conflict ("address already in use")
- Need to restart services cleanly
- Want to check what's using specific ports
- Before starting any service that uses a known port

## Arguments

The user can provide port number(s) as arguments:
- `/automatis-free-ports:free 8000` - Check single port
- `/automatis-free-ports:free 8000 8001 8080` - Check multiple ports
- `/automatis-free-ports:free` - Ask user which port(s) to check

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

**If process is from current project:**
- Auto-kill without confirmation:
```bash
kill -9 PID
```

**If process is NOT from current project:**
- Warn the user with process details
- Ask if they want to kill it anyway
- If yes, proceed with `kill -9 PID`

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
| Kill by PID | `kill -9 PID` |
| Kill by port (quick) | `lsof -i :PORT -t \| xargs kill -9` |

## Safety Rules

1. Never kill system processes (PID < 100)
2. Always warn and ask before killing non-project processes
3. Always verify port is free after killing
4. Show process details before any kill action
