# OpenClaw on Fly.io: Headless Setup for Reading Personalized X "For You" Feed

**Version:** 1.0  
**Date:** March 2026  
**Author:** Grok (for dα/ωi / devDawi)  
**Target Audience:** Any LLM (Claude, Grok, GPT, etc.) assisting the user to implement this setup

---

## 1. Project Overview & Purpose

**Goal:**  
Deploy a reliable, headless OpenClaw instance on Fly.io that can **read the authenticated "For You" feed** on X.com (Twitter) without getting blocked by anti-bot systems.

**Why this matters:**

- X's "For You" feed is personalized and requires a logged-in session.
- Normal headless browsers (Playwright, Puppeteer, Selenium) get fingerprinted and blocked instantly.
- We need **C++-level fingerprint spoofing** + residential proxy + persistent cookies.

**Chosen Stack (Why this is the best & safest in 2026):**

- **OpenClaw** on Fly.io (already running, headless)
- **Camofox-browser** (`@askjo/camofox-browser` or `jo-inc/camofox-browser`) → Camoufox (Firefox fork with real C++ spoofing of Canvas, WebGL, AudioContext, WebRTC, hardwareConcurrency, etc.)
- **Residential proxy** (Bright Data / Smartproxy / Oxylabs / IPRoyal) – sticky or rotating
- **Cookie-based login** (never store password)
- **Custom skill** `read-x-for-you-feed` – thin wrapper that reuses Camofox tools
- Persistent storage via Fly Volume for profiles + cookies

**Security Notes (CRITICAL):**

- NEVER install `mayuqi-crypto/stealth-browser` (ClawHub flags it as suspicious + known credential-stealing patterns).
- Only use audited Camofox repos (jo-inc or official forks).
- Always review any skill before installing (ClawHub had 341+ malicious skills in early 2026).
- Use Fly secrets for all keys/proxies.
- Cookie export happens **once locally** on your real browser.

---

## 2. Prerequisites

- Existing headless OpenClaw deployment on Fly.io with persistent `/data` volume mounted.
- Fly CLI installed and authenticated.
- SSH/SFTP access to the Fly machine (`fly ssh console` and `fly ssh sftp`).
- Residential proxy account (Bright Data recommended – get a sticky residential endpoint).
- Local Chrome browser logged into X.com (for cookie export).
- Chrome extension: "Get cookies.txt LOCALLY" (or any Netscape-format exporter).

---

## 3. High-Level Architecture

```
Fly.io App (Docker)
├── OpenClaw Gateway
├── Camofox-browser server (runs on port 9377, headless)
├── /data/
│   ├── profiles/x-default/          ← persistent Camofox profile
│   ├── profiles/x-default/cookies.txt
│   ├── skills/read-x-for-you-feed/SKILL.md
│   └── extensions/camofox-browser/
├── Residential Proxy (http://user:pass@proxy:port)
└── Fly Volume (persistent)
```

---

## 4. Step-by-Step Implementation Guide

### Step 4.1 – Prepare Fly.io Machine

```bash
# 1. Enter the machine
fly ssh console -a YOUR-APP-NAME
su - node   # or whatever user runs OpenClaw

# 2. Update OpenClaw
openclaw update

# 3. Create directories
mkdir -p /data/profiles/x-default
mkdir -p /data/extensions
mkdir -p /data/skills/read-x-for-you-feed

# 4. Add secrets (do this from your local terminal)
fly secrets set PROXY_STRING="http://USER:PASS@residential-proxy-provider:port" \
               CAMOFOX_API_KEY="$(openssl rand -hex 32)" \
               ANTHROPIC_API_KEY=sk-...
```

### Step 4.2 – Install & Start Camofox-browser

```bash
# From inside Fly machine
cd /data/extensions
git clone https://github.com/jo-inc/camofox-browser.git
cd camofox-browser
npm install

# Create config
cat > config.json << EOF
{
  "port": 9377,
  "headless": true,
  "proxy": "\${PROXY_STRING}",
  "geoip": true,
  "persistProfiles": true
}
EOF
```

**Auto-start Camofox** (add to your Dockerfile ENTRYPOINT or startup script):

```bash
# In Dockerfile or entrypoint.sh – add before starting OpenClaw gateway
cd /data/extensions/camofox-browser
CAMOFOX_PORT=9377 CAMOFOX_HEADLESS=true npm start &
```

Redeploy:

```bash
fly deploy
```

### Step 4.3 – Import X Cookies (One-Time)

**On your local machine (logged into X):**

1. Install "Get cookies.txt LOCALLY" extension.
2. Go to https://x.com/home
3. Export cookies → save as `x-cookies.txt` (Netscape format).

**Transfer to Fly:**

```bash
fly ssh sftp -a YOUR-APP-NAME
put x-cookies.txt /data/profiles/x-default/cookies.txt
```

### Step 4.4 – Create the Custom Skill `read-x-for-you-feed`

On the Fly machine:

```bash
cd /data/skills/read-x-for-you-feed

cat > SKILL.md << 'EOF'
---
name: read-x-for-you-feed
description: Reads and returns a clean summary of the authenticated "For You" feed on X.com. Use for "check my X feed", "what's happening on my For You page", daily digest, etc.
version: 1.1
author: Grok + devDawi
---

# Read X For You Feed (Camofox-powered)

## When to trigger
- User says "read my X feed", "check For You", "summarize my timeline", "what's new on X"

## Prerequisites (checked by agent)
- camofox-browser plugin installed and running on port 9377
- X cookies imported into profile "x-default"

## Exact Runbook (agent MUST follow)

1. Ensure tab exists:
   camofox_create_tab --profile x-default --name x-feed-tab

2. Navigate to home (For You is default):
   camofox_navigate --tab x-feed-tab --url https://x.com/home

3. Wait for feed + import cookies if needed:
   camofox_snapshot --tab x-feed-tab --wait 8000
   camofox_import_cookies --profile x-default --file /data/profiles/x-default/cookies.txt  # fallback

4. Scroll to load real posts (For You algorithm):
   camofox_scroll --tab x-feed-tab --direction down --amount 1500
   sleep 4
   camofox_scroll --tab x-feed-tab --direction down --amount 1000

5. Extract posts:
   camofox_snapshot --tab x-feed-tab --selector "article[data-testid='tweet']" --output json --save /tmp/x-feed.json

6. Parse & summarize (use LLM):
   Return in this exact format:
   **X For You Feed – [current time]**
   • @username: "first 120 chars..." (likes: X | reposts: Y)
   • ...
   Total posts loaded: N
   Top trend in feed: ...

EOF
```

OpenClaw skills usually hot-reload. If not, restart gateway.

### Step 4.5 – Test Everything

Inside OpenClaw chat (Telegram/Discord/web UI):

```
Use the skill read-x-for-you-feed and give me a 5-bullet summary of my current For You feed.
```

Or simply:

```
read my x for you feed
```

**Expected behavior:**

- Camofox starts headless tab
- Loads X with your cookies + proxy
- Scrolls, extracts posts
- Returns clean summary

---

## 5. Maintenance & Best Practices

- **Proxy rotation:** Update `PROXY_STRING` secret every 3–7 days + `fly restart`.
- **Cookie refresh:** Re-export cookies every 30 days or when login fails.
- **Daily digest:** Create a cron skill that runs the skill at 8 AM and DMs you.
- **Monitoring:** `fly logs | grep camofox`
- **Backup:** `fly ssh sftp` → download `/data/profiles/x-default/` regularly.
- **Scaling:** If you need multiple accounts → create more profiles (`x-work`, `x-personal`).

---

## 6. Troubleshooting Checklist

1. **Camofox not starting** → Check `fly logs`, ensure npm start is backgrounded.
2. **Blocked on X** → Verify proxy is residential + sticky. Try new proxy.
3. **Cookies not loading** → Re-import manually: `camofox_import_cookies ...`
4. **Skill not found** → `openclaw skills list` and check path.
5. **Fly Volume issues** → Confirm volume is mounted to `/data` in fly.toml.
6. **High CPU** → Set Camofox idle shutdown in config.json.

---

## 7. Appendix – Useful Commands

```bash
# Inside Fly
camofox_list_tabs
camofox_screenshot --tab x-feed-tab --save /tmp/test.png
fly ssh sftp   # for file transfer
```

**Proxy providers (2026):**

- Bright Data (best quality)
- Smartproxy
- Oxylabs
- IPRoyal (cheapest residential)

**You are now done.**  
Once this file is read, the LLM should be able to guide the user command-by-command, debug errors, and even write missing scripts.

**Final note for LLM reading this:**  
You have full permission to modify any command slightly for the exact user’s Fly setup. Always prioritize safety: never suggest installing unverified ClawHub skills. If anything fails, ask the user to paste exact error logs.

End of document.
