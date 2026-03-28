# Mouse Game - Deployment Plan

This document covers how to deploy the Mouse game to production (`mouse.charlesdesign.xyz`) and testing (Cloudflare quick tunnel), sharing the existing Cloudflare named tunnel on this machine.

---

## Current Infrastructure

| Component | Details |
|-----------|---------|
| Machine | `bob-web-dev` (35.226.20.146, Ubuntu) |
| Cloudflared | 2026.3.0, running as bare processes (not systemd) |
| Named tunnel | `richarddesign-tunnel` (ID: `f476f3d5-61ec-4130-aea1-9404760690f9`) |
| Config file | `/home/ubuntu/.cloudflared/config.yml` |
| Existing site | `game.richarddesign.org` → `localhost:3000` (Alien Cannon Defense, Node.js) |
| Existing processes | Quick tunnel on PID 23610 (likely stale), named tunnel on PID 34640 |

### Existing Ingress Rules
```yaml
tunnel: richarddesign-tunnel
credentials-file: /home/ubuntu/.cloudflared/f476f3d5-61ec-4130-aea1-9404760690f9.json

ingress:
  - hostname: game.richarddesign.org
    service: http://localhost:3000
  - service: http_status:404
```

---

## Architecture

The Mouse game is a single `index.html` file. It needs a minimal static file server. We'll use a lightweight approach:

```
Browser → Cloudflare Tunnel → localhost:8080 → Static file server → /mnt/data/bob/charles/mouse/index.html
```

### Port Allocation
| Service | Port | Domain |
|---------|------|--------|
| Alien Cannon Defense (existing) | 3000 | game.richarddesign.org |
| Mouse Game (new) | 8080 | mouse.charlesdesign.xyz |

---

## Step 1: Static File Server

The game is a single `index.html` with no dependencies. Use Python's built-in HTTP server or a simple Node.js server.

### Option A: Python (simplest, no install needed)

```bash
cd /mnt/data/bob/charles/mouse
python3 -m http.server 8080 &
```

### Option B: Node.js one-liner (if you prefer consistency with the existing setup)

Create `/mnt/data/bob/charles/mouse/server.js`:

```js
const http = require('http');
const fs = require('fs');
const path = require('path');

const PORT = 8080;
const FILE = path.join(__dirname, 'index.html');

http.createServer((req, res) => {
  // Serve index.html for all routes
  fs.readFile(FILE, (err, data) => {
    if (err) {
      res.writeHead(500);
      res.end('Error loading game');
      return;
    }
    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(data);
  });
}).listen(PORT, () => {
  console.log(`Mouse game server running on http://localhost:${PORT}`);
});
```

### Verification

```bash
curl -s http://localhost:8080 | head -5
```

Should return the opening lines of `index.html`.

---

## Step 2: Testing via Quick Tunnel

Before setting up the production domain, test with a Cloudflare quick tunnel. Quick tunnels generate a random `*.trycloudflare.com` URL — no DNS config needed.

### Start Quick Tunnel

```bash
cloudflared tunnel --url http://localhost:8080
```

This will output a URL like:
```
https://random-words.trycloudflare.com
```

### Test Checklist
- [ ] Open the URL in a browser
- [ ] Game loads and renders correctly
- [ ] Controls work (arrow keys, WASD, SPACE, P, M)
- [ ] Audio plays after first interaction
- [ ] No mixed-content or CORS errors in console

### Stop Quick Tunnel

```bash
# Find and kill the quick tunnel process
kill $(pgrep -f "cloudflared tunnel --url http://localhost:8080")
```

**Important:** Quick tunnels are temporary. The URL changes every time you restart. Do not use for production.

---

## Step 3: DNS Setup in Cloudflare Dashboard

Before the named tunnel can serve `mouse.charlesdesign.xyz`, you need a CNAME record pointing to the tunnel.

### 3.1 — Add CNAME Record

1. Log in to [Cloudflare Dashboard](https://dash.cloudflare.com)
2. Select the **charlesdesign.xyz** zone
3. Go to **DNS** → **Records**
4. Add a new record:

| Field | Value |
|-------|-------|
| Type | CNAME |
| Name | `mouse` |
| Target | `f476f3d5-61ec-4130-aea1-9404760690f9.cfargotunnel.com` |
| Proxy status | Proxied (orange cloud) |
| TTL | Auto |

**Note:** The target uses the tunnel ID. This is the same tunnel serving `game.richarddesign.org` — multiple domains can share one tunnel via ingress rules.

### 3.2 — Verify DNS Propagation

```bash
dig mouse.charlesdesign.xyz CNAME +short
```

Should return: `f476f3d5-61ec-4130-aea1-9404760690f9.cfargotunnel.com.`

---

## Step 4: Update Named Tunnel Config

Add the Mouse game as a new ingress rule to the existing tunnel config. The existing `game.richarddesign.org` route is preserved.

### 4.1 — Edit Config

Edit `/home/ubuntu/.cloudflared/config.yml`:

```yaml
tunnel: richarddesign-tunnel
credentials-file: /home/ubuntu/.cloudflared/f476f3d5-61ec-4130-aea1-9404760690f9.json

ingress:
  - hostname: game.richarddesign.org
    service: http://localhost:3000
  - hostname: mouse.charlesdesign.xyz
    service: http://localhost:8080
  - service: http_status:404
```

**Critical rules:**
- New hostname entries go **above** the catch-all `- service: http_status:404` (which must always be last)
- Each hostname routes to a different port
- The catch-all returns 404 for any unmatched request

### 4.2 — Validate Config

```bash
cloudflared tunnel ingress validate
```

Should output: `OK`

### 4.3 — Restart Named Tunnel

The named tunnel must be restarted to pick up config changes.

```bash
# Find and kill the existing named tunnel process
kill $(pgrep -f "cloudflared tunnel run richarddesign-tunnel")

# Wait a moment for cleanup
sleep 2

# Start the tunnel again
nohup cloudflared tunnel run richarddesign-tunnel > /tmp/cloudflared-tunnel.log 2>&1 &

# Verify it's running
pgrep -f "cloudflared tunnel run richarddesign-tunnel"
```

**Impact:** Restarting the tunnel causes ~2-5 seconds of downtime for ALL domains on this tunnel (including `game.richarddesign.org`). Plan accordingly.

### 4.4 — Verify Production

```bash
curl -s https://mouse.charlesdesign.xyz | head -5
```

Should return the opening lines of `index.html`.

Also verify the existing site still works:
```bash
curl -s https://game.richarddesign.org | head -5
```

---

## Step 5: Process Management

Currently, all processes (web servers, tunnel) run as bare processes that won't survive a reboot. This section sets up persistence.

### 5.1 — Systemd Service for Mouse Game Server

Create `/etc/systemd/system/mouse-game.service`:

```ini
[Unit]
Description=Mouse Game Static Server
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/mnt/data/bob/charles/mouse
ExecStart=/usr/bin/python3 -m http.server 8080
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable mouse-game
sudo systemctl start mouse-game
sudo systemctl status mouse-game
```

### 5.2 — Systemd Service for Named Tunnel

Create `/etc/systemd/system/cloudflared-tunnel.service`:

```ini
[Unit]
Description=Cloudflare Named Tunnel (richarddesign-tunnel)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=ubuntu
ExecStart=/usr/local/bin/cloudflared tunnel --config /home/ubuntu/.cloudflared/config.yml run richarddesign-tunnel
Restart=always
RestartSec=5
Environment=HOME=/home/ubuntu

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
# First, kill the existing bare tunnel process
kill $(pgrep -f "cloudflared tunnel run richarddesign-tunnel")
sleep 2

sudo systemctl daemon-reload
sudo systemctl enable cloudflared-tunnel
sudo systemctl start cloudflared-tunnel
sudo systemctl status cloudflared-tunnel
```

### 5.3 — Clean Up Stale Processes

The old quick tunnel (PID 23610, running since Mar 26) appears stale. Kill it:

```bash
kill 23610
```

---

## Step 6: Deployment Workflow

After the initial setup, deploying updates is simple:

### Update Game Code
```bash
cd /mnt/data/bob/charles/mouse
git pull origin master
# Server serves files directly from disk — no restart needed for Python http.server
# If using Node.js server.js, restart: sudo systemctl restart mouse-game
```

### Test Before Production

1. **Quick tunnel smoke test:**
   ```bash
   cloudflared tunnel --url http://localhost:8080
   # Open the URL in browser, verify game loads and basic gameplay works
   # Ctrl+C to stop when done
   ```

2. **Run the relevant regression checklist from `TESTING_PLAN.md`:**
   - After a Phase 1-only change → run the Phase 1 regression checklist
   - After a Phase 2 change → run Phase 1 + Phase 2 regression checklists
   - For any release that touches multiple phases or is a major update → run the full regression (all 111 tests)
   - Do **not** deploy to production until the relevant regression checklist passes

### Rollback

**Game-content rollback** (bad `index.html` or game logic):
```bash
cd /mnt/data/bob/charles/mouse
git log --oneline -5          # find the last known-good commit
git revert HEAD               # create a revert commit (preserves history)
# No server restart needed — files served directly from disk
```

**Full rollback** (bad deploy includes server, config, or deployment changes):
```bash
cd /mnt/data/bob/charles/mouse
git log --oneline -10         # find the last known-good commit
git revert HEAD               # or revert multiple: git revert HEAD~2..HEAD
# If server.js changed: sudo systemctl restart mouse-game
# If config.yml changed: restore from backup and restart tunnel
#   cp /home/ubuntu/.cloudflared/config.yml.bak /home/ubuntu/.cloudflared/config.yml
#   sudo systemctl restart cloudflared-tunnel
```

**Config backup practice:** Before editing `config.yml`, always back it up:
```bash
cp /home/ubuntu/.cloudflared/config.yml /home/ubuntu/.cloudflared/config.yml.bak
```

---

## Troubleshooting

### Game not loading at mouse.charlesdesign.xyz
1. Check DNS: `dig mouse.charlesdesign.xyz CNAME +short`
2. Check tunnel: `sudo systemctl status cloudflared-tunnel`
3. Check server: `sudo systemctl status mouse-game` and `curl localhost:8080`
4. Check config: `cloudflared tunnel ingress validate`
5. Check logs: `journalctl -u cloudflared-tunnel -f` and `journalctl -u mouse-game -f`

### Existing site (game.richarddesign.org) broken after config change
1. Verify ingress order in config.yml — catch-all must be last
2. Check the Node.js server: `curl localhost:3000`
3. Check tunnel logs: `journalctl -u cloudflared-tunnel -f`

### Quick tunnel not working
1. Check port: `curl localhost:8080`
2. Check for port conflict: `lsof -i :8080`
3. Try a different port

### Tunnel shows "connection reset"
1. Verify the local server is running on the correct port
2. Check firewall: `sudo ufw status`

---

## Summary

| Step | Action | Status |
|------|--------|--------|
| 1 | Start static file server on port 8080 | Pending |
| 2 | Test via quick tunnel | Pending |
| 3 | Add CNAME for mouse.charlesdesign.xyz in Cloudflare DNS | Pending |
| 4 | Add ingress rule to config.yml, restart tunnel | Pending |
| 5 | Set up systemd services for persistence | Pending |
| 6 | Deployment workflow documented | Done |

### Domain Map

```
Cloudflare Named Tunnel: richarddesign-tunnel
│
├── richarddesign.org zone (separate Cloudflare zone)
│   └── game.richarddesign.org  → localhost:3000 (Alien Cannon Defense)
│
├── charlesdesign.xyz zone
│   └── mouse.charlesdesign.xyz → localhost:8080 (Mouse Game)     ← NEW
│
└── *.trycloudflare.com         → quick tunnel (testing only, ephemeral)
```

**Note:** Both domains share the same named tunnel via ingress rules, but their DNS records live in separate Cloudflare zones. The CNAME for `mouse.charlesdesign.xyz` is added in the `charlesdesign.xyz` zone. The existing `game.richarddesign.org` CNAME lives in the `richarddesign.org` zone.

### Files Modified

| File | Change |
|------|--------|
| `/home/ubuntu/.cloudflared/config.yml` | Add mouse.charlesdesign.xyz ingress rule |
| `/etc/systemd/system/mouse-game.service` | New — static file server |
| `/etc/systemd/system/cloudflared-tunnel.service` | New — managed tunnel service |
