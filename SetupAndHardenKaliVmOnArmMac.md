# SetupAndHardenKaliVmOnArmMac.md

## 📘 Overview & Purpose

This guide documents **every step required to build and harden a Kali Linux ARM virtual machine on macOS (Apple Silicon) using VMware Fusion** for CTF competitions and malware analysis.
It is designed so that **if both your host and VM are lost**, you can **rebuild the entire environment exactly** as it was configured — hardened, tool-ready, and isolated.

This setup ensures:

* 🔐 **Network isolation** between host and VM
* 🔑 **SSH key-only access** over a private network
* 🛡️ **Firewall enforcement** to block unwanted inbound traffic
* 🧰 **Comprehensive toolchain** for exploitation, reversing, forensics, and more
* 🧪 **Multi-architecture support** for analyzing binaries from other platforms
* 🧬 **Safe and repeatable CTF workflow**

---

## 🖥️ macOS (Host) Setup – VMware Fusion

### 1. Install VMware Fusion

If not already installed, download and install [VMware Fusion](https://www.vmware.com/products/fusion.html) (Apple Silicon version).

---

### 2. Add a Second Network Adapter (Host-Only)

In VMware Fusion:

* Shut down the VM if it is running.
* Go to **Settings → Network Adapter → Add Device → Network Adapter**
* Set **Adapter 1** to **Share with my Mac (NAT)** – provides internet to the VM.
* Set **Adapter 2** to **Private to my Mac** – creates a host-only virtual network.

This provides one interface for internet access (`eth0`) and one for secure SSH communication (`eth1`).

---

### 3. Generate SSH Keys (Host)

Generate a new SSH key pair on your host:

```bash
ssh-keygen -t ed25519 -C "ctfvm-key"
```

When prompted for a passphrase, use a strong one (e.g., 20+ characters, random letters/numbers/symbols).

Your keys will be located in:

```
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

---

### 4. Copy Public Key to VM

Once the VM is set up and SSH enabled (see below), copy your host public key into the VM:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub kali@192.168.24.10
```

---

## 🐧 Kali VM Setup & Hardening

> ⚠️ Run all commands below inside the Kali VM unless otherwise noted.

---

### 1. Network Configuration

After adding two network adapters:

Check interfaces:

```bash
ip a
```

Create a static IP for `eth1`:

```bash
sudo nano /etc/network/interfaces
```

Add:

```ini
auto lo
iface lo inet loopback

auto eth1
iface eth1 inet static
  address 192.168.24.10
  netmask 255.255.255.0
```

Restart networking or reboot the VM:

```bash
sudo systemctl restart networking
```

Verify:

```bash
ip a
```

You should see `eth0` (internet) and `eth1` (`192.168.24.10`).

---

### 2. Enable & Harden SSH

Install and enable SSH:

```bash
sudo apt update && sudo apt install -y openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

Check status:

```bash
sudo systemctl status ssh
```

Edit SSH config:

```bash
sudo nano /etc/ssh/sshd_config
```

Ensure these settings:

```ini
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

✅ Now you can SSH from your host:

```bash
ssh -i ~/.ssh/id_ed25519 kali@192.168.24.10
```

---

### 3. Configure Firewall (UFW)

Install and configure `ufw`:

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on eth1 to any port 22 proto tcp
sudo ufw deny in on eth0 to any port 22 proto tcp
sudo ufw enable
sudo ufw status verbose
```

Result should show SSH **allowed on `eth1`** and **denied on `eth0`**.

---

### 4. Multi-Architecture Support (QEMU)

Install QEMU and binfmt support:

```bash
sudo apt install -y qemu-system qemu-user qemu-user-binfmt qemu-utils binfmt-support
sudo update-binfmts --enable
```

✅ Now your VM can run foreign binaries (e.g., MIPS, ARM, PPC) transparently.

---

### 5. Python Virtual Environment & Toolchain

Create and activate the environment:

```bash
python3 -m venv ~/ctf-env
source ~/ctf-env/bin/activate
```

Upgrade essentials:

```bash
pip install --upgrade pip setuptools wheel
```

Install CTF toolchain:

```bash
pip install pwntools ropper unicorn z3-solver volatility3 capstone
```

Verify:

```bash
pip list | egrep 'pwntools|ropper|angr|z3|volatility3|capstone|keystone|unicorn'
```

Expected output:

```
capstone      6.0.0a5
pwntools      4.14.1
ropper        1.13.13
unicorn       2.1.4
volatility3   2.26.2
z3-solver     4.15.3.0
```

✅ Reactivate any time:

```bash
source ~/ctf-env/bin/activate
```

---

### 6. Ghidra Installation

Install Ghidra from Kali’s repository:

```bash
sudo apt install -y ghidra
```

Launch Ghidra with:

```bash
ghidra
```

---

### 7. Install Core CTF Tools

Install binary analysis, forensics, and network tools:

```bash
sudo apt install -y nmap tcpdump wireshark tshark foremost sleuthkit binwalk radare2 gdb gdb-multiarch ltrace strace
```

---

## 🔒 Security Hardening Breakdown

| Feature                 | What It Does                                | Why It Matters                                      |
| ----------------------- | ------------------------------------------- | --------------------------------------------------- |
| **Dual network setup**  | `eth0` (internet) and `eth1` (isolated SSH) | Keeps VM reachable only via private network         |
| **Key-based SSH only**  | Disables password logins                    | Eliminates brute force attack surface               |
| **Firewall (UFW)**      | Denies all incoming except SSH on `eth1`    | Blocks exposure on public interfaces                |
| **QEMU + binfmt**       | Runs foreign binaries safely                | Essential for CTF challenges on other architectures |
| **Virtual environment** | Isolates Python tooling                     | Prevents dependency conflicts                       |
| **Ghidra**              | Reverse engineering GUI                     | Essential for binary analysis                       |
| **VM snapshots**        | Rollback points                             | Quick restore after breakage or infection           |

---

## 💾 Post-Install Snapshot & Backup (VMware Fusion)

📍 **Do this after initial setup and tool installation.**

### 1. Create a Snapshot

* In VMware Fusion: **Virtual Machine → Snapshots → Take Snapshot**
* Name it `Clean Hardened Base`
* Add a note: “Fresh hardened Kali CTF environment”

This snapshot is your reset point before malware analysis or risky challenges.

---

### 2. Backup the Entire VM

Your VM resides at:

```
~/Virtual Machines/<name of kali vm here>
```

Copy this folder to an external drive:

```bash
cp -r ~/Virtual Machines/<name of kali vm here> /Volumes/YourBackupDrive/
```

✅ This allows instant restore by importing back into VMware Fusion.

---

## 🧠 CTF Workflow Guide

### 1. SSH into Your VM (No GUI Needed)

From your Mac terminal:

```bash
ssh -i ~/.ssh/id_ed25519 kali@192.168.24.10
```

Now you can run all commands directly inside the VM as if you were at its console.

---

### 2. Activate the CTF Environment

Before starting work:

```bash
source ~/ctf-env/bin/activate
```

Now `python`, `pip`, and all installed tools (like `pwntools`, `ropper`, etc.) are ready.

---

### 3. Work on Challenges Securely

* Analyze binaries with `radare2`, `ghidra`, or `gdb`
* Test exploits with `pwntools` or `unicorn`
* Run foreign binaries transparently with QEMU
* Use `volatility3` for memory dumps
* Capture traffic with `tcpdump` or `wireshark`

All of this happens **inside an isolated VM**, ensuring malware or CTF payloads cannot escape.

---

### 4. Reset to a Clean State Before Next Challenge

After a competition or risky analysis:

* Restore from your **Clean Hardened Base snapshot**
* Re-import your VM from the `.vmwarevm` backup if needed

This keeps your host system and VM safe from persistence or compromise.

---

## ✅ Summary

Following this guide builds a hardened, fully-equipped Kali ARM VM tailored for CTF use:

* 🔐 Key-only SSH on a private network
* 🛡️ Firewall blocking all other ingress
* 🧰 Full CTF toolchain in a dedicated Python venv
* 🧬 Multi-architecture support with QEMU
* 🧠 Reverse engineering tools including Ghidra
* 💾 Snapshot & backup strategy for safe resets
* 🧪 Secure SSH-based workflow from host to VM

This document is sufficient to rebuild the **exact working environment** even from a blank Mac(M Series) and empty VM.
