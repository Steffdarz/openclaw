# X ProofSnap Monitor

Automated monitoring of X (Twitter) accounts using OpenClaw's cron scheduler and a Chrome extension for capturing screenshots.

## What It Does

- Monitors a target X account (e.g., @elonmusk) on a schedule
- Captures timestamped screenshots of new tweets via Chrome extension
- Stores proof snapshots locally with metadata

## Setup

### Components

1. **OpenClaw Gateway** - Runs the cron scheduler
2. **Python WebSocket Server** - Bridges OpenClaw to Chrome extension
3. **Chrome Extension** - Captures X page screenshots

### Starting All Components

You need 3 terminals running:

**Terminal 1 - OpenClaw Gateway:**
```powershell
openclaw gateway run --bind loopback --port 18789 --force
```

**Terminal 2 - Python WebSocket Server:**
```powershell
cd C:\ProofSnap\native-host
python proofsnap_host.py
```

**Terminal 3 - Verify everything is running:**
```powershell
openclaw cron status
openclaw cron list
```

### Cron Job

```bash
# Create the monitoring job
openclaw cron add \
  --name "X ProofSnap Monitor - Elon" \
  --schedule "*/15 * * * *" \
  --target isolated \
  --message "Check @elonmusk for new tweets and capture screenshots"
```

## Useful Commands

### Gateway

```powershell
# Start gateway
openclaw gateway run --bind loopback --port 18789 --force

# Check status
openclaw gateway status
openclaw channels status --probe
```

### Cron Jobs

```bash
# List all jobs
openclaw cron list

# Run job manually
openclaw cron run --id <job-id>

# View job details
openclaw cron get --id <job-id>

# Pause/resume
openclaw cron pause --id <job-id>
openclaw cron resume --id <job-id>

# Delete job
openclaw cron delete --id <job-id>
```

### Logs

```powershell
# View cron activity (PowerShell)
Get-Content \tmp\openclaw\openclaw-*.log | Select-String "cron"

# Follow logs
Get-Content \tmp\openclaw\openclaw-*.log -Tail 50 -Wait
```

### Manual Trigger via RPC

```powershell
$body = @{
    method = "cron.run"
    params = @{ id = "<job-id>" }
    id = 1
} | ConvertTo-Json -Compress

Invoke-RestMethod -Uri "http://127.0.0.1:18789/rpc" -Method POST -ContentType "application/json" -Body $body
```

## File Locations

| Item | Path (Windows) |
|------|----------------|
| Config | `C:\Users\<user>\.openclaw\openclaw.json` |
| Cron jobs | `C:\Users\<user>\.openclaw\cron\jobs.json` |
| Logs | `C:\tmp\openclaw\openclaw-YYYY-MM-DD.log` |
| Python server | `C:\ProofSnap\native-host\proofsnap_host.py` |
