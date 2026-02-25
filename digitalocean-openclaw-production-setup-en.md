# Complete Guide to Production OpenClaw Setup on DigitalOcean

**Read time: 10 minutes**

**Category: Tutorial**

---

## Introduction

OpenClaw is a powerful framework that enables you to connect AI assistants to messaging platforms like Telegram, WhatsApp, and Discord. This article provides a complete guide to deploying OpenClaw on DigitalOcean for production use, including solutions to three common challenges.

### What You'll Learn

- Initial OpenClaw setup on DigitalOcean
- Solving sandbox internet access issues
- Configuring browser automation
- Adding authentication to the Web UI for enhanced security

### Prerequisites

- DigitalOcean account ([signup with $200 free credit](https://m.do.co/c/signup))
- SSH key pair (or password authentication)
- Basic Linux command knowledge
- Time required: ~30 minutes

### Cost

- **$6/month** (1 vCPU, 1GB RAM, 25GB SSD)
- Or **$4/month** (with reserved pricing)

---

## Step 1: Create a Droplet

### 1.1 Basic Configuration

1. Log into [DigitalOcean](https://cloud.digitalocean.com/)
2. Click **Create → Droplets**
3. Select:
   - **Region:** Closest to you (Singapore or San Francisco for Asia)
   - **Image:** Ubuntu 24.04 LTS
   - **Size:** Basic → Regular → **$6/mo** (1 vCPU, 1GB RAM, 25GB SSD)
   - **Authentication:** SSH key (recommended) or password
4. Click **Create Droplet**
5. Note the IP address

### 1.2 Connect via SSH

```bash
ssh root@YOUR_DROPLET_IP
```

---

## Step 2: System Preparation

### 2.1 System Update

```bash
apt update && apt upgrade -y
```

### 2.2 Add Swap (Essential for 1GB RAM)

With only 1GB RAM, add swap to prevent OOM (Out of Memory) errors:

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# Verify
free -h
```

### 2.3 Install Node.js 22

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs

# Verify
node --version  # v22.x.x
npm --version   # 10.x.x
```

---

## Step 3: Install OpenClaw

### 3.1 OpenClaw Installation

```bash
curl -fsSL https://openclaw.ai/install.sh | bash

# Verify
openclaw --version
```

### 3.2 Onboarding

```bash
openclaw onboard --install-daemon
```

The wizard will guide you through:

- **Model authentication** (API keys or OAuth)
- **Channel setup** (Telegram, WhatsApp, Discord, etc.)
- **Gateway token** (auto-generated)
- **Daemon installation** (systemd)

### 3.3 Verify Installation

```bash
# Check status
openclaw status

# Check service
systemctl --user status openclaw-gateway.service

# View logs
journalctl --user -u openclaw-gateway.service -f
```

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
openclaw config set tools.exec.host gateway

# Disable interactive confirmation
openclaw config set tools.exec.ask off

# Enable full security mode
openclaw config set tools.exec.security full

# Restart Gateway
openclaw gateway restart
```

**Configuration Explanation:**

- `tools.exec.host gateway`: Run on gateway host instead of sandbox
- `tools.exec.ask off`: Disable confirmation before command execution
- `tools.exec.security full`: Enable full security checks

**Verification:**

```bash
# Check configuration
openclaw config get tools.exec

# Test external access via agent
# (In Telegram or other channel) Ask: "What's the weather today?"
```

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

#### 2.2 Configure OpenClaw Browser

```bash
# Set browser path
openclaw config set browser.enabled true
openclaw config set browser.executablePath /usr/bin/google-chrome
openclaw config set browser.headless true
openclaw config set browser.noSandbox true

# Restart Gateway
openclaw gateway restart
```

#### 2.3 Verify Operation

```bash
# Check browser status
openclaw browser status

# Test browser startup
openclaw browser start

# Simple test
openclaw browser open https://example.com
openclaw browser screenshot
```

**Expected Output:**

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

If errors occur, check for missing libraries:

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
The DigitalOcean template allows Web UI access via public IP address by default with no authentication. This is a **serious security risk**.

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
        admin <paste-your-generated-hash-here>
    }
    
    reverse_proxy localhost:18789
    header X-DO-MARKETPLACE "openclaw"
}
```

**Save:** `Ctrl+O` → `Enter` → `Ctrl+X`

#### 3.3 Restart Caddy

```bash
# Reload configuration
systemctl reload caddy

# Check status
systemctl status caddy
```

#### 3.4 Verify

1. Access `https://YOUR_IP/` in browser
2. You should see username and password prompt
3. Enter credentials to log in

**Success!** 🎉 Your Web UI is now protected with authentication.

---

## Step 5: Connect Channels

### Connect Telegram

```bash
# View pairing code
openclaw pairing list telegram

# Send code to Telegram bot and approve
openclaw pairing approve telegram <CODE>
```

### Connect WhatsApp

```bash
# Login with QR code
openclaw channels login whatsapp

# Scan QR code
```

For other channels (Discord, Slack, etc.), see [official documentation](https://docs.openclaw.ai/channels).

---

## Step 6: Advanced Configuration (Optional)

### Enhanced Security Options

#### Option 1: IP Restriction

Allow access only from specific IP addresses:

```caddy
YOUR_IP {
    tls { ... }
    
    # IP address restriction
    @blocked not remote_ip YOUR_HOME_IP YOUR_OFFICE_IP
    respond @blocked "Access Denied" 403
    
    basicauth { ... }
    reverse_proxy localhost:18789
}
```

#### Option 2: Tailscale VPN

Most secure method:

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# Configure OpenClaw
openclaw config set gateway.bind tailnet
openclaw gateway restart
```

Now only accessible from your Tailscale network.

---

## Step 7: Monitoring and Maintenance

### Check Logs

```bash
# Gateway logs
journalctl --user -u openclaw-gateway.service -f

# Caddy logs
journalctl -u caddy -f

# System resources
htop
```

### Regular Maintenance

```bash
# System update
apt update && apt upgrade -y

# OpenClaw update
npm update -g openclaw

# Restart (if needed)
openclaw gateway restart
```

### Backup

Important files:

```bash
# OpenClaw configuration
~/.openclaw/openclaw.json
~/.openclaw/agents/main/agent.json

# Workspace
~/.openclaw/workspace/

# Backup command example
tar -czf openclaw-backup-$(date +%Y%m%d).tar.gz ~/.openclaw/
```

---

## Troubleshooting

### Issue: Out of Memory (OOM)

**Symptom:** Gateway crashes frequently

**Solution:**

```bash
# Check swap
free -h

# Check memory usage
ps aux --sort=-%mem | head -10

# Upgrade to larger Droplet
# Or use API-based models
```

### Issue: Browser Won't Start

**Symptom:** `Failed to start Chrome CDP`

**Solution:**

```bash
# Test Chrome manually
google-chrome --headless --no-sandbox --disable-gpu --dump-dom https://example.com

# Check system libraries
ldd /usr/bin/google-chrome | grep "not found"

# Check configuration
openclaw browser status
```

### Issue: Caddy Configuration Error

**Symptom:** Authentication not working

**Solution:**

```bash
# Test Caddy configuration
caddy validate --config /etc/caddy/Caddyfile

# Check logs
journalctl -u caddy -f

# Manual reload
systemctl reload caddy
```

---

## Summary

You now have OpenClaw running in production on DigitalOcean!

### What We Achieved

✅ OpenClaw installation and configuration  
✅ Internet access from sandbox environment  
✅ Browser automation enabled  
✅ Web UI secured with authentication  
✅ Messaging channels connected  

### Next Steps

- Read [OpenClaw official documentation](https://docs.openclaw.ai/)
- Add [Skills](https://docs.openclaw.ai/skills) to extend your agent
- Set up [Heartbeat](https://docs.openclaw.ai/automation/heartbeat) for periodic tasks
- Configure [Memory management](https://docs.openclaw.ai/memory) for agent persistence

### Cost Optimization

- **Cheaper option:** [Hetzner](https://docs.openclaw.ai/install/hetzner) (€4/month)
- **Free option:** [Oracle Cloud](https://docs.openclaw.ai/platforms/oracle) (ARM, Always Free Tier)

### Support

- **Official Discord:** https://discord.com/invite/clawd
- **GitHub:** https://github.com/openclaw/openclaw
- **Documentation:** https://docs.openclaw.ai/

---

**About the Author**

This article was written based on real-world experience deploying OpenClaw on DigitalOcean for production use.

**About OpenClaw**

OpenClaw is an open-source framework that connects AI assistants to messaging platforms, enabling automation, memory management, and skill extensions.

---

**Related Articles**

- [OpenClaw Core Concepts](/)
- [Creating Telegram Bots](/)
- [Advanced Browser Automation Techniques](/)
