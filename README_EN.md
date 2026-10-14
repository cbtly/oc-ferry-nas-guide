# OC Ferry on NAS: From Source to 24/7

> Field notes from a real migration on 2026-10-15: OC Ferry 0.3.0 moved from Windows to a NAS (J3160 / fnOS), running as a systemd service with 9 free models available and a custom Chinese WebUI.

---

## Contents

- [1. Introduction](#1-introduction)
- [2. Prerequisites](#2-prerequisites)
- [3. Migration Steps](#3-migration-steps)
- [4. Pitfall Guide](#4-pitfall-guide)
- [5. Advanced](#5-advanced)
- [6. Reference](#6-reference)

---

## 1. Introduction

### What is OC Ferry

OC Ferry is a bridge service that exposes [OpenCode](https://github.com/sst/opencode)'s free models through an OpenAI-compatible API. Any client supporting a custom base URL (WorkBuddy, Hermes, Codex, etc.) can connect.

Key features:

- OpenAI-compatible `/v1` relay endpoint (chat completions)
- Automatic model capability probing at startup (including tool-call support)
- API key authentication
- Built-in Chinese WebUI (`/ui`): status monitoring, model list, one-click probing, proxy toggle

### Why run it on NAS

| | Windows | NAS |
|---|---|---|
| Uptime | PC must stay on | 24/7 always-on |
| LAN access | Dies when PC sleeps | Always reachable |
| Multi-device | Configure per machine | One address for all |
| Power | Tens of watts | J3160 sips ~15W |

Deploy once, benefit long-term. Any device on your LAN just needs one address.

---

## 2. Prerequisites

### 2.1 NAS requirements

- SSH access (root or sudo user)
- systemd (fnOS / Debian / Ubuntu all ship it)
- At least 2GB free disk (OpenCode binary ~100MB + Node runtime)

Environment used in this guide:

```
Device: J3160 mini PC (4GB RAM)
OS: fnOS
Hostname: hiyori
IP: 192.168.31.53
Deploy dir: /vol1/1000/oc-ferry
Data dir: /vol1/1000/oc-ferry/data
Port: 41980
```

### 2.2 Node.js v22+ (critical)

**This is the most important prerequisite.** NAS systems often ship Node v18, under which OC Ferry's `undici` dependency fails to load and the service exits immediately (see [Pitfalls](#41-node-18-vs-22)).

Check your version:

```bash
node -v
# If it prints v18.x, you must upgrade
```

Install Node v22 binary (x64 example; use arm64 for ARM machines):

```bash
cd /tmp
curl -sL https://nodejs.org/dist/v22.22.0/node-v22.22.0-linux-x64.tar.xz -o node.tar.xz
tar -xf node.tar.xz
cp -r node-v22.22.0-linux-x64/bin/* /usr/local/bin/
cp -r node-v22.22.0-linux-x64/lib/* /usr/local/lib/ 2>/dev/null
cp -r node-v22.22.0-linux-x64/include/* /usr/local/include/ 2>/dev/null
cp -r node-v22.22.0-linux-x64/share/* /usr/local/share/ 2>/dev/null
/usr/local/bin/node -v
# Should print v22.22.0
```

> Don't overwrite the system's `/usr/bin/node`. Installing v22 under `/usr/local` is enough; we'll use an absolute path in the systemd unit.

### 2.3 Prepare the source (on your dev machine)

Assuming you have a working OC Ferry checkout with `node_modules` installed:

```bash
cd ~/workspace/oc-ferry-build
tar --exclude='RuiC-oc-ferry/node_modules' \
    --exclude='RuiC-oc-ferry/release' \
    --exclude='RuiC-oc-ferry/.git' \
    -czf oc-ferry-nas.tar.gz RuiC-oc-ferry
ls -lh oc-ferry-nas.tar.gz
# ~3-7MB without node_modules
```

> Skip `node_modules` in the tarball and run `npm install --production` on the NAS to avoid architecture mismatches (e.g., Windows dev machine vs. Linux NAS).

---

## 3. Migration Steps

### 3.1 Transfer to NAS

Use whatever works for you (scp, rsync, file sharing, etc.):

```bash
scp oc-ferry-nas.tar.gz root@192.168.31.53:/vol1/1000/oc-ferry/
```

### 3.2 Extract and install dependencies

```bash
ssh root@192.168.31.53
mkdir -p /vol1/1000/oc-ferry
cd /vol1/1000/oc-ferry
tar -xzf oc-ferry-nas.tar.gz
cd RuiC-oc-ferry
npm install --production
```

### 3.3 Create the systemd service

```bash
cat > /etc/systemd/system/oc-ferry.service << 'EOF'
[Unit]
Description=OC Ferry - OpenCode free models bridge
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/vol1/1000/oc-ferry/RuiC-oc-ferry
Environment=FERRY_HOST=0.0.0.0
Environment=FERRY_PORT=41980
Environment=FERRY_DATA_DIR=/vol1/1000/oc-ferry/data
Environment=FERRY_NO_SYNC=1
ExecStart=/usr/local/bin/node src/main.js
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

Key settings:

| Setting | Purpose |
|---------|---------|
| `FERRY_HOST=0.0.0.0` | Listen on all interfaces for LAN access (default `127.0.0.1` is localhost-only) |
| `FERRY_PORT=41980` | Service port |
| `FERRY_DATA_DIR` | Data dir (API key, probe cache) |
| `FERRY_NO_SYNC=1` | Skip WorkBuddy sync (not needed on NAS) |
| `ExecStart=/usr/local/bin/node` | **Must point to Node v22**, not the system's v18 |
| `Restart=always` | Auto-restart on crash |

### 3.4 Start the service

```bash
systemctl daemon-reload
systemctl enable oc-ferry
systemctl start oc-ferry
sleep 10
systemctl status oc-ferry --no-pager | head -12
```

You should see `Active: active (running)`.

### 3.5 Confirm it's ready

The first start downloads the OpenCode binary automatically (a few minutes). Check the logs:

```bash
journalctl -u oc-ferry --no-pager | grep -i "ready\|opencode" | tail -5
```

Look for:

```
OC Ferry ready at http://127.0.0.1:41980/v1; 9 free models; probe cache hit 0, probing 9
```

Or query the health endpoint directly:

```bash
curl -s http://127.0.0.1:41980/health \
  -H "Authorization: Bearer $(cat /vol1/1000/oc-ferry/data/api-key)" | head -c 300
```

`phase: ready` means you're good.

### 3.6 Get the API key and open the WebUI

```bash
cat /vol1/1000/oc-ferry/data/api-key
```

Open in your browser:

```
http://192.168.31.53:41980/ui
```

Paste the key to see the model list, per-model probe status, one-click re-probe, and proxy toggle.

Relay endpoint for clients:

```
http://192.168.31.53:41980/v1
```

---

## 4. Pitfall Guide

### 4.1 Node 18 vs 22

**Symptom:**

```bash
journalctl -u oc-ferry --no-pager | tail -20
# undici fails to load, Node.js v18.20.4, service exits with status 1/FAILURE
```

**Root cause:**

The NAS ships Node v18, under which OC Ferry's `undici` dependency cannot load. The service crashes on start; systemd restarts won't help.

**Fix:**

Install Node v22 under `/usr/local` per [2.2](#22-nodejs-v22-critical), and make sure the unit uses the absolute path:

```ini
# Wrong
ExecStart=/usr/bin/node src/main.js
# Correct
ExecStart=/usr/local/bin/node src/main.js
```

Verify:

```bash
grep ExecStart /etc/systemd/system/oc-ferry.service
/usr/local/bin/node -v   # should print v22.x
```

### 4.2 daemon-reload after editing the unit

**Symptom:**

You fixed the `ExecStart` path and restarted, but logs still show the old Node version.

**Root cause:**

systemd caches unit files; `systemctl restart` alone doesn't re-read them.

**Fix:**

```bash
systemctl daemon-reload
systemctl restart oc-ferry
```

Run `daemon-reload` after every edit to `/etc/systemd/system/oc-ferry.service`.

### 4.3 FERRY_HOST binding

**Symptom:**

`curl http://127.0.0.1:41980/health` works on the NAS, but `http://192.168.31.53:41980/ui` is unreachable from other LAN devices.

**Root cause:**

The default `FERRY_HOST=127.0.0.1` binds loopback only.

**Fix:**

```ini
Environment=FERRY_HOST=0.0.0.0
```

Then `daemon-reload` + `restart`.

> Security note: `0.0.0.0` exposes the service to your whole LAN. The API key is the only auth — keep it safe and don't expose the port publicly.

### 4.4 API key location

**Symptom:**

Can't find the key; WebUI says unauthorized.

**Root cause:**

The key isn't in any config file — it's randomly generated on first start and stored in the data dir.

**Fix:**

```bash
cat $FERRY_DATA_DIR/api-key
# i.e. /vol1/1000/oc-ferry/data/api-key
```

### 4.5 First run downloads OpenCode

**Symptom:**

Service shows running, but `/health` returns `phase: starting` and there's no "OC Ferry ready" in logs.

**Root cause:**

First start downloads the OpenCode binary (~100MB). The service stays in `starting` until it finishes.

**Fix:**

Wait a few minutes and watch:

```bash
journalctl -u oc-ferry -f
```

`OC Ferry ready at ...` means done. If it hangs, check the NAS network connectivity.

---

## 5. Advanced

### 5.1 textMode: fixing tool-call incompatibility for some models

**Background:**

OC Ferry constrains model output via OpenCode's `format: { type: 'json_schema' }` mechanism. Some upstream endpoints reject this, failing probes with:

```
only 'auto' is supported for 'tool_choice'
```

The prober then mislabels the model as chat-only, even though the model itself supports tool calls (`opencode/muse-spark-1.3-contributor-free` hit exactly this).

**Fix (textMode fallback):**

When the specific `tool_choice` incompatibility is detected, the probe retries in plain-text mode: the `format` field is omitted from the request, and the bridge parses the JSON envelope from text using its existing `decode()`.

The WebUI shows such models as available (text mode) with full tool-call support. The `textMode` flag persists in the probe cache across restarts.

### 5.2 Reasoning-effort auto-downgrade

**Background:**

Free tiers usually cap reasoning effort. Muse Spark 1.3's free tier maxes out at `high`; a client sending `reasoning_effort: max` gets a hard 400:

```
Requested reasoning effort is not available for this model
```

...and falls back to another model — poor UX.

**Fix:**

The bridge auto-downgrades: when the requested level exceeds what the model offers, it silently uses the highest available level at or below the request (`max` → `high`). Level ordering:

```
none < minimal < low < medium < high < xhigh < max
```

Special rules:

- Requesting `none` (disable reasoning) on a model that can't disable it still returns 400 — silently enabling reasoning would be wrong
- The downgrade is recorded on the request object for log tracing

### 5.3 Connecting clients

OC Ferry is a standard OpenAI-compatible endpoint. You need three things:

| Item | Value |
|------|-------|
| Base URL | `http://192.168.31.53:41980/v1` |
| API Key | Contents of `$FERRY_DATA_DIR/api-key` |
| Model ID | e.g. `opencode/muse-spark-1.3-contributor-free` |

List all models:

```bash
curl -s http://192.168.31.53:41980/v1/models \
  -H "Authorization: Bearer YOUR_API_KEY" | python3 -m json.tool
```

---

## 6. Reference

### 6.1 Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `FERRY_HOST` | `127.0.0.1` | Bind address; use `0.0.0.0` for LAN access |
| `FERRY_PORT` | `41980` | Service port |
| `FERRY_DATA_DIR` | `./data` | Data dir (API key, cache) |
| `FERRY_NO_SYNC` | `0` | Set to `1` to skip WorkBuddy sync |

### 6.2 API endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /health` | Health check (auth required): phase, model count, versions |
| `GET /v1/models` | Model list (OpenAI format) |
| `POST /v1/chat/completions` | Chat completions (OpenAI format) |
| `GET /ui` | Chinese WebUI dashboard |
| `GET /ui/api/status` | WebUI status API (Bearer key required) |
| `POST /ui/api/probe` | WebUI trigger model probing (Bearer key required) |

### 6.3 File locations

| Path | Description |
|------|-------------|
| `/vol1/1000/oc-ferry/RuiC-oc-ferry` | Application directory |
| `/vol1/1000/oc-ferry/data/api-key` | API key |
| `/vol1/1000/oc-ferry/data/` | Probe cache and runtime data |
| `/etc/systemd/system/oc-ferry.service` | systemd unit file |
| `/usr/local/bin/node` | Node v22 binary |

### 6.4 Common ops commands

```bash
# Status
systemctl status oc-ferry --no-pager | head -12

# Live logs
journalctl -u oc-ferry -f

# Restart
systemctl restart oc-ferry

# Enable on boot
systemctl enable oc-ferry

# Health check
curl -s http://127.0.0.1:41980/health \
  -H "Authorization: Bearer $(cat /vol1/1000/oc-ferry/data/api-key)"
```

---

*中文版见 [README.md](README.md)*
