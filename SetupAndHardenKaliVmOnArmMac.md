# KaliCTF-Hardening.md

## 🎯 Overview

This document provides a **complete hardening guide** for a Kali Linux ARM VM on Apple Silicon (M-series) Macs using VMware Fusion. This configuration is optimized for **CTF competitions and secure binary analysis** with proper network isolation, SSH hardening, and firewall configuration.

**Threat Model:** This setup protects against remote network attacks, brute force attempts, and containment of malicious CTF binaries. It provides adequate isolation for competitive CTF work and standard security challenges.

---

## 🏗️ Architecture Overview

### Network Configuration
- **eth0 (NAT)**: Internet access for tool downloads and updates
- **eth1 (Host-Only)**: Isolated private network (192.168.24.0/24) for SSH access only
- **Firewall**: UFW configured to block all incoming traffic except SSH on eth1

### Security Layers
```
┌─────────────────────────────────────────┐
│         macOS Host (Apple Silicon)      │
│  - SSH client with ED25519 keys         │
│  - Private network: 192.168.24.1        │
└─────────────────┬───────────────────────┘
                  │ Host-Only Network (eth1)
                  │ SSH only, key-based auth
┌─────────────────▼───────────────────────┐
│         Kali Linux ARM VM               │
│  ┌─────────────────────────────────┐   │
│  │  UFW Firewall                   │   │
│  │  - Deny all incoming (default)  │   │
│  │  - Allow SSH on eth1 only       │   │
│  │  - Rate limiting enabled        │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │  Hardened SSH (port 22)         │   │
│  │  - Key-only authentication      │   │
│  │  - Root login disabled          │   │
│  │  - Max 3 auth attempts          │   │
│  └─────────────────────────────────┘   │
│  ┌─────────────────────────────────┐   │
│  │  CTF Toolchain                  │   │
│  │  - Python venv with pwntools    │   │
│  │  - QEMU multi-arch support      │   │
│  │  - Ghidra, radare2, GDB         │   │
│  └─────────────────────────────────┘   │
└─────────────────┬───────────────────────┘
                  │ NAT Network (eth0)
                  │ Internet access
                  ▼
            [ Internet ]
```

---

## 🛠️ Prerequisites

- macOS with Apple Silicon (M1/M2/M3/M4)
- VMware Fusion installed (Apple Silicon version)
- Kali Linux ARM image

---

## 📋 Part 1: macOS Host Configuration

### 1. Install VMware Fusion

Download and install [VMware Fusion](https://www.vmware.com/products/fusion.html) for Apple Silicon.

### 2. Configure VM Network Adapters

In VMware Fusion (VM must be powered off):

1. **Settings → Network Adapter → Add Device → Network Adapter**
2. Configure adapters:
   - **Adapter 1**: Share with my Mac (NAT) - for internet access
   - **Adapter 2**: Private to my Mac - for SSH access

### 3. Generate SSH Key Pair

```bash
ssh-keygen -t ed25519 -C "kali-ctf-vm" -f ~/.ssh/kali_ctf_ed25519
```

**Important:** Use a strong passphrase (20+ characters recommended).

Your keys are now located at:
- Private key: `~/.ssh/kali_ctf_ed25519`
- Public key: `~/.ssh/kali_ctf_ed25519.pub`

### 4. Create SSH Config (Optional but Recommended)

```bash
nano ~/.ssh/config
```

Add:

```
Host kali-ctf
    HostName 192.168.24.10
    User kali
    IdentityFile ~/.ssh/kali_ctf_ed25519
    IdentitiesOnly yes
```

Now you can connect with simply: `ssh kali-ctf`

---

## 🔧 Part 2: Kali VM Initial Setup

> ⚠️ All commands below run **inside the Kali VM**

### 1. Update System

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Configure Static IP on Host-Only Interface

Check your network interfaces:

```bash
ip a
```

Identify your two interfaces (typically `eth0` and `eth1`).

Create network configuration:

```bash
sudo nano /etc/network/interfaces
```

Add:

```ini
auto lo
iface lo inet loopback

# Host-only network for SSH
auto eth1
iface eth1 inet static
    address 192.168.24.10
    netmask 255.255.255.0
```

**Note:** eth0 will be managed by DHCP automatically (NAT network).

Apply configuration:

```bash
sudo systemctl restart networking
```

Verify:

```bash
ip a show eth1
```

You should see `192.168.24.10/24` assigned to eth1.

---

## 🔐 Part 3: SSH Hardening

### 1. Install OpenSSH Server

```bash
sudo apt install -y openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

### 2. Copy Public Key from Host

From your **macOS terminal**:

```bash
ssh-copy-id -i ~/.ssh/kali_ctf_ed25519.pub kali@192.168.24.10
```

Enter your Kali user password when prompted.

### 3. Harden SSH Configuration

```bash
sudo nano /etc/ssh/sshd_config
```

Ensure these settings (add or modify):

```ini
# Authentication
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
PermitEmptyPasswords no
MaxAuthTries 3

# Security
Protocol 2
X11Forwarding no
AllowTcpForwarding no
AllowStreamLocalForwarding no

# Timeout settings
ClientAliveInterval 300
ClientAliveCountMax 2

# User restrictions
AllowUsers kali

# Strong cryptography
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

### 4. Test SSH Connection

From your **macOS terminal**:

```bash
ssh -i ~/.ssh/kali_ctf_ed25519 kali@192.168.24.10
# Or if you created the SSH config:
ssh kali-ctf
```

**⚠️ Do not proceed until SSH key authentication works!**

---

## 🛡️ Part 4: Firewall Configuration

### 1. Install and Configure UFW

```bash
sudo apt install -y ufw
```

### 2. Set Default Policies

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### 3. Configure SSH Access Rules

```bash
# Allow SSH on host-only interface with rate limiting
sudo ufw limit in on eth1 to any port 22 proto tcp

# Explicitly deny SSH on internet-facing interface
sudo ufw deny in on eth0 to any port 22 proto tcp
```

### 4. Block Common Attack Vectors

```bash
# Block unnecessary ports
sudo ufw deny 23    # Telnet
sudo ufw deny 139   # NetBIOS
sudo ufw deny 445   # SMB
sudo ufw deny 3389  # RDP
```

### 5. Enable Firewall

```bash
sudo ufw enable
```

**⚠️ Type 'y' when prompted**

### 6. Verify Configuration

```bash
sudo ufw status verbose
```

Expected output:

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp on eth1             LIMIT IN    Anywhere
22/tcp on eth0             DENY IN     Anywhere
23                         DENY IN     Anywhere
139                        DENY IN     Anywhere
445                        DENY IN     Anywhere
3389                       DENY IN     Anywhere
```

### 7. Enable UFW Logging (Optional)

```bash
sudo ufw logging medium
```

View logs:

```bash
sudo tail -f /var/log/ufw.log
```

---

## 🔒 Part 5: System Hardening

### 1. Kernel Security Parameters

```bash
sudo nano /etc/sysctl.conf
```

Add these security settings:

```ini
# Disable IP forwarding
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# Ignore ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# Ignore source-routed packets
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# Enable TCP SYN cookie protection (DDoS mitigation)
net.ipv4.tcp_syncookies = 1

# Log suspicious packets
net.ipv4.conf.all.log_martians = 1

# Ignore ICMP ping requests (optional - may break some CTF challenges)
# net.ipv4.icmp_echo_ignore_all = 1

# Increase system security
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
```

Apply changes:

```bash
sudo sysctl -p
```

### 2. Automatic Security Updates

```bash
sudo apt install -y unattended-upgrades apt-listchanges
```

Configure automatic updates:

```bash
sudo dpkg-reconfigure -plow unattended-upgrades
```

Select **Yes** to enable automatic updates.

Verify configuration:

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
```

Should show:

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

### 3. Install Fail2Ban

Protects against brute force attacks:

```bash
sudo apt install -y fail2ban
```

Create local configuration:

```bash
sudo nano /etc/fail2ban/jail.local
```

Add:

```ini
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
```

Enable and start:

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Check status:

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

### 4. Disable Unnecessary Services

List enabled services:

```bash
systemctl list-unit-files --type=service --state=enabled
```

Disable unnecessary services (VM-specific):

```bash
# Bluetooth (not needed in VM)
sudo systemctl disable bluetooth.service
sudo systemctl stop bluetooth.service

# Modem Manager (not needed in VM)
sudo systemctl disable ModemManager.service
sudo systemctl stop ModemManager.service

# Avahi daemon (network discovery - not needed)
sudo systemctl disable avahi-daemon.service
sudo systemctl stop avahi-daemon.service

# Printer services (if not needed)
# sudo systemctl disable cups.service
# sudo systemctl stop cups.service
```

Verify changes:

```bash
systemctl list-unit-files --type=service --state=enabled | grep -E 'bluetooth|ModemManager|avahi'
```

Should return no results.

### 5. Secure Shared Memory

Edit `/etc/fstab`:

```bash
sudo nano /etc/fstab
```

Add this line:

```
tmpfs /run/shm tmpfs defaults,noexec,nodev,nosuid 0 0
```

Apply without reboot:

```bash
sudo mount -o remount /run/shm
```

---

## 🧰 Part 6: CTF Toolchain Installation

### 1. Install Multi-Architecture Support (QEMU)

Essential for running binaries from other architectures:

```bash
sudo apt install -y qemu-system qemu-user qemu-user-static qemu-user-binfmt binfmt-support
```

Enable binfmt:

```bash
sudo update-binfmts --enable
```

Verify:

```bash
update-binfmts --display | grep qemu
```

### 2. Create Python Virtual Environment

```bash
python3 -m venv ~/ctf-env
source ~/ctf-env/bin/activate
```

Upgrade pip:

```bash
pip install --upgrade pip setuptools wheel
```

### 3. Install Core Python Tools

```bash
pip install pwntools ropper unicorn z3-solver capstone keystone-engine
```

For memory forensics (if needed):

```bash
pip install volatility3
```

Verify installation:

```bash
pip list | grep -E 'pwntools|ropper|unicorn|z3|capstone|keystone'
```

### 4. Install Binary Analysis Tools

```bash
sudo apt install -y \
    ghidra \
    radare2 \
    gdb gdb-multiarch \
    ltrace strace \
    binwalk \
    foremost \
    sleuthkit \
    hexedit \
    xxd
```

### 5. Install Network Analysis Tools

```bash
sudo apt install -y \
    nmap \
    netcat-traditional \
    tcpdump \
    wireshark \
    tshark \
    nikto \
    gobuster \
    feroxbuster
```

### 6. Install Web Application Tools

```bash
sudo apt install -y \
    burpsuite \
    sqlmap \
    hydra \
    john \
    hashcat \
    wfuzz
```

### 7. Create Activation Alias

Add to `~/.bashrc` or `~/.zshrc`:

```bash
echo 'alias ctf="source ~/ctf-env/bin/activate"' >> ~/.bashrc
source ~/.bashrc
```

Now you can activate your environment with just: `ctf`

---

## 📦 Part 7: Snapshot & Backup Strategy

### 1. Create VMware Snapshots

After completing the setup:

1. **Shut down the VM completely**
2. In VMware Fusion: **Virtual Machine → Snapshots → Take Snapshot**
3. Name it: **"Fully Hardened CTF Base"**
4. Description: "Complete hardened setup with all tools - [Date]"

**Recommended snapshot workflow:**

```
Clean Hardened Base (Initial)
    ↓
Before Competition 1
    ↓
After Competition 1 (if modifications needed)
    ↓
Before Competition 2
    ...
```

### 2. Backup Entire VM

Your VM is located at:

```
~/Virtual Machines/<VM_NAME>.vmwarevm/
```

Backup to external drive:

```bash
# From macOS terminal
cp -r ~/Virtual\ Machines/<VM_NAME>.vmwarevm /Volumes/BackupDrive/KaliCTF-Backup/
```

**Backup schedule recommendations:**
- After major tool installations
- Before analyzing unknown binaries
- Monthly (minimum)
- Before major system updates

### 3. Export VM (Optional)

For portability:

1. **File → Export to OVF**
2. Save to external storage
3. Can be imported on other systems

---

## 🎯 Part 8: CTF Workflow Guide

### Daily Workflow

#### 1. Connect to VM

From macOS terminal:

```bash
ssh kali-ctf
# Or: ssh -i ~/.ssh/kali_ctf_ed25519 kali@192.168.24.10
```

#### 2. Activate CTF Environment

```bash
ctf
# Or: source ~/ctf-env/bin/activate
```

#### 3. Work on Challenges

Your environment now has:
- ✅ All Python tools (pwntools, ropper, etc.)
- ✅ Binary analysis (Ghidra, radare2, GDB)
- ✅ Multi-architecture support (QEMU)
- ✅ Network tools (nmap, Wireshark, Burp)
- ✅ Complete isolation from host system

#### 4. After Competition

```bash
# Deactivate environment
deactivate

# Exit VM
exit
```

Then in VMware Fusion:
- **Restore to "Fully Hardened CTF Base" snapshot**
- Or take a new snapshot if you made valuable changes

### File Transfer Between Host and VM

#### Option 1: SCP (Recommended)

From macOS to VM:

```bash
scp -i ~/.ssh/kali_ctf_ed25519 challenge.bin kali@192.168.24.10:~/
```

From VM to macOS:

```bash
scp -i ~/.ssh/kali_ctf_ed25519 kali@192.168.24.10:~/exploit.py ~/Desktop/
```

#### Option 2: VMware Shared Folders

1. In VMware: **Settings → Sharing**
2. Enable and add a folder
3. Access in VM at: `/mnt/hgfs/<folder_name>`

**⚠️ Security Note:** Disable shared folders when analyzing untrusted binaries.

---

## 🔍 Part 9: Security Monitoring

### Check Firewall Status

```bash
sudo ufw status verbose
```

### Review Failed Login Attempts

```bash
# Failed SSH attempts
sudo lastb

# Successful logins
last

# Monitor auth log
sudo tail -f /var/log/auth.log
```

### Check Fail2Ban Status

```bash
sudo fail2ban-client status sshd
```

### Monitor System Services

```bash
# Check for failed services
sudo systemctl --failed

# Check running services
sudo systemctl list-units --type=service --state=running
```

### View UFW Logs

```bash
sudo tail -f /var/log/ufw.log
```

---

## 🆘 Troubleshooting

### Cannot SSH into VM

1. **Check VM is running and network is up:**
   ```bash
   ip a show eth1
   ```

2. **Verify SSH is running:**
   ```bash
   sudo systemctl status ssh
   ```

3. **Check firewall rules:**
   ```bash
   sudo ufw status verbose
   ```

4. **Test from VM itself:**
   ```bash
   ssh kali@127.0.0.1
   ```

5. **Check host can reach VM:**
   ```bash
   # From macOS
   ping 192.168.24.10
   ```

### UFW Blocking Legitimate Traffic

Temporarily disable to test:

```bash
sudo ufw disable
# Test connectivity
sudo ufw enable
```

Check logs:

```bash
sudo grep -i 'BLOCK' /var/log/ufw.log
```

### Fail2Ban Banned Your Own IP

Unban yourself:

```bash
sudo fail2ban-client set sshd unbanip 192.168.24.1
```

### Tools Not Found After SSH

Make sure to activate the environment:

```bash
source ~/ctf-env/bin/activate
```

### QEMU Not Running Foreign Binaries

Re-enable binfmt:

```bash
sudo update-binfmts --enable
```

---

## 📊 Security Posture Summary

### Threat Protection Matrix

| Threat Vector | Protection | Status |
|---------------|------------|--------|
| **Remote SSH Brute Force** | Key-only auth + Fail2Ban + rate limiting | ✅ Protected |
| **Network Scanning** | UFW blocks all unsolicited incoming | ✅ Protected |
| **Password Attacks** | No password auth enabled | ✅ Protected |
| **Malicious Binaries** | VM isolation + snapshots | ✅ Contained |
| **Privilege Escalation** | Root login disabled, kernel hardening | ✅ Mitigated |
| **VM Escape** | Relies on VMware security | ⚠️ Low Risk |
| **Persistent Malware** | Snapshot rollback capability | ✅ Recoverable |
| **Data Exfiltration** | Network monitoring + firewall | ⚠️ Monitoring Required |

### What This Setup Protects Against

✅ **Fully Protected:**
- Brute force SSH attacks
- Network-based remote exploitation
- Port scanning and reconnaissance
- Common malware spread vectors
- Accidental internet exposure

⚠️ **Mitigated but Requires Vigilance:**
- Advanced persistent threats (use snapshots)
- Zero-day VM escape exploits (keep VMware updated)
- Sophisticated rootkits (restore from known-good snapshot)

❌ **Not Protected Against:**
- Physical access to host machine
- Compromise of host macOS system
- User error (running `rm -rf /` as root)
- Social engineering attacks

---

## 🔄 Maintenance Schedule

### Weekly
- [ ] Check for failed services: `sudo systemctl --failed`
- [ ] Review UFW logs for suspicious activity
- [ ] Verify Fail2Ban is catching attempts: `sudo fail2ban-client status sshd`

### Monthly
- [ ] Update all packages: `sudo apt update && sudo apt upgrade -y`
- [ ] Review enabled services: `systemctl list-unit-files --state=enabled`
- [ ] Create new snapshot after updates
- [ ] Backup VM to external drive

### After Each CTF/Competition
- [ ] Restore to "Fully Hardened CTF Base" snapshot
- [ ] Or create new snapshot if tools were added
- [ ] Review system logs for anomalies
- [ ] Update tools in Python venv if needed

### Quarterly
- [ ] Review and update SSH configuration
- [ ] Audit firewall rules for relevance
- [ ] Test full VM restore from backup
- [ ] Review and update this documentation

---

## 🎓 Additional Resources

### Official Documentation
- [Kali Linux Documentation](https://www.kali.org/docs/)
- [VMware Fusion Documentation](https://docs.vmware.com/en/VMware-Fusion/)
- [UFW Documentation](https://help.ubuntu.com/community/UFW)
- [Fail2Ban Documentation](https://www.fail2ban.org/)

### CTF Resources
- [CTF Time](https://ctftime.org/) - Upcoming CTF competitions
- [PicoCTF](https://picoctf.org/) - Beginner-friendly CTF
- [HackTheBox](https://www.hackthebox.com/) - Penetration testing practice
- [pwntools Documentation](https://docs.pwntools.com/)

### Security Best Practices
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)

---

## ✅ Final Checklist

Before considering your VM fully hardened, verify:

- [ ] Two network adapters configured (NAT + Host-Only)
- [ ] Static IP assigned to eth1 (192.168.24.10)
- [ ] SSH key-based authentication working
- [ ] Password authentication disabled in SSH
- [ ] UFW enabled with proper rules
- [ ] SSH rate limiting active on eth1
- [ ] Unnecessary services disabled
- [ ] Kernel security parameters applied
- [ ] Automatic security updates enabled
- [ ] Fail2Ban installed and monitoring SSH
- [ ] Python virtual environment created
- [ ] All CTF tools installed and verified
- [ ] QEMU multi-arch support working
- [ ] VMware snapshot created ("Fully Hardened CTF Base")
- [ ] VM backed up to external storage
- [ ] SSH config created on host for easy access
- [ ] This documentation saved for future reference

---

## 📝 License

This documentation is provided as-is for educational purposes. Use at your own risk. Always ensure you have permission before conducting security testing or analysis.

---

**Last Updated:** October 2025  
**Target Platform:** Apple Silicon (M1/M2/M3/M4) macOS with VMware Fusion  
