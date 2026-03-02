# OpenClaw on DigitalOcean — Always-On Instance Guide

This guide sets up a dedicated OpenClaw node on DigitalOcean so Claudius stays online even when your notebook is closed.

## 0) Recommended architecture (important)

- **Primary = VPS** (always on, owns messaging channels)
- **Secondary = laptop** (maintenance/dev fallback)
- Keep identity/memory in GitHub (`clawd-memory`) and sync on VPS

> Do **not** run the same messaging channel token actively on two gateways at once (can cause duplicate/conflicting replies).

---

## 1) Create the Droplet

Recommended baseline:
- Region: nearest to you
- Image: **Ubuntu 24.04 LTS**
- Size: **2 vCPU / 4 GB RAM** (safe starting point)
- Disk: 60+ GB (logs, caches, browsers)
- Auth: **SSH key only** (disable password login)
- Enable: monitoring + automatic backups (DO snapshot backup)

Optional but good:
- Reserve a static IP
- Attach a domain/subdomain (if you want clean access)

---

## 2) First server hardening

SSH in:
```bash
ssh root@YOUR_DROPLET_IP
```

Create admin user + sudo:
```bash
adduser claw
usermod -aG sudo claw
```

Firewall (UFW):
```bash
ufw allow OpenSSH
ufw enable
ufw status
```

(Recommended) restrict SSH to your IP in DO Cloud Firewall.

---

## 3) Install Node 22+

As `claw` user:
```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
node -v
npm -v
```

---

## 4) Install OpenClaw

```bash
npm install -g openclaw@latest
openclaw --version
```

If you prefer beta/dev track, switch after install:
```bash
openclaw update --channel beta --yes
# or
openclaw update --channel dev --yes
```

---

## 5) Onboard + daemon

```bash
openclaw onboard --install-daemon
openclaw status
openclaw gateway status
```

Run checks:
```bash
openclaw doctor
openclaw security audit --deep
```

---

## 6) Restore your identity/memory/workspace

```bash
mkdir -p ~/.openclaw
cd ~/.openclaw
# If workspace already exists, back it up first
mv workspace workspace.bak.$(date +%s) 2>/dev/null || true
git clone https://github.com/lf1up/clawd-memory.git workspace
```

Verify key files exist:
- `SOUL.md`
- `IDENTITY.md`
- `MEMORY.md`
- `personas/`

---

## 7) Channel ownership strategy

For always-on behavior:
- Move active messaging channels to VPS gateway
- Keep laptop gateway either:
  - stopped, or
  - running without channel ownership (maintenance only)

This prevents double replies.

---

## 8) Daily update + backup jobs (VPS)

You already have these patterns; recreate/verify on VPS:
- daily workspace backup cron (commit+push)
- daily forced update cron (with fallback tag logic)

Check:
```bash
openclaw cron list
```

---

## 9) Access safely

Best practice:
- Keep gateway private (loopback/local)
- Use SSH tunnel or Tailscale instead of public wide-open ports

Example SSH tunnel for dashboard:
```bash
ssh -L 18789:127.0.0.1:18789 claw@YOUR_DROPLET_IP
```
Then open `http://127.0.0.1:18789` locally.

---

## 10) Disaster recovery checklist

If VPS dies:
1. Create new Droplet
2. Install Node + OpenClaw
3. Clone `clawd-memory` into `~/.openclaw/workspace`
4. Restore channels/tokens
5. Recreate cron jobs
6. Verify with `openclaw status` + test message

---

## Optional: active/standby two-node model

- Node A (VPS) active; Node B standby
- Both sync from GitHub
- Only A owns channels
- On failover, disable A channel ownership and enable B

This is the safest practical HA model for assistant messaging flows.
