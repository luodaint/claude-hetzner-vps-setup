# Hetzner VPS Setup Guide

> **This file is a Claude Code setup prompt. When you feed this file to Claude Code, it must follow the instructions in the CLAUDE INSTRUCTIONS block below before doing anything else.**

---

## CLAUDE INSTRUCTIONS

Before executing any setup steps, ask the user the following questions **one at a time** and wait for each answer:

1. **Username:** "What username do you want to create on the server?"

2. **SSH public key:** "Please paste your SSH public key (or multiple keys, one per line). This will be the only way to log into the server."

3. **Tailscale:** "Do you want to set up Tailscale for private network access from your phone and computer? (yes/no)."
   - If yes: "Please paste your Tailscale auth key. You can generate one at https://login.tailscale.com/admin/settings/keys — use a reusable auth key if you plan to add multiple devices."

Once you have the answers, proceed with the setup steps below. Substitute `<USERNAME>`, `<SSH_PUBLIC_KEY>`, and Tailscale steps accordingly. Skip Section 9 (Tailscale) entirely if the user said no.

Also, ask the user if they want dokploy installed. If the answer is yes, proceed to Step 10 of the dokploy installation. If Dokploy is installed, configure it so that it is ONLY accessible via Tailscale (tailscale0 interface) and not exposed on the public internet. Do not open any Dokploy ports in UFW except through tailscale0.

---

## System Baseline

- **OS:** Ubuntu 24.04 LTS
- **Hostname:** set via Hetzner console or `hostnamectl set-hostname <your-hostname>`

---

## 1. Initial Package Install

```bash
apt update && apt upgrade -y
apt install -y \
  fail2ban \
  curl \
  git \
  vim \
  ufw \
  ca-certificates \
  gnupg \
  lsb-release \
  unattended-upgrades \
  apt-transport-https \
  software-properties-common \
  jq
```

---

## 2. Create the Admin User

```bash
useradd -m -s /bin/bash -G sudo,adm <USERNAME>
```

Set home directory permissions:

```bash
chmod 750 /home/<USERNAME>
chown <USERNAME>:<USERNAME> /home/<USERNAME>
```

Lock the root password as an extra safeguard:

```bash
passwd -l root
```

---

## 3. SSH Key Authentication Setup

Create the `.ssh` directory with strict permissions:

```bash
mkdir -p /home/<USERNAME>/.ssh
chmod 700 /home/<USERNAME>/.ssh
chown <USERNAME>:<USERNAME> /home/<USERNAME>/.ssh
```

Write the authorized_keys file using the key(s) the user provided:

```bash
cat > /home/<USERNAME>/.ssh/authorized_keys << 'EOF'
<SSH_PUBLIC_KEY>
EOF
```

Lock down the file:

```bash
chmod 600 /home/<USERNAME>/.ssh/authorized_keys
chown <USERNAME>:<USERNAME> /home/<USERNAME>/.ssh/authorized_keys
```

---

## 4. Sudo Access (Passwordless)

```bash
echo "<USERNAME> ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/<USERNAME>
chmod 440 /etc/sudoers.d/<USERNAME>
visudo -cf /etc/sudoers.d/<USERNAME>
```

---

## 5. SSH Server Hardening

Back up the current SSH config:

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%F-%H%M%S)
```

Replace `/etc/ssh/sshd_config` with the following:

```bash
cat > /etc/ssh/sshd_config << 'EOF'
# Basic auth hardening
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
PermitEmptyPasswords no
UsePAM yes
StrictModes yes

# Restrict who can log in
AllowUsers <USERNAME>

# Session controls
LoginGraceTime 30
MaxAuthTries 3
MaxSessions 5
ClientAliveInterval 300
ClientAliveCountMax 2
MaxStartups 10:30:60

# Disable unused features
AllowAgentForwarding no
AllowTcpForwarding no
X11Forwarding no
PermitTunnel no
PrintMotd no
TCPKeepAlive no

# Logging
SyslogFacility AUTH
LogLevel VERBOSE

# SFTP
Subsystem sftp /usr/lib/openssh/sftp-server
EOF
```

Test config before restarting:

```bash
sshd -t
```


Restart SSH (keep your current session open until verified):

```bash
systemctl restart ssh
systemctl enable ssh
```

**Verify login works** in a new terminal before closing the current session.

---

## 6. Fail2ban Configuration

Create `/etc/fail2ban/jail.local`:

```bash
cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime = 24h
findtime = 10m
maxretry = 3
backend = systemd
ignoreip = 127.0.0.1/8 ::1

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = %(sshd_log)s
maxretry = 3
EOF
```

> **Note:** If you have a static home/office IP, add it to `ignoreip` to avoid locking yourself out.

Enable and start fail2ban:

```bash
systemctl enable fail2ban
systemctl restart fail2ban
```

Verify the jail is active:

```bash
fail2ban-client status
fail2ban-client status sshd
```

---

## 7. Unattended Security Upgrades

```bash
apt install -y unattended-upgrades
dpkg-reconfigure --priority=low unattended-upgrades
```

Ensure daily checks are enabled:

```bash
cat > /etc/apt/apt.conf.d/20auto-upgrades << 'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
EOF
```

Optional cleanup settings:

```bash
cat > /etc/apt/apt.conf.d/52unattended-upgrades-local << 'EOF'
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "false";
EOF
```

### Basic Kernel / Network Hardening

```bash
cat > /etc/sysctl.d/99-hardening.conf << 'EOF'
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
EOF
```

Apply:

```bash
sysctl --system
```

---

## 8. Firewall (UFW)

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp comment 'SSH'
ufw allow 80/tcp comment 'HTTP'
ufw allow 443/tcp comment 'HTTPS'
ufw --force enable
ufw status verbose
```

> If Tailscale is being set up (Section 9), also run:
> ```bash
> ufw allow in on tailscale0 comment 'Tailscale'
> ```

---

## 9. Tailscale (skip if user said no)

Install Tailscale:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Bring up Tailscale using the auth key the user provided:

```bash
tailscale up --authkey=<TAILSCALE_AUTH_KEY> --ssh
```

The `--ssh` flag enables **Tailscale SSH**, which means you can also SSH into this server over the Tailscale network without needing to expose port 22 publicly at all.

Enable and verify:

```bash
systemctl enable tailscaled
systemctl restart tailscaled
tailscale status
tailscale ip -4
```

Tailscale will assign the server a private IP (e.g., `100.x.x.x`). You can then SSH via:

```bash
ssh <USERNAME>@<tailscale-ip>
# or by hostname:
ssh <USERNAME>@<hostname>
```

Allow all traffic coming from the Tailscale interface:

```bash
ufw allow in on tailscale0 comment 'Tailscale'
ufw reload
```

You can SSH through Tailscale with:

```bash
ssh <USERNAME>@<TAILSCALE_IP>
```

or 

```bash
ssh <USERNAME>@<HOSTNAME>
```

**Connect your devices:**
- **iPhone/Android:** Install the Tailscale app from the App Store / Play Store and log in with the same Tailscale account.
- **Mac/PC:** Install from https://tailscale.com/download and log in with the same account.

All devices on the same Tailscale account form a private network — no open ports required.

**Optional: lock down public SSH once Tailscale is working**

Once you've confirmed SSH over Tailscale works from your devices, you can remove the public SSH rule and only allow SSH from the Tailscale interface:

```bash
ufw delete allow 22/tcp
ufw allow in on tailscale0 to any port 22 comment 'SSH via Tailscale only'
ufw reload
ufw status verbose
```

> Only do this after confirming Tailscale SSH works, or you will lock yourself out.

---

## 10. Install Dokploy (if user allowed it)

“correct” architecture

```
Internet
   ↓
Hetzner Server
   ├── Public ports:
   │     80/443 → your app only
   │
   ├── Private (Tailscale only):
   │     3000 → Dokploy
   │     22 → SSH
   │
   └── Internal:
         Postgres
         Redis
         Workers
```

### Install Docker (required for Dokploy)

Remove conflicting packages:

```bash
apt remove -y docker docker-engine docker.io containerd runc || true
```

Install Docker using the official repository:

```bash
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  > /etc/apt/sources.list.d/docker.list

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Enable Docker

```bash
systemctl enable docker
systemctl restart docker
docker version
```

Allow the admin user to manage Docker:

```
usermod -aG docker <USERNAME>
```

### Install Dokploy

```bash
curl -sSL https://dokploy.com/install.sh | sh
```

#### Harden Dokploy network exposure

Dokploy’s dashboard commonly uses port `3000`. Deny public access:

```bash
ufw deny 3000/tcp
```

If Tailscale is enabled, allow Dokploy only over Tailscale:

```bash
ufw allow in on tailscale0 to any port 3000 proto tcp comment 'Dokploy via Tailscale only'
ufw reload
```

Verify firewall rules:

```bash
ufw status numbered
```

#### Important notes for Dokploy

* Dokploy dashboard should be accessed only via:

  * `http://<TAILSCALE_IP>:3000`
  * or via a Tailscale-resolved hostname
* Do **not** expose port `3000` publicly
* Ports `80/443` may remain public only for your deployed applications
* Postgres, Redis, and internal services should **not** be exposed publicly unless explicitly required

If Docker created iptables rules that bypass your intended exposure model, verify from another machine that:

* public IP cannot reach port 3000
* Tailscale IP can reach port 3000

Example checks:

```bash
ss -tulpn | grep 3000 || true
docker ps
ufw status verbose
```

---

## 11. How to Connect via SSH

**Direct (public IP):**
```bash
ssh <USERNAME>@<server-ip>
```

**Via Tailscale (if enabled):**
```bash
ssh <USERNAME>@<tailscale-ip-or-hostname>
```

- Root login is **disabled**.
- Password login is **disabled** — SSH key required.
- Only `<USERNAME>` is allowed to SSH in.
- Fail2ban bans an IP after **3 failed SSH attempts** for **24 hours**.

---

## 12. Verification Checklist

```bash
# Hostname
hostnamectl

# SSH
systemctl status ssh
sshd -t

# Firewall
ufw status verbose

# Fail2ban
systemctl status fail2ban
fail2ban-client status
fail2ban-client status sshd

# Tailscale
systemctl status tailscaled
tailscale status
tailscale ip -4

# Docker
systemctl status docker
docker version
docker ps

# Dokploy
ss -tulpn | grep 3000 || true

# User
id <USERNAME>
```

Expected user groups should include at least:

* `adm`
* `sudo`

and, if Dokploy was installed:

* `docker`

---

## 13. Final Security Notes

* Root SSH login is disabled
* Password SSH login is disabled
* Only `<USERNAME>` may log in over SSH
* Fail2ban protects SSH
* Automatic security updates are enabled
* Dokploy dashboard is not public
* Tailscale is the preferred access path for admin services
* Only expose `80/443` publicly if the server will host public applications

---

## 14. Suggested Next Steps After Base Setup

After this baseline is complete, the next good hardening steps are:

1. Add off-server backups for PostgreSQL
2. Add monitoring and alerts
3. Add a domain and TLS for your public apps
4. Keep Dokploy admin private on Tailscale
5. Avoid exposing Redis and PostgreSQL to the internet
6. Test recovery from a fresh VPS

## Notes

- The `adm` group membership grants read access to system logs (`/var/log`).
- `qemu-guest-agent` is installed by default on Hetzner VPS images — no action needed.
- This script runs as root — ensure `<USERNAME>` has the necessary permissions for resources they'll manage.
