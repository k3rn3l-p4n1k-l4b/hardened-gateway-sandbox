# Hardened Egress Gateway & Automated Secure Ingestion Sandbox

An architectural blueprint for repurposing an old, upgraded MacBook Pro (yes i had 2) into an isolated, Data Ingestion Application. This repository presents an enterprise-grade approach to network engineering, endpoint hardening, and automated pipeline security. 

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
*   **Software Layer:** The data ingestion application is hard-bound exclusively to the virtual network adapter interface (`tun0`), rendering it incapable of initiating sockets over raw wires.
*   **OS Firewall Layer (`ufw`):** System-wide firewall policies default to a hard `DENY` for incoming and outgoing data, strictly restricting out-of-tunnel physical data transmissions.
*   **Hardware Gateway Layer (SDN ACL):** A centralized network gateway policy enforces stateful traffic packet drops at the physical home router border if the host operating system is ever compromised.

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
*   **The Symptom:** Upon deploying the Data Ingestion Application as a system background service (`Type=forking`), the critical, securely randomized default web administrator password was completely missing from system logs (`journalctl`), preventing first-time access.
*   **The Root Cause Analysis:** When a software binary forks itself manually using the background daemon flag (`-d`), it detaches from the systemd standard controller output wrapper. Consequently, the initialization stdout stream (which contains the one-time secure password) prints out into an untracked process memory vacuum instead of routing to the system logger.
*   **The Solution:** Refactored the custom Linux initialization runtime configuration (`Data Ingestion Application`). Altered the system execution topology from `Type=forking` to **`Type=simple`** and removed the detached background flag from the execution binary arguments. This forced the running software process to stay actively attached to systemd, cleanly streaming its entire terminal startup sequence and secure access tokens directly into the accessible system log registers.

### Challenge 4: Missing Local USB Drive Mappings Across System Boot Sequences
*   **The Symptom:** The attached 1TB external storage drive failed to mount to standard `/media/` directories automatically, causing file download scripts to break on system reboots.
*   **The Root Cause Analysis:** Headless Linux server operating systems do not include automated graphic-desktop volume-mounting services (like udisks2). Additionally, using generic hardware block naming conventions (like `/dev/sdb2`) in static configuration files is unreliable, as drive labels can dynamically shift letters depending on the order devices are physically initialized during a reboot.
*   **The Solution:** Used low-level block identification parameters (`blkid`) to pinpoint the raw partition's immutable **UUID hardware string signature**. Tucked this unique ID directly into the permanent system file layout table (`/etc/fstab`) with a `defaults,nofail` flag array. This guarantees the 1TB drive safely and predictably maps onto the same directory path (`/mnt/usb-storage`) every single time the MacBook powers on, without stalling the host operating system if the drive is ever unplugged.

---

## 📂 Repository Blueprint
*   `README.md` - Technical portfolio overview, architectural review, and engineering troubleshooting journal (this file).
*   [`DEPLOYMENT.md`](./DEPLOYMENT.md) - **The Complete Guide:** Chronological, step-by-step production manual mapping out the server build from bare-metal up to final verification.

### 🚀 Post-Deployment Retrospective: Physical Rack Migration & Lifecycle Edge Cases

Following the successful initial deployment of the server node, the machine was physically disconnected from the testing bench, mounted into a permanent server rack frame, and subjected to a cold reboot. This real-world physical migration immediately surface-tested the system's infrastructure integrity, exposing two critical post-deployment architecture edge cases that were systematically investigated, diagnosed, and resolved.

#### 1. Management Lockout & Tunnel Routing Hijacks (Cross-Subnet Gateway Conflict)
*   **The Symptom:** Upon booting up inside the server rack, the server successfully initialized its firewalls and VPN tunnel, but the Admin Workstation sitting on a separate management subnet pool (`XXX.XXX.XXX.XXX/32`) instantly lost all SSH and Web UI dashboard access.
*   **The Root Cause Analysis:** Forcing the system default internet gateway route to point out of the virtual tunnel interface (`tunX` with a priority metric) worked perfectly for external internet isolation. However, because the Admin Workstation was located on a different network subnet, the server attempted to route its management responses back to the admin machine using that new default path. Instead of passing the data back across the local physical interface wire, the operating system pushed local management packets out through the VPN tunnel. The public VPN node flagged local internal subnet payloads as invalid and dropped them, causing a total management communication block.
*   **The Solution:** Resolved this by manually injecting a static host route exception into the routing engine table: `sudo ip route add XXX.XXX.XXX.XXX/32 via YYY.YYY.YYY.YYY dev XX_physical_nic_XX`. This instructs the kernel to explicitly route responses bound for the exact Admin PC IP back through the local physical interface and hardware gateway, bypassing the VPN tunnel entirely. To automate this permanently across reboots and IP renewals, this static exception was written into an execution delay script (`route-delay 2`) tied directly into the VPN client's native initialization hook structure.

#### 2. The "Missing Files" File Handler Panic (Dynamic Hardware Letter Drifts)
*   **The Symptom:** After the physical rack power cycle, the Data Ingestion Application Web UI transfer panel flagged active downloads with a red exclamation point and locked up under a `Missing files` status error.
*   **The Root Cause Analysis:** Because the external USB storage array was physically disconnected and re-connected during the rack installation, the Linux kernel initialized the storage controllers in a different sequence on boot. This caused the storage block locations to dynamically swap letters (shifting the 1TB partition label from `/dev/sdXX` over to a different target node block). While our permanent hardware table layout (`/etc/fstab`) successfully tracked the change using the disk's unique **UUID** and mapped it to the correct path, the data ingestion background daemon initialized a few milliseconds *before* the storage mount sequence finished. Finding an empty directory path, the application assumed the files had been hard-deleted and panicked, throwing a protective file-handler lock to prevent corrupted chunk writes.
*   **The Solution:** Restructured the recovery sequence into an optimized three-tier workflow:
    1.  Executed `sudo mount -a` to force an immediate system table re-read and mount confirmation.
    2.  Bounced the background application memory (`sudo systemctl restart Data Ingestion Application`) to clear the dead process logs and force the daemon to recognize the newly mapped storage directories.
    3.  Triggered an interactive application **`Force Recheck`** directly inside the Web UI dashboard panel. This forced the core software engine to recursively scan the blocks inside the isolated `/mnt/usb-storage/data_transfer_in_progress/` directory, verify the exact data percentage bit-by-bit, and seamlessly shift the torrent status back into a functional, optimized `Downloading` loop.
