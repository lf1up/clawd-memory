# OpenClaw on DigitalOcean — Complete Production Guide (Primary + Local Fallback)

This is the full end-to-end guide to run Claudius on a **DigitalOcean Droplet** as the always-on primary, while keeping your notebook as a synchronized standby.

Goal:
- You can close your notebook and still continue in Discord from phone.
- Local and cloud stay as close as possible in state.
- Failover is predictable (no double replies, no race conditions).

---

## 1) Architecture you want

Use **active/passive**, not active/active.

- **Node A (Cloud / DO Droplet) = Active**
  - owns Discord token
  - processes live messages
- **Node B (Local notebook) = Passive standby**
  - keeps synced state
  - does not own channel while A is active

Why: running the same bot token on two active gateways can produce duplicate replies and race behavior.

---

## 2) Prerequisites

- DigitalOcean account
- Your SSH key on your laptop (`~/.ssh/id_ed25519.pub` recommended)
- GitHub repo with your workspace state (`clawd-memory`)
- Local OpenClaw already working (you have this)

Optional but recommended:
- Domain/subdomain for VPS
- Tailscale for private access

---

## 3) Create the Droplet (DigitalOcean)

In DO Control Panel:

1. Create → **Droplets**
2. Region: closest to you
3. Image: **Ubuntu 24.04 LTS**
4. Size: start with **2 vCPU / 4 GB RAM / 80 GB SSD**
5. Authentication: **SSH Key** (no password)
6. Enable:
   - Monitoring
   - Automated backups
7. Optional:
   - Reserved IP
   - Project + tags (e.g., `openclaw`, `prod`)

Reference: DO Droplet creation docs (create workflow, region/image/size/auth/backups).

---

## 4) Baseline server hardening (first login)

SSH in as root:
```bash
ssh root@YOUR_DROPLET_IP
```

Create non-root admin:
```bash
adduser claw
usermod -aG sudo claw
```

Set SSH daemon hardening:
```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
systemctl restart sshd
```

Enable host firewall (UFW):
```bash
ufw allow OpenSSH
ufw enable
ufw status verbose
```

Set DO Cloud Firewall (outside VM) for defense-in-depth:
- Inbound: TCP 22 only from your IP range
- Inbound: none else unless explicitly needed
- Outbound: allow all (or tighten later)

Then test new SSH session before closing root shell:
```bash
ssh claw@YOUR_DROPLET_IP
```

---

## 5) Install runtime dependencies

As `claw` user:

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs git rsync jq
node -v
npm -v
```

Install OpenClaw:
```bash
npm install -g openclaw@latest
openclaw --version
```

Choose channel (recommended beta or stable):
```bash
openclaw update --channel beta --yes
# or stable:
# openclaw update --channel stable --yes
```

---

## 6) Onboard OpenClaw on the VPS

```bash
openclaw onboard --install-daemon
openclaw status
openclaw gateway status
openclaw doctor
openclaw security audit --deep
```

If dashboard is needed over SSH tunnel:
```bash
# run on your laptop
ssh -L 18789:127.0.0.1:18789 claw@YOUR_DROPLET_IP
# then open locally:
# http://127.0.0.1:18789
```

---

## 7) Restore identity/memory/personas on VPS

```bash
mkdir -p ~/.openclaw
cd ~/.openclaw
mv workspace workspace.bak.$(date +%s) 2>/dev/null || true
git clone https://github.com/lf1up/clawd-memory.git workspace
```

Verify:
```bash
ls -la ~/.openclaw/workspace
```

Must include your core files (`SOUL.md`, `IDENTITY.md`, `MEMORY.md`, `memory/`, `personas/`).

---

## 8) Configure channel ownership (critical)

Move Discord bot ownership to VPS instance.

Rules:
- While VPS is active, local must not be channel-active.
- Keep token on local only for emergency failover.

Operational policy:
- **Normal mode:** VPS gateway running, local gateway stopped or channel-disabled.
- **Failover mode:** disable/stop VPS ownership first, then enable/start local.

---

## 9) Keep cloud and local in sync (exact-copy strategy)

You need two sync layers:

### Layer A — Git sync (human-readable memory + configs)

Sync repo files:
- `README.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`, `MEMORY.md`
- `memory/*.md`, `personas/*`, guides/docs

Cloud pushes; local pulls.

### Layer B — Runtime sync (session transcripts/state)

Sync `~/.openclaw/agents/` from VPS → local.

This captures session logs/transcripts not stored in git.

---

## 10) Implement sync scripts

### 10.1 On local notebook, create sync script

`~/bin/openclaw-sync-from-vps.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

VPS="claw@YOUR_DROPLET_IP"
LOCAL_OC="$HOME/.openclaw"

mkdir -p "$LOCAL_OC"

# 1) pull workspace repo updates
if [ -d "$LOCAL_OC/workspace/.git" ]; then
  git -C "$LOCAL_OC/workspace" pull --rebase || true
fi

# 2) pull runtime agent state from VPS
rsync -az --delete \
  "$VPS:$HOME/.openclaw/agents/" \
  "$LOCAL_OC/agents/"

# 3) optional: copy config snapshot (do not overwrite blindly)
rsync -az \
  "$VPS:$HOME/.openclaw/openclaw.json" \
  "$LOCAL_OC/openclaw.json.vps.snapshot" || true

echo "sync complete"
```

Make executable:
```bash
chmod +x ~/bin/openclaw-sync-from-vps.sh
```

### 10.2 Schedule periodic sync on local

```bash
crontab -e
```

Add:
```cron
*/15 * * * * /Users/YOUR_USER/bin/openclaw-sync-from-vps.sh >> /Users/YOUR_USER/.openclaw/sync.log 2>&1
```

This keeps local within ~15 minutes of cloud state.

---

## 11) Failover runbook (cloud -> local)

When VPS is unavailable or maintenance is needed:

1. Ensure VPS won’t answer (stop gateway or disable channel ownership).
2. Run final sync (if reachable):
   ```bash
   ~/bin/openclaw-sync-from-vps.sh
   ```
3. Start local gateway/channel ownership.
4. Send one test DM in Discord.
5. Confirm single response.

### Failback (local -> cloud)

1. Stop local channel ownership.
2. Push local workspace changes:
   ```bash
   git -C ~/.openclaw/workspace add -A
   git -C ~/.openclaw/workspace commit -m "chore(sync): failback handoff" || true
   git -C ~/.openclaw/workspace push
   ```
3. On VPS pull latest repo + start ownership.
4. Test DM again.

---

## 12) OpenClaw jobs to keep system healthy

On VPS, verify/create:

- Daily backup job (already in your setup style)
- Daily forced update job (with tag fallback)
- Optional weekly security audit reminder

Check:
```bash
openclaw cron list
openclaw update status
```

---

## 13) Security hardening checklist (recommended)

- SSH key-only auth ✅
- Root SSH login disabled ✅
- UFW enabled ✅
- DO Cloud Firewall applied ✅
- OpenClaw security audit baseline ✅
- 2FA on GitHub + Discord bot owner account ✅
- Keep gateway private where possible (SSH tunnel/Tailscale preferred) ✅

---

## 14) Validation test (do this once end-to-end)

1. Send DM while notebook is open: VPS replies.
2. Close notebook completely.
3. Send DM from phone: VPS still replies.
4. Simulate VPS outage; fail over to local using runbook.
5. Confirm local replies once (no duplicates).
6. Restore VPS and fail back.

If all six pass, your setup is production-ready.

---

## 15) Known limitations (important)

- “Exact same state” means **exact up to last sync point**.
- Active/active with same bot token is unsafe.
- Runtime logs are instance-local unless you rsync/centralize them.
- Never overwrite configs blindly during sync; keep snapshots and review diffs.

---

## 16) Quick command appendix

### VPS health
```bash
openclaw status
openclaw gateway status
openclaw doctor
openclaw security audit --deep
openclaw update status
```

### Repo backup
```bash
git -C ~/.openclaw/workspace add -A
git -C ~/.openclaw/workspace commit -m "chore(memory): backup" || true
git -C ~/.openclaw/workspace push
```

### SSH tunnel dashboard
```bash
ssh -L 18789:127.0.0.1:18789 claw@YOUR_DROPLET_IP
```

---

If you want, next I can generate two ready files for your exact username/IP:
1) `openclaw-sync-from-vps.sh`
2) `openclaw-failover-checklist.md`
