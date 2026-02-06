# X ProofSnap Monitor

Automated monitoring of X (Twitter) accounts using OpenClaw's cron scheduler and a Chrome extension for capturing screenshots.

## What It Does

- Monitors a target X account (e.g., @elonmusk) every 15 minutes via cron
- Captures timestamped screenshots of new tweets via ProofSnap Chrome extension
- Stores proof snapshots locally with metadata
- Tracks captured posts to avoid duplicates

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

Run these in order:

### Step 1 - Start Python Server (keep window open)

```powershell
python -u C:\ProofSnap\native-host\proofsnap_host.py
```

### Step 2 - Start Clawd Browser (new PowerShell window)

```powershell
openclaw browser start
```

### Step 3 - Check Connection

```powershell
curl http://localhost:19999/status
```

### Step 4 - Fix Extension if Needed

If `connected_clients: 0`:
1. Go to `chrome://extensions/` in the clawd browser
2. Find ProofSnap extension and click **Reload**

### Step 5 - Verify

Once `connected_clients: 1`, cron will auto-run every 15 minutes.

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
| `Invoke-RestMethod` hangs in PowerShell | Use `curl` instead |
| Ctrl+C in PowerShell kills Python server | Use mouse to select/copy text |
| Extension disconnects after browser start | Reload extension at `chrome://extensions/`, check for `connected_clients: 1` |
| Some users' `/with_replies` page is empty | Use regular profile URL instead |

## Not Yet Automated

- Auto-start for Python server on boot
- Auto-start for clawd browser on boot

Currently requires manual startup after PC restart (see Startup section above).
