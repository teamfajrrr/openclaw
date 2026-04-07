# Deploying OpenClaw on Render

## Architecture

```
Browser → Render HTTPS → nginx (port 8080) → OpenClaw Gateway (port 18789, loopback)
```

The `coollabsio/openclaw:latest` Docker image bundles nginx + the OpenClaw gateway. Nginx handles HTTP health checks, basic auth, gateway token injection, and WebSocket proxying. The gateway runs on loopback behind nginx.

A thin wrapper Dockerfile (`Dockerfile.render`) patches the coollabsio image to fix cloud-specific issues.

## Quick Start

### 1. Create Web Service on Render

- **New** → **Web Service** → **Build and deploy from a Git repository**
- Repo: `teamfajrrr/openclaw` (or your fork)
- Branch: `main`
- Runtime: **Docker**
- Dockerfile Path: **`Dockerfile.render`**
- Region: Singapore (or your preference)
- Instance Type: **Standard** (2 GB RAM minimum — will OOM on free/starter)
- Docker Command: **leave empty**

### 2. Environment Variables

| Key | Value | Required |
|-----|-------|----------|
| `OPENCLAW_GATEWAY_TOKEN` | Strong random string (e.g. `openssl rand -hex 16`) | Yes |
| `AUTH_USERNAME` | Your username for nginx basic auth | Yes |
| `AUTH_PASSWORD` | Your password for nginx basic auth | Yes |
| `PORT` | `8080` | Yes |
| `OPENCLAW_ALLOWED_ORIGINS` | `https://your-service.onrender.com` | Yes |
| `OPENROUTER_API_KEY` | Your OpenRouter API key | Pick one |
| `ANTHROPIC_API_KEY` | Your Anthropic API key | Pick one |
| `OPENAI_API_KEY` | Your OpenAI API key | Pick one |

### 3. Deploy

Click **Create Web Service**. First build takes ~5 minutes (pulling the 880MB base image). Subsequent deploys use cached layers and take ~2 minutes.

### 4. Access

Go to `https://your-service.onrender.com/`:
1. Enter nginx basic auth credentials (`AUTH_USERNAME` / `AUTH_PASSWORD`)
2. Enter your `OPENCLAW_GATEWAY_TOKEN` in the "Gateway Token" field
3. Click **Connect**

## What Dockerfile.render Fixes

### 1. Browser Sidecar (nginx crash)

The coollabsio image's nginx config references `proxy_pass http://browser:3000/` for a Chrome sidecar container used in Docker Compose. On Render (single container), this hostname doesn't exist and nginx crashes at startup.

**Fix:** `sed` replaces `proxy_pass http://browser:3000/;` with `return 404;` in the entrypoint at build time.

### 2. Trusted Proxies

The gateway rejects connections from nginx because it doesn't recognize it as a trusted proxy.

**Fix:** Injects `gateway.trustedProxies: ["127.0.0.1", "10.0.0.0/8"]` and `gateway.allowRealIpFallback: true` into the configure script.

### 3. Device Pairing (Control UI)

The gateway enforces device pairing for non-local Control UI connections. There's no way to complete pairing from a cloud-only deployment.

**Fix:** Sets `gateway.controlUi.dangerouslyDisableDeviceAuth: true` — the official break-glass config that skips device identity/pairing for operator connections.

### 4. Persistent Disk Stale Config

`configure.js` only sets defaults when creating new config. The persistent disk (`/data`) keeps old config across redeploys, so new config changes don't take effect.

**Fix:** A runtime patch script (`render-patch-config.js`) runs AFTER `configure.js` every startup to ensure critical config values are applied regardless of what's on disk.

## Troubleshooting

### 502 Bad Gateway
- Check Render logs for startup errors
- Verify instance type is Standard (2 GB RAM)
- Check if nginx started: look for `[entrypoint] starting nginx on port 8080...` without `[emerg]` errors after it

### "origin not allowed"
- Add your Render URL to `OPENCLAW_ALLOWED_ORIGINS` env var
- Must include `https://` prefix

### "pairing required"
- Verify `dangerouslyDisableDeviceAuth` is in the config: run `grep dangerously /data/.openclaw/openclaw.json` in the Render shell
- If missing, delete `/data/.openclaw` and redeploy to regenerate config

### "gateway token missing" / "token_mismatch"
- Ensure `OPENCLAW_GATEWAY_TOKEN` env var is set
- Enter the exact same token in the Control UI's "Gateway Token" field

### OOM Crash
- Upgrade to Standard instance (2 GB RAM)
- Set `NODE_OPTIONS=--max-old-space-size=1536` env var if needed
