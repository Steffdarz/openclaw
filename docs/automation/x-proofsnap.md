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

---

# Full Setup Guide (From Scratch)

This section explains how to set up everything from zero on a new Windows machine.

## Step 1: Install OpenClaw

```powershell
# Install Node.js first (https://nodejs.org), then:
npm install -g openclaw

# Run the setup wizard
openclaw

# Verify installation
openclaw --version
```

## Step 2: Configure OpenClaw

```powershell
# Enable browser automation
openclaw config set browser.enabled true

# Enable cron
openclaw config set cron.enabled true

# View your config
openclaw config get
```

## Step 3: Create Project Folders

```powershell
# Create workspace folder
mkdir C:\Users\Asus\.openclaw\workspace\x-proofsnap

# Create ProofSnap folder
mkdir C:\ProofSnap
mkdir C:\ProofSnap\native-host
```

## Step 4: Create the State File

Create `C:\Users\Asus\.openclaw\workspace\x-proofsnap\state.json`:

```json
{
  "capturedPosts": [],
  "lastCheckedAt": null
}
```

## Step 5: Create the Trigger Script

Create `C:\Users\Asus\.openclaw\workspace\x-proofsnap\trigger-proofsnap.ps1`:

```powershell
# ProofSnap HTTP Trigger Script
# Sends HTTP request to local Python server to trigger screenshot capture

$serverUrl = "http://localhost:19999/trigger"

try {
    $response = Invoke-WebRequest -Uri $serverUrl -Method POST -TimeoutSec 10 -UseBasicParsing
    if ($response.StatusCode -eq 200) {
        Write-Host "ProofSnap triggered successfully"
    } else {
        Write-Host "ProofSnap trigger returned: $($response.StatusCode)"
    }
} catch {
    Write-Host "ProofSnap HTTP trigger failed: $($_.Exception.Message)"
}
```

## Step 6: Create the Python WebSocket Server

Create `C:\ProofSnap\native-host\proofsnap_host.py`:

```python
#!/usr/bin/env python3
"""
ProofSnap Trigger Server
HTTP server (port 19999) + WebSocket server (port 19998)
Receives HTTP POST /trigger and broadcasts to connected Chrome extensions
"""

import asyncio
import json
import threading
from http.server import HTTPServer, BaseHTTPRequestHandler
from typing import Set

# Try to import websockets, install if not available
try:
    import websockets
    from websockets.server import serve
except ImportError:
    import subprocess
    import sys
    print("Installing websockets...")
    subprocess.check_call([sys.executable, "-m", "pip", "install", "websockets"])
    import websockets
    from websockets.server import serve

HOST = "localhost"
HTTP_PORT = 19999
WS_PORT = 19998

# Connected WebSocket clients
clients: Set[websockets.WebSocketServerProtocol] = set()

class TriggerHandler(BaseHTTPRequestHandler):
    def log_message(self, format, *args):
        print(f"[HTTP] {args[0]}")

    def _send_response(self, status: int, body: dict):
        self.send_response(status)
        self.send_header("Content-Type", "application/json")
        self.send_header("Access-Control-Allow-Origin", "*")
        self.end_headers()
        self.wfile.write(json.dumps(body).encode())

    def do_OPTIONS(self):
        self.send_response(200)
        self.send_header("Access-Control-Allow-Origin", "*")
        self.send_header("Access-Control-Allow-Methods", "GET, POST, OPTIONS")
        self.send_header("Access-Control-Allow-Headers", "Content-Type")
        self.end_headers()

    def do_GET(self):
        if self.path == "/status":
            self._send_response(200, {
                "ok": True,
                "service": "ProofSnap Trigger Server",
                "version": "2.0.0",
                "websocket_port": WS_PORT,
                "connected_clients": len(clients)
            })
        else:
            self._send_response(404, {"error": "Not found"})

    def do_POST(self):
        if self.path == "/trigger":
            if len(clients) == 0:
                self._send_response(503, {
                    "ok": False,
                    "error": "No ProofSnap extension connected"
                })
                return
            
            # Broadcast trigger to all connected clients
            message = json.dumps({"action": "capture"})
            asyncio.run(broadcast(message))
            
            self._send_response(200, {
                "ok": True,
                "message": "Trigger sent",
                "clients": len(clients)
            })
        else:
            self._send_response(404, {"error": "Not found"})

async def broadcast(message: str):
    if clients:
        await asyncio.gather(*[client.send(message) for client in clients])
        print(f"[WS] Broadcast to {len(clients)} client(s)")

async def ws_handler(websocket: websockets.WebSocketServerProtocol):
    clients.add(websocket)
    print(f"[WS] Client connected. Total: {len(clients)}")
    try:
        async for message in websocket:
            print(f"[WS] Received: {message}")
    except websockets.exceptions.ConnectionClosed:
        pass
    finally:
        clients.discard(websocket)
        print(f"[WS] Client disconnected. Total: {len(clients)}")

def run_http_server():
    server = HTTPServer((HOST, HTTP_PORT), TriggerHandler)
    print(f"HTTP server: http://{HOST}:{HTTP_PORT}")
    print(f"  GET  /status  - Check server status")
    print(f"  POST /trigger - Trigger screenshot capture")
    server.serve_forever()

async def run_ws_server():
    print(f"WebSocket server: ws://{HOST}:{WS_PORT}")
    async with serve(ws_handler, HOST, WS_PORT):
        await asyncio.Future()  # Run forever

def main():
    print("=" * 50)
    print("ProofSnap Trigger Server v2.0.0")
    print("=" * 50)
    
    # Start HTTP server in a thread
    http_thread = threading.Thread(target=run_http_server, daemon=True)
    http_thread.start()
    
    # Run WebSocket server in main thread
    asyncio.run(run_ws_server())

if __name__ == "__main__":
    main()
```

## Step 7: Create the Cron Watchdog

Create `C:\ProofSnap\cron-watchdog.ps1` (run this command in PowerShell):

```powershell
@'
# OpenClaw Cron Watchdog
# Runs the X ProofSnap job every 15 minutes

$jobId = "YOUR_JOB_ID_HERE"  # Replace with your actual job ID
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

## Step 8: Set Up the ProofSnap Chrome Extension

The ProofSnap extension needs to connect to the WebSocket server. You'll need a Chrome extension with:

1. **manifest.json** with permissions for `activeTab`, `scripting`, and connecting to `ws://localhost:19998`
2. **background.js** that connects to WebSocket and listens for `{"action": "capture"}` messages
3. When triggered, capture a screenshot with timestamp overlay

Example extension background script (`background.js`):

```javascript
// Connect to local WebSocket server
let ws = null;

function connect() {
  ws = new WebSocket("ws://localhost:19998");
  
  ws.onopen = () => {
    console.log("ProofSnap: Connected to trigger server");
  };
  
  ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    if (data.action === "capture") {
      captureScreenshot();
    }
  };
  
  ws.onclose = () => {
    console.log("ProofSnap: Disconnected, reconnecting in 5s...");
    setTimeout(connect, 5000);
  };
  
  ws.onerror = (error) => {
    console.error("ProofSnap: WebSocket error", error);
  };
}

async function captureScreenshot() {
  // Get the active tab
  const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
  if (!tab) return;
  
  // Capture visible tab
  const dataUrl = await chrome.tabs.captureVisibleTab(null, { format: "png" });
  
  // Create filename with timestamp
  const now = new Date();
  const timestamp = now.toISOString().replace(/[:.]/g, "-");
  const filename = `proofsnap-${timestamp}.png`;
  
  // Download the screenshot
  chrome.downloads.download({
    url: dataUrl,
    filename: filename,
    saveAs: false
  });
  
  console.log(`ProofSnap: Captured ${filename}`);
}

// Connect on extension load
connect();
```

Example `manifest.json`:

```json
{
  "manifest_version": 3,
  "name": "ProofSnap",
  "version": "2.0.0",
  "description": "Capture timestamped screenshots on command",
  "permissions": [
    "activeTab",
    "downloads",
    "tabs"
  ],
  "host_permissions": [
    "ws://localhost:19998/*"
  ],
  "background": {
    "service_worker": "background.js"
  }
}
```

Load the extension:
1. Go to `chrome://extensions/`
2. Enable "Developer mode"
3. Click "Load unpacked"
4. Select the folder containing `manifest.json`

## Step 9: Create the Cron Job

```powershell
openclaw cron add `
  --name "X ProofSnap Monitor - Elon" `
  --schedule "*/15 * * * *" `
  --target isolated `
  --message "Monitor @elonmusk for NEW posts and replies. Steps:
1) Use browser tool to go to https://x.com/elonmusk/with_replies
2) Read ~/.openclaw/workspace/x-proofsnap/state.json to get capturedPosts
3) Scroll and collect post IDs (look for /elonmusk/status/ID links, skip pinned posts) until you find 10 NEW IDs not in capturedPosts, or reach end of reasonable scroll
4) For EACH new post ID:
   a) Use browser tool to navigate to https://x.com/elonmusk/status/POST_ID
   b) Wait 1 second
   c) Run: powershell -ExecutionPolicy Bypass -File C:/Users/Asus/.openclaw/workspace/x-proofsnap/trigger-proofsnap.ps1
   d) Wait 1 second
5) Update state.json with all new IDs. Max 10 captures per run."
```

Note the job ID from the output - you'll need it for the watchdog script.

## Step 10: Update Watchdog with Job ID

Edit `C:\ProofSnap\cron-watchdog.ps1` and replace `YOUR_JOB_ID_HERE` with the actual job ID from Step 9.

## Step 11: Start Everything

Open 3 PowerShell terminals:

**Terminal 1 - Gateway:**
```powershell
openclaw gateway run --bind loopback --port 18789 --force
```

**Terminal 2 - Python Server:**
```powershell
python -u C:\ProofSnap\native-host\proofsnap_host.py
```

**Terminal 3 - Browser + Watchdog:**
```powershell
openclaw browser start
# Wait for browser to open, then check connection:
curl http://localhost:19999/status
# If connected_clients: 0, reload extension at chrome://extensions/
# Then start watchdog:
powershell -ExecutionPolicy Bypass -File C:\ProofSnap\cron-watchdog.ps1
```

## Verify Setup

1. **Check gateway:** `openclaw gateway status` should show running
2. **Check cron:** `openclaw cron list` should show your job
3. **Check server:** `curl http://localhost:19999/status` should show `connected_clients: 1`
4. **Test manually:** `openclaw cron run YOUR_JOB_ID --force` should run the job

## Where Screenshots Are Saved

By default, Chrome downloads go to `C:\Users\Asus\Downloads\` with filenames like:
- `proofsnap-2026-02-08T12-30-45-123Z.png`

You can change this in Chrome settings or modify the extension code.

---

## Troubleshooting

### "Gateway not reachable" error
The gateway isn't running. Start it with:
```powershell
openclaw gateway run --bind loopback --port 18789 --force
```

### "No ProofSnap extension connected" error
1. Make sure Chrome is running with the OpenClaw browser profile
2. Check `chrome://extensions/` - is ProofSnap enabled?
3. Reload the extension
4. Verify with `curl http://localhost:19999/status`

### Job runs but no screenshots appear
1. Check the extension console (right-click extension icon → Inspect)
2. Make sure you're on an X.com page when the trigger fires
3. Check Chrome downloads folder

### Job doesn't detect new posts
1. Check `state.json` - are posts being added to `capturedPosts`?
2. Try clearing `capturedPosts: []` to rescan
3. Check if the account has new posts visible at `/with_replies`

### Watchdog says "openclaw not found"
OpenClaw isn't in your PATH. Use the full path:
```powershell
$env:Path += ";C:\Users\Asus\AppData\Roaming\npm"
```
Or edit the watchdog script to use the full path to `openclaw.cmd`.
