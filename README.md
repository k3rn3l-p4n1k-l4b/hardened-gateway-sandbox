# Headless Mac Fort-Box: Dual-Killswitch Torrenting Server with Automated Antivirus Pipeline

An architectural blueprint for repurposing an old, upgraded MacBook Pro (yes i had 2) into an isolated, headless torrenting server. This repository presents an enterprise-grade approach to network engineering, endpoint hardening, and automated pipeline security. 

---

## 🛠️ Hardware Specification
*   **Host Machine:** MacBook Pro (Mid-2012)
*   **Operating System:** Ubuntu Server 24.04 LTS (Headless)
*   **Storage Upgrade:** 480GB SATA SSD (Internal OS Boot Drive)
*   **Memory Upgrade:** 8GB DDR3 RAM
*   **External Storage:** 1TB External USB Hard Drive (Dedicated Target Data Array)

---

## 🎯 Project Motivation & Architecture Goals

### 1. Privacy Preservation (The Dual-Killswitch Engine)
Standard VPN configurations routinely leak unencrypted data during microsecond disconnection windows, service crashes, or boot cycles. This project implements three strict, overlapping layers of absolute boundary enforcement:
*   **Software Layer:** The torrent daemon is hard-bound exclusively to the virtual network adapter interface (`tun0`), rendering it incapable of initiating sockets over raw wires.
*   **OS Firewall Layer (`ufw`):** System-wide firewall policies default to a hard `DENY` for incoming and outgoing data, strictly restricting out-of-tunnel physical data transmissions.
*   **Hardware Gateway Layer (TP-Link Omada ACL):** A centralized network gateway policy enforces stateful traffic packet drops at the physical home router border if the host operating system is ever compromised.

### 2. Attack Surface Reduction (Network Isolation)
To protect the local home network, the machine is isolated to a specific network segment. Using strict local firewall filtering, **only a single designated Admin Workstation** is granted network authorization to access the SSH console (Port 22) and the Web Dashboard (Port 8080). All other local network discovery attempts are instantly dropped.

### 3. Supply Chain Security (Sanitization Pipeline)
Downloaded files from public peer networks represent an untrusted security risk. To prevent arbitrary malware payloads from crossing over into a clean network environment, this machine leverages a fully automated post-download orchestration pipeline using an asynchronous shell scanner engine (**ClamAV**).

---

## 🧠 Engineering Journal: Troubleshooting & Advanced Resolutions

This project served as an intensive exercise in Linux systems administration, low-level network analysis, and configuration auditing. Below are the key engineering hurdles encountered during prototyping and how they were systematically resolved.

### Challenge 1: Total Host Isolation Upon Firewalls Activation (The UFW/VPN Deadlock)
*   **The Symptom:** Activating `ufw` rules cleanly permitted incoming local SSH management connections, but the server completely lost outbound internet connectivity and the virtual `tun0` interface vanished entirely.
*   **The Root Cause Analysis:** OpenVPN was configured to handshake over standard `1194 UDP` and `443 TCP` vectors. However, the commercial VPN provider (NordVPN) frequently rotates remote target configurations dynamically to unstandardized high-numbered ports (e.g., `1231`-`1234 UDP`). The rigid initial firewall policy was dropping these out-of-band handshake packets on the physical network interface (`enp1s0f0`), causing the initialization sequence to fail and choke the tunnel.
*   **The Solution:** Rewrote the `ufw` outgoing matrix to safely allow stateful outbound UDP traffic globally on the physical card (`enp1s0f0`) while leaving outbound TCP fully blocked. This granted the local OpenVPN service process total flexibility to negotiate dynamically rotated handshakes with upstream infrastructure, while perfectly maintaining an airtight block on unencrypted TCP torrent peer connections.

### Challenge 2: Intermittent DNS Resolving Timeouts Under Active Firewalls
*   **The Symptom:** Even with successful tunnel creation and functional IP pings, web address lookups (`curl ifconfig.me`) threw severe timeout errors (`Resolving timed out`).
*   **The Root Cause Analysis:** Outbound DNS clearance rules on the physical interface were narrowly declared for `53/tcp`. In practice, standard Linux network resolvers issue system lookup queries over **UDP** on Port 53 rather than TCP. Because UDP 53 was omitted, the underlying OS was blinded from translating domain names into structural target IPs during boot configurations.
*   **The Solution:** Restructured the physical interface bootstrap architecture by adding explicit firewall rules for outbound traffic over `53/udp`. This opened up standard network address lookups while safely keeping the rest of the physical interface securely blocked.

### Challenge 3: Application Password Obscurity in Background Daemons
*   **The Symptom:** Upon deploying the `qbittorrent-nox` package as a system background service (`Type=forking`), the critical, securely randomized default web administrator password was completely missing from system logs (`journalctl`), preventing first-time access.
*   **The Root Cause Analysis:** When a software binary forks itself manually using the background daemon flag (`-d`), it detaches from the systemd standard controller output wrapper. Consequently, the initialization stdout stream (which contains the one-time secure password) prints out into an untracked process memory vacuum instead of routing to the system logger.
*   **The Solution:** Refactored the custom Linux initialization runtime configuration (`qbittorrent-nox.service`). Altered the system execution topology from `Type=forking` to **`Type=simple`** and removed the detached background flag from the execution binary arguments. This forced the running software process to stay actively attached to systemd, cleanly streaming its entire terminal startup sequence and secure access tokens directly into the accessible system log registers.

### Challenge 4: Missing Local USB Drive Mappings Across System Boot Sequences
*   **The Symptom:** The attached 1TB external storage drive failed to mount to standard `/media/` directories automatically, causing file download scripts to break on system reboots.
*   **The Root Cause Analysis:** Headless Linux server operating systems do not include automated graphic-desktop volume-mounting services (like udisks2). Additionally, using generic hardware block naming conventions (like `/dev/sdb2`) in static configuration files is unreliable, as drive labels can dynamically shift letters depending on the order devices are physically initialized during a reboot.
*   **The Solution:** Used low-level block identification parameters (`blkid`) to pinpoint the raw partition's immutable **UUID hardware string signature**. Tucked this unique ID directly into the permanent system file layout table (`/etc/fstab`) with a `defaults,nofail` flag array. This guarantees the 1TB drive safely and predictably maps onto the same directory path (`/mnt/usb-storage`) every single time the MacBook powers on, without stalling the host operating system if the drive is ever unplugged.

---

## ⚖️ Legal, Ethical, and Compliance Notice

This repository is presented solely as an educational technical framework for systems hardening, local area network micro-segmentation, and secure data pipeline orchestration. 

Users are strictly advised and reminded to utilize peer-to-peer file sharing applications **responsibly, safely, and ethically** . All downloading and hosting operations must align with your respective federal, state, regional, and local copyright laws . The developer of this guide does not condone, promote, or assume liability for the unauthorized distribution, reproduction, or acquisition of copyrighted intellectual property.

---

## 📂 Repository Blueprint
*   `README.md` - Technical portfolio overview, architectural review, and engineering troubleshooting journal (this file).
*   [`DEPLOYMENT.md`](./DEPLOYMENT.md) - **The Complete Guide:** Chronological, step-by-step production manual mapping out the server build from bare-metal up to final verification.
