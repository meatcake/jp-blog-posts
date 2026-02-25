# Complete Production Guide: OpenClaw on DigitalOcean Marketplace

**Read time: 15 minutes**

**Category: Tutorial**

---

## Introduction

This article explains how to set up a production environment using the [OpenClaw 1-Click Droplet from DigitalOcean Marketplace](https://marketplace.digitalocean.com/apps/openclaw). The Marketplace version comes with OpenClaw pre-installed and includes dedicated helper scripts for quick setup.

However, there are **three critical challenges** that must be resolved for production use. This article provides detailed solutions for these issues.

### What You'll Learn

- Creating an OpenClaw Marketplace Droplet on DigitalOcean
- AI model selection and configuration (Anthropic Sonnet recommended)
- Marketplace-specific setup procedures (`/opt/openclaw-cli.sh`, etc.)
- Solutions to three critical challenges
- Security hardening (Web UI authentication)

### Prerequisites

- DigitalOcean account ([signup with $200 free credit](https://m.do.co/c/signup))
- Anthropic API key (obtain at [https://console.anthropic.com/](https://console.anthropic.com/))
- Basic Linux command knowledge
- Time required: ~40 minutes

### Cost

- **Droplet: $12/month** (2 vCPU, 2GB RAM, 60GB SSD) - Recommended
- **Anthropic API: Usage-based**
  - Claude 3.5 Sonnet: $3/MTok (input), $15/MTok (output)
  - Claude 3 Opus: $15/MTok (input), $75/MTok (output)

**Recommendation: Use Sonnet** (reasons explained below)

---

## Step 1: Create Marketplace Droplet

### 1.1 Create OpenClaw Droplet

1. Visit [DigitalOcean Marketplace - OpenClaw](https://marketplace.digitalocean.com/apps/openclaw)
2. Click **Create OpenClaw Droplet**
3. Select:
   - **Region:** Closest to you (Singapore for Asia)
   - **Droplet Size:** 
     - **Recommended:** Basic → Regular → **$12/mo** (2 vCPU, 2GB RAM, 60GB SSD)
     - Minimum: $6/mo (1 vCPU, 1GB RAM) - Requires swap
   - **Authentication:** SSH key (recommended)
   - **Hostname:** Descriptive name (e.g., `openclaw-production`)
4. Click **Create Droplet**
5. Note the IP address

### 1.2 Connect via SSH

```bash
ssh root@YOUR_DROPLET_IP
```

On first connection, you'll see a welcome message:

```
Welcome to OpenClaw on DigitalOcean!

To get started:
1. Configure your AI model: /opt/openclaw-cli.sh config
2. Check status: /opt/status-openclaw.sh
3. View logs: journalctl -u openclaw -f

Documentation: https://docs.openclaw.ai
Support: https://discord.com/invite/clawd
```

---

## Step 2: AI Model Configuration (Important!)

### 2.1 Why We Recommend Anthropic Sonnet

OpenClaw supports multiple AI model providers, but we **strongly recommend Anthropic Claude 3.5 Sonnet**:

**Reasons to choose Sonnet:**

1. **Rate Limiting Issues**
   - **Claude 3 Opus:** High performance but strict rate limits cause frequent errors in production
   - **Claude 3.5 Sonnet:** Reasonable rate limits with practical operation speed
   
2. **Cost-Performance**
   - Sonnet: $3/MTok (input), $15/MTok (output)
   - Opus: $15/MTok (input), $75/MTok (output) - 5x more expensive
   
3. **Performance**
   - Sonnet 3.5 is sufficiently powerful for most tasks
   - Opus's performance gains don't justify the rate limits and cost

**Conclusion: We strongly recommend using Sonnet for production environments.**

### 2.2 Obtain Anthropic API Key

1. Visit [Anthropic Console](https://console.anthropic.com/)
2. Create account / Sign in
3. Create a new key in the **API Keys** section
4. Copy the key (starts with `sk-ant-api03-...`)

### 2.3 Edit Environment Variables File

In the Marketplace version, environment variables are managed in `/opt/openclaw.env`:

```bash
# Edit environment variables file
nano /opt/openclaw.env
```

Find this line and set your API key:

```bash
# For Anthropic Claude (recommended):
ANTHROPIC_API_KEY=sk-ant-api03-YOUR_KEY_HERE
```

**Save:** `Ctrl+O` → `Enter` → `Ctrl+X`

### 2.4 Configure Model Settings (Set to Sonnet)

```bash
# Set default model to Sonnet
/opt/openclaw-cli.sh config set agents.defaults.model.primary "anthropic/claude-sonnet-3-5"

# Verify configuration
/opt/openclaw-cli.sh config get agents.defaults.model
```

**Expected output:**
```json
{
  "primary": "anthropic/claude-sonnet-3-5"
}
```

### 2.5 Restart OpenClaw

```bash
/opt/restart-openclaw.sh
```

Or:

```bash
systemctl restart openclaw
```

### 2.6 Verify Operation

```bash
# Check status
/opt/status-openclaw.sh

# Check logs for operation
journalctl -u openclaw -f
```

No errors? Success! ✅

---

## Step 3: Understanding the Marketplace Version

### 3.1 Important Files and Scripts

The Marketplace version includes dedicated helper scripts different from standard installations:

#### `/opt/openclaw-cli.sh`
Wrapper for all OpenClaw commands. Runs as the `openclaw` user.

```bash
# Standard installation
openclaw status

# Marketplace version
/opt/openclaw-cli.sh status
```

#### `/opt/openclaw.env`
Environment variables configuration file (API keys, Gateway settings, etc.)

```bash
# Manage API keys and settings here
nano /opt/openclaw.env

# Restart required after changes
systemctl restart openclaw
```

#### Other Helper Scripts

```bash
/opt/openclaw-tui.sh           # Launch TUI (Text User Interface)
/opt/restart-openclaw.sh       # Restart OpenClaw
/opt/status-openclaw.sh        # Check status
/opt/update-openclaw.sh        # Update OpenClaw
/opt/setup-openclaw-domain.sh  # Configure custom domain
```

### 3.2 Users and Permissions

In the Marketplace version, OpenClaw runs as a dedicated `openclaw` user:

```bash
# Switch to openclaw user
su - openclaw

# Configuration file location
ls -la /home/openclaw/.openclaw/

# Return to root
exit
```

**Important:** Always run OpenClaw commands via `/opt/openclaw-cli.sh`.

---

## Step 4: Three Critical Challenges and Solutions

### Challenge 1: Sandbox Environment Can't Access Internet 🚫

**Symptom:**
Agent cannot access external APIs (weather, news, search, etc.)

**Cause:**
By default, OpenClaw runs agents in a sandboxed environment (Docker container) for security. This sandbox has restricted network access.

**Solution:**

```bash
# Configure to run on host
/opt/openclaw-cli.sh config set tools.exec.host gateway

# Disable interactive confirmation
/opt/openclaw-cli.sh config set tools.exec.ask off

# Enable full security mode
/opt/openclaw-cli.sh config set tools.exec.security full

# Restart OpenClaw
/opt/restart-openclaw.sh
```

**Configuration explanation:**

- `tools.exec.host gateway`: Run on gateway host instead of sandbox
- `tools.exec.ask off`: Disable confirmation before execution (needed for automation)
- `tools.exec.security full`: Enable full security checks

**Verification:**

```bash
# Check configuration
/opt/openclaw-cli.sh config get tools.exec

# Expected output:
{
  "host": "gateway",
  "ask": "off",
  "security": "full"
}
```

**Test:**
Ask your agent via Telegram: "What's the weather today?" If it can access external APIs, it works!

---

### Challenge 2: Agent Can't Access Browser 🌐

**Symptom:**
Agent cannot scrape web pages or perform browser automation.

**Cause:**
Chrome or Chromium is not installed or not properly configured.

**Solution:**

#### 2.1 Install Google Chrome

```bash
# Download and install Google Chrome
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
dpkg -i google-chrome-stable_current_amd64.deb

# Fix dependencies
apt --fix-broken install -y

# Verify
google-chrome --version
```

**Expected output:**
```
Google Chrome 145.0.7632.116
```

#### 2.2 Configure OpenClaw Browser

```bash
# Enable browser
/opt/openclaw-cli.sh config set browser.enabled true

# Set Chrome path
/opt/openclaw-cli.sh config set browser.executablePath /usr/bin/google-chrome

# Enable headless mode
/opt/openclaw-cli.sh config set browser.headless true

# Disable sandbox (required)
/opt/openclaw-cli.sh config set browser.noSandbox true

# Restart OpenClaw
/opt/restart-openclaw.sh
```

#### 2.3 Verify Operation

```bash
# Check browser status
/opt/openclaw-cli.sh browser status

# Test browser startup
/opt/openclaw-cli.sh browser start

# Simple test
/opt/openclaw-cli.sh browser open https://example.com
/opt/openclaw-cli.sh browser screenshot
```

**Expected output:**

```
profile: openclaw
enabled: true
running: true
cdpPort: 18800
cdpUrl: http://127.0.0.1:18800
browser: unknown
detectedBrowser: custom
detectedPath: /usr/bin/google-chrome
profileColor: #FF4500
```

#### 2.4 Troubleshooting

If errors occur:

```bash
# Check for missing system libraries
ldd /usr/bin/google-chrome | grep "not found"

# Install missing libraries
apt install -y \
  libatk1.0-0 \
  libatk-bridge2.0-0 \
  libcups2 \
  libxkbcommon0 \
  libxcomposite1 \
  libxdamage1 \
  libxrandr2 \
  libgbm1 \
  libpango-1.0-0 \
  libasound2
```

---

### Challenge 3: No Authentication on Web UI 🔓

**Symptom:**
The DigitalOcean Marketplace version allows Web UI access via public IP with no authentication by default. This is a **serious security risk**.

**Cause:**
Caddy reverse proxy is configured as:

```caddy
YOUR_IP {
    tls { ... }
    reverse_proxy localhost:18789
}
```

Anyone can access `https://YOUR_IP/` without authentication.

**Solution: Add Basic Authentication**

#### 3.1 Generate Password Hash

```bash
# Hash your password
caddy hash-password

# Enter password when prompted
# Copy the output hash
```

**Example:**
```
Enter password: ●●●●●●●●
Confirm password: ●●●●●●●●
$2a$14$JuViLfdKLPjLUabGupYi.p5uGV0O.FXt67nQ04bqdoBiIO0GSRFi
```

Note this hash.

#### 3.2 Edit Caddyfile

```bash
# Edit Caddyfile
nano /etc/caddy/Caddyfile
```

**Before:**

```caddy
YOUR_IP {
    tls {
        issuer acme {
            dir https://acme-v02.api.letsencrypt.org/directory
            profile shortlived
        }
    }
    reverse_proxy localhost:18789
    header X-DO-MARKETPLACE "openclaw"
}
```

**After:**

```caddy
YOUR_IP {
    tls {
        issuer acme {
            dir https://acme-v02.api.letsencrypt.org/directory
            profile shortlived
        }
    }
    
    # Add Basic authentication
    basicauth {
        # Username: admin (change to any name you prefer)
        admin $2a$14$JuViLfdKLPjLUabGupYi.p5uGV0O.FXt67nQ04bqdoBiIO0GSRFi
    }
    
    reverse_proxy localhost:18789
    header X-DO-MARKETPLACE "openclaw"
}
```

**Important:** Replace `admin` with your preferred username. Use the hash you generated.

**Save:** `Ctrl+O` → `Enter` → `Ctrl+X`

#### 3.3 Restart Caddy

```bash
# Reload configuration
systemctl reload caddy

# Check status
systemctl status caddy
```

**No errors? Success!**

#### 3.4 Verify

1. Access `https://YOUR_IP/` in browser
2. You should see username and password prompt
3. Enter credentials to log in

**Success!** 🎉 Your Web UI is now protected with authentication.

---

## Step 5: Connect Channels

### 5.1 Create and Connect Telegram Bot

#### Create Telegram Bot

1. Open [@BotFather](https://t.me/botfather) on Telegram
2. Send `/newbot` command
3. Enter bot name (e.g., `My OpenClaw Bot`)
4. Enter username (e.g., `my_openclaw_bot`)
5. Copy bot token (starts with `1234567890:ABCdefGHI...`)

#### Configure in OpenClaw

```bash
# Add token to environment variables file
nano /opt/openclaw.env
```

Find this line and set your token:

```bash
# Telegram Bot Token
TELEGRAM_BOT_TOKEN=1234567890:ABCdefGHIjklMNOpqrsTUVwxyz...
```

**Save:** `Ctrl+O` → `Enter` → `Ctrl+X`

```bash
# Restart OpenClaw
/opt/restart-openclaw.sh
```

#### Pairing

```bash
# Get pairing code
/opt/openclaw-cli.sh pairing list telegram
```

**Example output:**
```
Pairing Code: ABCD1234
Expires: 2026-02-25 18:00:00
```

1. Open your bot on Telegram
2. Send the code (e.g., `ABCD1234`)
3. Approve in OpenClaw:

```bash
/opt/openclaw-cli.sh pairing approve telegram ABCD1234
```

**Success!** Your Telegram bot is connected. Send a message to test.

### 5.2 Other Channels

- **WhatsApp:** `/opt/openclaw-cli.sh channels login whatsapp`
- **Discord:** Add `DISCORD_BOT_TOKEN` to `/opt/openclaw.env`
- **Slack:** Add `SLACK_BOT_TOKEN` and `SLACK_APP_TOKEN` to `/opt/openclaw.env`

See [official documentation](https://docs.openclaw.ai/channels) for details.

---

## Step 6: Advanced Configuration (Optional)

### 6.1 Custom Domain Setup

Use your own domain (e.g., `bot.example.com`) instead of public IP:

```bash
# Run setup script
/opt/setup-openclaw-domain.sh
```

Follow the prompts:

1. **Enter domain name:** `bot.example.com`
2. **Enter email address:** `admin@example.com` (for Let's Encrypt notifications)

The script automatically:
- Updates Caddyfile
- Obtains SSL certificate from Let's Encrypt
- Restarts OpenClaw

**Important:** Set up DNS A record pointing your domain to the Droplet IP.

### 6.2 Security Hardening: IP Restriction

Allow access only from specific IP addresses:

```bash
nano /etc/caddy/Caddyfile
```

```caddy
YOUR_IP_OR_DOMAIN {
    tls { ... }
    
    # IP address restriction
    @blocked not remote_ip YOUR_HOME_IP YOUR_OFFICE_IP
    respond @blocked "Access Denied" 403
    
    basicauth { ... }
    reverse_proxy localhost:18789
}
```

### 6.3 Security Hardening: Tailscale VPN (Most Secure)

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# Configure OpenClaw
/opt/openclaw-cli.sh config set gateway.bind tailnet

# Restart OpenClaw
/opt/restart-openclaw.sh
```

Now only accessible from your Tailscale network.

---

## Step 7: Monitoring and Maintenance

### 7.1 Check Logs

```bash
# OpenClaw logs
journalctl -u openclaw -f

# Caddy logs
journalctl -u caddy -f

# System resources
htop
```

### 7.2 Regular Maintenance

```bash
# System update
apt update && apt upgrade -y

# OpenClaw update
/opt/update-openclaw.sh

# Restart (if needed)
/opt/restart-openclaw.sh
```

### 7.3 Backup

Important files:

```bash
# OpenClaw environment variables
/opt/openclaw.env

# OpenClaw configuration (as openclaw user)
su - openclaw
tar -czf ~/openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw/
exit

# Download backup
scp root@YOUR_IP:/home/openclaw/openclaw-backup-*.tar.gz ./
```

---

## Troubleshooting

### Issue 1: "Permission denied" Error

**Symptom:** Error when executing commands with `/opt/openclaw-cli.sh`

**Solution:**

```bash
# Verify you're root user
whoami  # Output: root

# Verify script has execute permission
ls -l /opt/openclaw-cli.sh

# Verify openclaw user exists
id openclaw
```

### Issue 2: API Rate Limit Errors

**Symptom:** Frequent `Rate limit exceeded` errors

**Solution:**

```bash
# Check current model
/opt/openclaw-cli.sh config get agents.defaults.model

# Change from Opus to Sonnet (recommended)
/opt/openclaw-cli.sh config set agents.defaults.model.primary "anthropic/claude-sonnet-3-5"

# Restart OpenClaw
/opt/restart-openclaw.sh
```

**Opus has strict rate limits. Use Sonnet for production.**

### Issue 3: Out of Memory (OOM)

**Symptom:** OpenClaw crashes frequently

**Solution:**

```bash
# Check memory usage
free -h

# Add swap (for 1GB RAM)
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# Verify
free -h
```

### Issue 4: Browser Won't Start

**Symptom:** `Failed to start Chrome CDP`

**Solution:**

```bash
# Test Chrome manually
google-chrome --headless --no-sandbox --disable-gpu --dump-dom https://example.com

# If errors, install system libraries
apt install -y \
  libatk1.0-0 libatk-bridge2.0-0 libcups2 \
  libxkbcommon0 libxcomposite1 libxdamage1 \
  libxrandr2 libgbm1 libpango-1.0-0 libasound2

# Check configuration
/opt/openclaw-cli.sh browser status
```

---

## Summary

You now have OpenClaw Marketplace running in production on DigitalOcean!

### What We Achieved ✅

- ✅ Created OpenClaw Marketplace Droplet
- ✅ Configured Anthropic Claude Sonnet (rate limit mitigation)
- ✅ Enabled internet access from sandbox environment
- ✅ Enabled browser automation
- ✅ Added authentication to Web UI for security
- ✅ Connected messaging channels

### Key Points 🎯

1. **Use Sonnet**
   - Opus has strict rate limits, impractical for production
   - Sonnet is sufficiently powerful with good cost-performance

2. **Understand Marketplace Version**
   - Use `/opt/openclaw-cli.sh`
   - Manage environment variables in `/opt/openclaw.env`
   - Runs as `openclaw` user

3. **Don't Forget Security**
   - Add Basic authentication to Web UI
   - Use Tailscale VPN if possible
   - Regular backups

### Next Steps 🚀

- Read [OpenClaw official documentation](https://docs.openclaw.ai/)
- Add [Skills](https://docs.openclaw.ai/skills) to extend your agent
- Set up [Heartbeat](https://docs.openclaw.ai/automation/heartbeat) for periodic tasks
- Configure [Memory management](https://docs.openclaw.ai/memory) for agent persistence

### Support 💬

- **Official Discord:** https://discord.com/invite/clawd
- **GitHub:** https://github.com/openclaw/openclaw
- **Documentation:** https://docs.openclaw.ai/
- **Marketplace:** https://marketplace.digitalocean.com/apps/openclaw

---

**About the Author**

This article was written based on real-world experience building and operating OpenClaw on DigitalOcean Marketplace.

**About OpenClaw**

OpenClaw is an open-source framework that connects AI assistants to messaging platforms, enabling automation, memory management, and skill extensions.

---

**Related Articles**

- [OpenClaw Core Concepts](/)
- [Creating Telegram Bots](/)
- [Advanced Browser Automation Techniques](/)
- [Anthropic Claude API Guide](/)
