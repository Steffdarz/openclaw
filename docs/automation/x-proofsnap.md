# X ProofSnap Monitor

Automated monitoring of X (Twitter) accounts using OpenClaw's cron scheduler and a Chrome extension for capturing timestamped screenshot evidence.

## Overview

This project creates an automated system that monitors specific X (Twitter) accounts and captures screenshots of new posts as proof/evidence. It runs on a Windows PC and checks for new tweets every 15 minutes.

**Use case:** Capturing evidence of posts that may be deleted later (e.g., monitoring public figures, tracking statements for accountability).

## How It Works

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  OpenClaw       │────▶│  Python Server   │────▶│  Chrome + Ext   │
│  Cron Watchdog  │     │  (WebSocket)     │     │  (ProofSnap)    │
└─────────────────┘     └──────────────────┘     └─────────────────┘
        │                                                │
        │                                                ▼
        │                                        ┌─────────────────┐
        └───────────────────────────────────────▶│  X.com page     │
          (browser automation)                   │  Screenshots    │
                                                 └─────────────────┘
```

1. **Watchdog** triggers the cron job every 15 minutes
2. **OpenClaw agent** uses browser automation to visit X.com and find new posts
3. For each new post, it navigates to the post URL and triggers **ProofSnap**
4. **ProofSnap extension** captures a timestamped screenshot via WebSocket trigger
5. State file tracks which posts have been captured to avoid duplicates

## Prerequisites

- Windows 10/11
- [OpenClaw](https://docs.openclaw.ai) installed (`npm install -g openclaw`)
- Python 3.x installed
- Google Chrome installed
- ProofSnap Chrome extension (custom build with WebSocket trigger)

## What It Does

- Monitors a target X account (e.g., @elonmusk) every 15 minutes
- Captures timestamped screenshots of new tweets via ProofSnap Chrome extension
- Stores proof snapshots locally with metadata
- Tracks captured posts to avoid duplicates
- Skips pinned posts and reposts automatically

## Components

1. **OpenClaw Gateway** - Runs the cron scheduler (auto-starts on boot)
2. **Python HTTP+WebSocket Server** - Bridges OpenClaw to Chrome extension without stealing focus
3. **ProofSnap Chrome Extension** - Custom build with WebSocket trigger in clawd browser profile

## File Locations

| Item | Path |
|------|------|
| Config | `C:\Users\Asus\.openclaw\openclaw.json` |
| Cron jobs | `C:\Users\Asus\.openclaw\cron\jobs.json` |
| Logs | `C:\tmp\openclaw\openclaw-YYYY-MM-DD.log` |
| State file | `C:\Users\Asus\.openclaw\workspace\x-proofsnap\state.json` |
| Trigger script | `C:\Users\Asus\.openclaw\workspace\x-proofsnap\trigger-proofsnap.ps1` |
| Python server | `C:\ProofSnap\native-host\proofsnap_host.py` |
| Cron watchdog | `C:\ProofSnap\cron-watchdog.ps1` |

## Current Cron Job

- **Name:** X ProofSnap Monitor - Elon
- **ID:** `5cd4effc-f074-4a5d-8b5c-90ef4cef80d8`
- **Schedule:** `*/15 * * * *` (every 15 minutes)

**Job Message:**
```
Monitor @elonmusk for NEW posts and replies. Steps:
1) Use browser tool to go to https://x.com/elonmusk/with_replies
2) Read ~/.openclaw/workspace/x-proofsnap/state.json to get capturedPosts
3) Scroll and collect post IDs (look for /elonmusk/status/ID links, skip pinned posts) until you find 10 NEW IDs not in capturedPosts, or reach end of reasonable scroll
4) For EACH new post ID:
   a) Use browser tool to navigate to https://x.com/elonmusk/status/POST_ID
   b) Wait 1 second
   c) Run: powershell -ExecutionPolicy Bypass -File C:/Users/Asus/.openclaw/workspace/x-proofsnap/trigger-proofsnap.ps1
   d) Wait 1 second
5) Update state.json with all new IDs. Max 10 captures per run.
```

## Startup After PC Restart

Run these in order (each in a separate terminal, keep all open):

### Step 1 - Start Gateway

```powershell
openclaw gateway run --bind loopback --port 18789 --force
```

### Step 2 - Start Python Server

```powershell
python -u C:\ProofSnap\native-host\proofsnap_host.py
```

### Step 3 - Start Browser

```powershell
openclaw browser start
```

### Step 4 - Check Extension Connection

```powershell
curl http://localhost:19999/status
```

If `connected_clients: 0`:
1. Go to `chrome://extensions/` in the clawd browser
2. Find ProofSnap extension and click **Reload**
3. Run the curl command again until `connected_clients: 1`

### Step 5 - Start Cron Watchdog

Due to a known issue with OpenClaw's internal cron timer on Windows (timers may not fire reliably), use this watchdog script:

```powershell
powershell -ExecutionPolicy Bypass -File C:\ProofSnap\cron-watchdog.ps1
```

This runs the job every 15 minutes reliably.

### Summary - 4 Terminals Running

1. Gateway: `openclaw gateway run --bind loopback --port 18789 --force`
2. Python: `python -u C:\ProofSnap\native-host\proofsnap_host.py`
3. Browser started via: `openclaw browser start` (terminal can close after)
4. Watchdog: `powershell -ExecutionPolicy Bypass -File C:\ProofSnap\cron-watchdog.ps1`

## Useful Commands

### Cron Status

```powershell
# Check cron status
openclaw cron status

# List all jobs
openclaw cron list --all

# Force run the job now
openclaw cron run 5cd4effc-f074-4a5d-8b5c-90ef4cef80d8 --force

# View run history
openclaw cron runs --id 5cd4effc-f074-4a5d-8b5c-90ef4cef80d8
```

### Gateway

```powershell
# Check gateway status
openclaw gateway status

# Start gateway manually (if needed)
openclaw gateway run --bind loopback --port 18789 --force
```

### Server Connection

```powershell
# Check ProofSnap server status (use curl, not Invoke-RestMethod)
curl http://localhost:19999/status
```

### Clear Captured Posts

To re-capture posts that were already captured:

1. Open `C:\Users\Asus\.openclaw\workspace\x-proofsnap\state.json` in Notepad
2. Set `"capturedPosts": []`
3. Save

## Known Issues

| Issue | Workaround |
|-------|------------|
| OpenClaw cron timer doesn't fire on Windows | Use the watchdog script (`cron-watchdog.ps1`) instead |
| `Invoke-RestMethod` hangs in PowerShell | Use `curl` instead |
| Ctrl+C in PowerShell kills Python server | Use mouse to select/copy text |
| Extension disconnects after browser start | Reload extension at `chrome://extensions/`, check for `connected_clients: 1` |
| Some users' `/with_replies` page is empty | Use regular profile URL instead |

## Cron Watchdog Script

If you need to recreate `C:\ProofSnap\cron-watchdog.ps1`:

```powershell
@'
# OpenClaw Cron Watchdog
# Runs the X ProofSnap job every 15 minutes

$jobId = "5cd4effc-f074-4a5d-8b5c-90ef4cef80d8"
$intervalMinutes = 15

Write-Host "OpenClaw Cron Watchdog started"
Write-Host "Job ID: $jobId"
Write-Host "Interval: $intervalMinutes minutes"
Write-Host "Press Ctrl+C to stop"
Write-Host ""

while ($true) {
    $now = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    Write-Host "[$now] Running cron job..."
    
    & openclaw cron run $jobId --force 2>&1 | Out-Null
    
    $next = (Get-Date).AddMinutes($intervalMinutes).ToString("HH:mm:ss")
    Write-Host "[$now] Done. Next run at $next"
    Write-Host ""
    
    Start-Sleep -Seconds ($intervalMinutes * 60)
}
'@ | Out-File -FilePath "C:\ProofSnap\cron-watchdog.ps1" -Encoding UTF8
```

To use a different job ID, edit the `$jobId` variable in the script.

## Not Yet Automated

- Auto-start for Python server on boot
- Auto-start for clawd browser on boot
- Auto-start for watchdog script on boot

Currently requires manual startup after PC restart (see Startup section above).

## Quick Start (TL;DR)

After PC restart, open 3 PowerShell windows and run:

```powershell
# Terminal 1: Gateway
openclaw gateway run --bind loopback --port 18789 --force

# Terminal 2: Python server
python -u C:\ProofSnap\native-host\proofsnap_host.py

# Terminal 3: Browser + watchdog
openclaw browser start
curl http://localhost:19999/status  # should show connected_clients: 1
powershell -ExecutionPolicy Bypass -File C:\ProofSnap\cron-watchdog.ps1
```

If `connected_clients: 0`, reload ProofSnap extension at `chrome://extensions/` and check again.

## Changing the Target Account

To monitor a different account (e.g., @realDonaldTrump instead of @elonmusk):

1. Update the cron job message:
   ```powershell
   openclaw cron update 5cd4effc-f074-4a5d-8b5c-90ef4cef80d8 --message "Monitor @realDonaldTrump for NEW posts..."
   ```

2. Clear the state file to start fresh:
   - Open `C:\Users\Asus\.openclaw\workspace\x-proofsnap\state.json`
   - Set `"capturedPosts": []`

## Creating a New Cron Job

To create a new monitoring job from scratch:

```powershell
openclaw cron add --name "X ProofSnap Monitor - Someone" --schedule "*/15 * * * *" --target isolated --message "Monitor @username for NEW posts and replies. Steps: 1) Use browser tool to go to https://x.com/username/with_replies 2) Read ~/.openclaw/workspace/x-proofsnap/state.json to get capturedPosts 3) Scroll and collect post IDs until you find 10 NEW IDs 4) For EACH new post: navigate to it, wait 1s, run trigger-proofsnap.ps1, wait 1s 5) Update state.json. Max 10 captures per run."
```

Then update the watchdog script with the new job ID.
