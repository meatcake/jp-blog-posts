# OpenClaw Production Guide on DigitalOcean - As of Feb 25, 2026

**Read time: 12 minutes**

---

## Introduction

This guide explains how to set up a working OpenClaw environment using the [OpenClaw 1-Click Droplet from DigitalOcean Marketplace](https://marketplace.digitalocean.com/apps/openclaw).

When I set this up, I ran into three problems. This guide shows you how to solve them, plus some practical use cases.

### What You'll Learn

- Setting up OpenClaw on DigitalOcean Marketplace
- Connecting a Telegram bot
- Choosing an AI model (Anthropic Sonnet recommended)
- Solving three common problems
- Practical examples like automated blog translation

### What You Need

- DigitalOcean account ([signup with $200 credit](https://m.do.co/c/signup))
- Anthropic API key ([https://console.anthropic.com/](https://console.anthropic.com/))
- Time: ~40 minutes

### Cost

- **Droplet:** $12/month (2 vCPU, 2GB RAM)
- **Anthropic API:** Usage-based
  - Sonnet: $3/MTok input, $15/MTok output
  - Opus: $15/MTok input, $75/MTok output (5x more expensive)

Sonnet is good enough. More on that later.

---

## Step 1: Create a Droplet

### 1.1 Create OpenClaw Droplet

1. Visit [DigitalOcean Marketplace - OpenClaw](https://marketplace.digitalocean.com/apps/openclaw)
2. Click **Create OpenClaw Droplet**
3. Settings:
   - **Region:** Closest to you (Singapore for Asia)
   - **Droplet Size:** $12/mo (2 vCPU, 2GB RAM) recommended
   - **Authentication:** SSH key
   - **Hostname:** Something like `openclaw-production`
4. Click **Create Droplet**
5. Note the IP address

### 1.2 SSH Connect

```bash
ssh root@YOUR_DROPLET_IP
```

You'll see a welcome message on first connection.

---

## Step 2: Connect Telegram Bot

Setting up Telegram first makes testing easier later.

### 2.1 Create Telegram Bot

1. Open [@BotFather](https://t.me/botfather) on Telegram
2. Send `/newbot`
3. Enter bot name (e.g., `My OpenClaw Bot`)
4. Enter username (e.g., `my_openclaw_bot`)
5. Copy the token (looks like `1234567890:ABC...`)

### 2.2 Configure OpenClaw

```bash
# Edit environment variables
nano /opt/openclaw.env
```

Find this line and add your token:

```bash
# Telegram Bot Token
TELEGRAM_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz...
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

```bash
# Restart
/opt/restart-openclaw.sh
```

### 2.3 Pairing

```bash
# Get pairing code
/opt/openclaw-cli.sh pairing list telegram
```

Output:
```
Pairing Code: ABCD1234
Expires: 2026-02-25 18:00:00
```

1. Open your bot on Telegram
2. Send the code (`ABCD1234`)
3. Approve:

```bash
/opt/openclaw-cli.sh pairing approve telegram ABCD1234
```

Send a message to your bot. If it responds, you're good.

---

## Step 3: Configure AI Model

### 3.1 Why Sonnet?

OpenClaw supports multiple AI models, but I recommend **Anthropic Claude 3.5 Sonnet**.

**Reasons:**

1. **Rate Limits**
   - Opus: High performance but hits rate limits quickly
   - Sonnet: Reasonable limits
   
2. **Cost**
   - Sonnet: $3/$15 (input/output)
   - Opus: $15/$75 (5x more)
   
3. **Performance**
   - Sonnet is good enough

Opus is hard to use in production.

### 3.2 Get Anthropic API Key

1. Create account at [Anthropic Console](https://console.anthropic.com/)
2. Create new key in **API Keys**
3. Copy the key (starts with `sk-ant-api03-...`)

### 3.3 Set Environment Variable

```bash
nano /opt/openclaw.env
```

Find this line and add your API key:

```bash
# For Anthropic Claude (recommended):
ANTHROPIC_API_KEY=sk-ant-api03-YOUR_KEY_HERE
```

Save and exit.

### 3.4 Set to Sonnet

```bash
# Set default model to Sonnet
/opt/openclaw-cli.sh config set agents.defaults.model.primary "anthropic/claude-sonnet-3-5"

# Verify
/opt/openclaw-cli.sh config get agents.defaults.model
```

```bash
# Restart
/opt/restart-openclaw.sh
```

### 3.5 Test

Send "Hello" to your Telegram bot. If it responds, it works.

---

## Step 4: Understanding the Marketplace Version

### 4.1 Important Files

The Marketplace version has dedicated helper scripts.

#### `/opt/openclaw-cli.sh`
Wrapper for all OpenClaw commands.

```bash
# Standard installation
openclaw status

# Marketplace version
/opt/openclaw-cli.sh status
```

#### `/opt/openclaw.env`
Environment variables configuration (API keys, etc.).

```bash
nano /opt/openclaw.env  # Edit
systemctl restart openclaw  # Apply
```

#### Other Scripts

```bash
/opt/openclaw-tui.sh           # Launch TUI
/opt/restart-openclaw.sh       # Restart
/opt/status-openclaw.sh        # Check status
/opt/update-openclaw.sh        # Update
/opt/setup-openclaw-domain.sh  # Domain setup
```

### 4.2 Users and Permissions

OpenClaw runs as the `openclaw` user.

```bash
# Switch to openclaw user
su - openclaw

# Config file location
ls -la ~/.openclaw/

# Back to root
exit
```

**Important:** Run commands via `/opt/openclaw-cli.sh`.

---

## Step 5: Three Problems and Solutions

### Problem 1: Sandbox Can't Access Internet

**Symptom:**
Can't access external services like weather APIs.

**Cause:**
By default, runs in a sandbox (Docker container) with restricted network access.

**Solution:**

```bash
# Run on host instead
/opt/openclaw-cli.sh config set tools.exec.host gateway
/opt/openclaw-cli.sh config set tools.exec.ask off
/opt/openclaw-cli.sh config set tools.exec.security full

# Restart
/opt/restart-openclaw.sh
```

**Verify:**

```bash
/opt/openclaw-cli.sh config get tools.exec
```

Try asking on Telegram: "What's the weather today?"

---

### Problem 2: Browser Not Working

**Symptom:**
Can't scrape web pages.

**Cause:**
Chrome isn't installed.

**Solution:**

#### 2.1 Install Chrome

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
dpkg -i google-chrome-stable_current_amd64.deb
apt --fix-broken install -y

# Verify
google-chrome --version
```

#### 2.2 Configure OpenClaw

```bash
/opt/openclaw-cli.sh config set browser.enabled true
/opt/openclaw-cli.sh config set browser.executablePath /usr/bin/google-chrome
/opt/openclaw-cli.sh config set browser.headless true
/opt/openclaw-cli.sh config set browser.noSandbox true

# Restart
/opt/restart-openclaw.sh
```

#### 2.3 Verify

```bash
/opt/openclaw-cli.sh browser status
/opt/openclaw-cli.sh browser start
```

If it says `running: true`, you're good.

---

### Problem 3: No Web UI Authentication

**Symptom:**
Anyone can access `https://YOUR_IP/`.

**Cause:**
Caddy config has no authentication.

**Solution:**

#### 3.1 Hash Password

```bash
caddy hash-password
```

Enter password, copy the hash.

#### 3.2 Edit Caddyfile

```bash
nano /etc/caddy/Caddyfile
```

Add `basicauth` section:

```caddy
YOUR_IP {
    tls {
        issuer acme {
            dir https://acme-v02.api.letsencrypt.org/directory
            profile shortlived
        }
    }
    
    # Add authentication
    basicauth {
        admin $2a$14$JuVi...(generated hash)
    }
    
    reverse_proxy localhost:18789
    header X-DO-MARKETPLACE "openclaw"
}
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

#### 3.3 Restart Caddy

```bash
systemctl reload caddy
systemctl status caddy
```

#### 3.4 Verify

Open `https://YOUR_IP/` in browser. You should see username/password prompt.

---

## Step 6: Advanced Settings

### 6.1 Custom Domain

If you want to use your own domain:

```bash
/opt/setup-openclaw-domain.sh
```

Enter domain name and email, it'll configure everything.

### 6.2 IP Restrictions

Allow only specific IPs:

```bash
nano /etc/caddy/Caddyfile
```

```caddy
@blocked not remote_ip YOUR_HOME_IP YOUR_OFFICE_IP
respond @blocked "Access Denied" 403
```

### 6.3 Tailscale VPN (Most Secure)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

/opt/openclaw-cli.sh config set gateway.bind tailnet
/opt/restart-openclaw.sh
```

---

## Step 7: Practical Example - Automated Blog Translation

Here's what I actually use this for.

### 7.1 What I Want

1. Check English blog every morning at 9 AM
2. If there's a new post, fetch the content
3. Translate to Japanese (editor-adapted)
4. Auto-create GitHub PR

### 7.2 Setup

#### Create GitHub Personal Access Token

1. [GitHub Settings → Personal access tokens](https://github.com/settings/tokens)
2. **Generate new token (classic)**
3. Scope: Check `repo`
4. Copy token (starts with `ghp_...`)

#### Configure Token

```bash
nano /opt/openclaw.env
```

Add:

```bash
GITHUB_TOKEN=ghp_YourTokenHere
```

Save and restart.

### 7.3 Create Cron Job

```bash
/opt/openclaw-cli.sh cron add \
  --name "blog-translation" \
  --cron "0 9 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --announce \
  --message "Check https://www.zipteam.com/blog/ with browser. If there are new articles, fetch content, translate to Japanese, and create PR at https://github.com/meatcake/jp-blog-posts. Track state in workspace/zipteam-blog-state.json. Reply HEARTBEAT_OK if no new posts."
```

### 7.4 Create State File

```bash
su - openclaw
cd ~/.openclaw/workspace

cat > zipteam-blog-state.json << 'EOF'
{
  "lastCheck": null,
  "seenPosts": []
}
EOF

exit
```

### 7.5 Results

With this setup:

- 3 articles auto-translated
- 7 PRs auto-created
- Manual work: 5 hours/week → 30 minutes (95% reduction)

**Cost:**
- Droplet: $12/month
- API: ~$1.65/week
- **Total: ~$19/month**

**At $50/hour, ROI is about 84x.**

---

## Step 8: Monitoring and Maintenance

### 8.1 Check Logs

```bash
# OpenClaw logs
journalctl -u openclaw -f

# Caddy logs
journalctl -u caddy -f

# System resources
htop
```

### 8.2 Regular Maintenance

```bash
# System update
apt update && apt upgrade -y

# OpenClaw update
/opt/update-openclaw.sh
```

### 8.3 Backup

```bash
# Important files
/opt/openclaw.env
~/.openclaw/

# Backup
su - openclaw
tar -czf ~/openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw/
exit

# Download
scp root@YOUR_IP:/home/openclaw/openclaw-backup-*.tar.gz ./
```

---

## Troubleshooting

### API Rate Limit Errors

**Symptom:** Frequent `Rate limit exceeded`

**Solution:**

```bash
# Switch to Sonnet
/opt/openclaw-cli.sh config set agents.defaults.model.primary "anthropic/claude-sonnet-3-5"
/opt/restart-openclaw.sh
```

### Out of Memory

**Symptom:** OpenClaw crashes

**Solution:**

```bash
# Add swap
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

free -h  # Verify
```

### Browser Won't Start

**Symptom:** `Failed to start Chrome CDP`

**Solution:**

```bash
# Test Chrome
google-chrome --headless --no-sandbox --disable-gpu --dump-dom https://example.com

# If errors, install libraries
apt install -y \
  libatk1.0-0 libatk-bridge2.0-0 libcups2 \
  libxkbcommon0 libxcomposite1 libxdamage1 \
  libxrandr2 libgbm1 libpango-1.0-0 libasound2
```

---

## Summary

What we did:

- Created OpenClaw Marketplace Droplet
- Connected Telegram bot
- Configured Anthropic Claude Sonnet
- Solved three problems
- Implemented automated blog translation

### Next Steps

- [OpenClaw official docs](https://docs.openclaw.ai/)
- [Skills](https://docs.openclaw.ai/skills) to extend functionality
- [Heartbeat](https://docs.openclaw.ai/automation/heartbeat) for periodic tasks

### Support

- Discord: https://discord.com/invite/clawd
- GitHub: https://github.com/openclaw/openclaw
- Docs: https://docs.openclaw.ai/

---

**Information as of February 25, 2026.**

OpenClaw and DigitalOcean Marketplace specs may change. Check official documentation for latest info.
