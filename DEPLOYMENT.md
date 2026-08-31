# Chronological Server Deployment Manual

Follow this sequential layout to build the server exactly as it was prototyped.

---

## Step 1: Ubuntu Server Installation
1. Flash **Ubuntu Server 24.04 LTS** onto a bootable USB drive.
2. Insert the USB into the MacBook Pro, power it on, and hold down the **Option (Alt)** key to access the Apple Boot Manager. Select the EFI boot target.
3. Follow the installation prompts:
   * Set a static IP profile (`XXX.XXX.XXX.XXX`) matching your infrastructure layout to prevent DHCP migration.
   * Allocate the full internal SSD capacity to the root filesystem.
   * Explicitly enable the **OpenSSH Server** bundle during the setup process.
4. Finish installation, eject the USB, reboot, and disconnect the monitor/keyboard to run the machine completely headless.

---

## Step 2: OpenVPN Client Configuration
1. Connect to your server from your workstation via SSH (`ssh XXXXX@XXX.XXX.XXX.XXX`).
2. Move your VPN provider's OpenVPN profile to the system configuration directory:
   ```bash
   sudo mv /path/to/your_profile.ovpn /etc/openvpn/XX_vpn_profile_XX.conf
   ```
3. Securely store your VPN Service Credentials (not your account email) to allow automated, unattended connections on boot:
   ```bash
   sudo nano /etc/openvpn/auth.txt
   ```
   * Line 1: Your private VPN Service Username token string.
   * Line 2: Your private VPN Service Password token string.
4. Point your configuration file to the newly created credentials payload:
   ```bash
   sudo nano /etc/openvpn/XX_vpn_profile_XX.conf
   ```
   Find the line `auth-user-pass` and change it to: `auth-user-pass auth.txt`
5. Enable the client service daemon to launch automatically on boot:
   ```bash
   sudo systemctl enable openvpn-client@XX_vpn_profile_XX
   sudo systemctl start openvpn-client@XX_vpn_profile_XX
   ```
   Find instructions for your particular VPN provider.For NordVPN(https://support.nordvpn.com/hc/en-us/articles/20164827795345-How-to-set-up-a-manual-connection-on-Linux-using-OpenVPN)

---
## Step 3: Local UFW Firewall Setup (The OS Killswitch)
Build a strict local firewall around the server. This explicitly forces internet traffic through the virtual VPN interface, while keeping a local administrative bridge open for your single authorized workstation.

```bash
# Reset UFW to clear default rules out completely
sudo ufw reset

# Block all incoming and outgoing traffic by default
sudo ufw default deny incoming
sudo ufw default deny outgoing

# SSH Isolation: Allow SSH IN/OUT solely from your designated Admin Workstation IP
sudo ufw allow in on XX_physical_nic_XX from XXX.XXX.XXX.XXX to any port 22 proto tcp
sudo ufw allow out on XX_physical_nic_XX to XXX.XXX.XXX.XXX

# Web UI Access: Allow your Admin Workstation to reach the future Web UI port
sudo ufw allow in on XX_physical_nic_XX from XXX.XXX.XXX.XXX to any port 8080 proto tcp

# THE NETWORK KILLSWITCH: Allow all outbound internet traffic ONLY through the VPN tunnel
sudo ufw allow out on tunX from any to any

# VPN Bootstrap: Allow the physical wire to establish outbound UDP tunnels with the VPN servers
sudo ufw allow out on XX_physical_nic_XX proto udp

# DNS Bootstrap: Allow physical wire DNS queries so the VPN can resolve initial hostnames
sudo ufw allow out on XX_physical_nic_XX to any port 53 proto udp
sudo ufw allow out on XX_physical_nic_XX to any port 53 proto tcp

# Turn on the firewall permanently
sudo ufw enable
```

---

## Step 4: Network Gateway ACLs (Hardware Controller Safeguard)
Log into your centralized hardware router network controller dashboard to implement a gateway-level hardware killswitch under **Settings > Network Security > ACL > Gateway ACL**.

### Rule 1: Authorize Local Workstation Access (Permit)
*   **Policy:** Permit
*   **Source:** IP Group -> Your specific server host machine machine (`XXX.XXX.XXX.XXX/32`)
*   **Destination:** IP Group -> Your single authorized Admin Workstation (`XXX.XXX.XXX.XXX/32`)

### Rule 2: The Hardware WAN Killswitch (Deny)
*   **Direction:** LAN -> WAN
*   **Policy:** Deny
*   **Protocol:** TCP
*   **Source:** IP Group -> Your specific server host machine (`XXX.XXX.XXX.XXX/32`)
*   **Destination:** IP Group -> `IPGroup_Any` *(Global External Addresses)*

> 💡 **Why TCP Only?** Torrents transfer peer data over TCP connections. By blocking WAN TCP at the gateway router level, your torrent client cannot leak data if the host OS firewall fails. Leaving UDP unblocked lets the OpenVPN daemon reach out to the WAN to find new provider servers and automatically repair the connection.

---

## Step 5: Headless qBittorrent Service Deployment
1. Install the headless application package wrapper:
   ```bash
   sudo apt update && sudo apt install qbittorrent-nox -y
   ```
2. Create a dedicated system management profile for the daemon wrapper:
   ```bash
   sudo nano /etc/systemd/system/qbittorrent-nox.service
   ```
   Paste the following configuration (replace the execution user profile with your own local Linux username):
   ```ini
   [Unit]
   Description=qBittorrent-nox service
   After=network.target openvpn-client@XX_vpn_profile_XX.service

   [Service]
   Type=simple
   User=XXXXXXXX
   ExecStart=/usr/bin/qbittorrent-nox --webui-port=8080 --confirm-legal-notice
   Restart=on-failure

   [Install]
   WantedBy=multi-user.target
   ```
Save and close nano.

3. Initialize and enable the application daemon:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable qbittorrent-nox
   sudo systemctl start qbittorrent-nox
   ```

---

## Step 6: Hard-Binding qBittorrent to the VPN (The Software Killswitch)
1. Read the system startup logs to capture the secure, auto-generated temporary web password:
   ```bash
   sudo journalctl -u qbittorrent-nox -n 30 --no-pager
   ```
2. Open a browser on your Admin PC and navigate to `http://xxx.xxx`. Log in with username `admin` and the captured alphanumeric password.
3. **Change Defaults:** Go immediately to **Tools > Options > Web UI** and set a personalized custom username and password.
4. **Lock the Interface:** Navigate to **Tools > Options > Advanced**:
   * Set **Network interface** explicitly to your VPN interface (e.g., **`tun0`**).
   * Set **Optional IP address to bind to** to **`All IPv4 addresses`**.
5. Click **Save**. The software application is now completely incapable of passing data outside the VPN lane.

---

## Step 7: External USB Drive Storage Setup
1. Mount and format the external USB data hard drive connected to the MacBook Pro machine:
   ```bash
   sudo mkfs.ext4 /dev/sdXX
   sudo mkdir -p /mnt/usb-storage
   sudo mount /dev/sdXX /mnt/usb-storage
   ```
2. Fetch the unique hardware ID (UUID) signature of the storage partition:
   ```bash
   sudo blkid /dev/sdXX
   ```
3. Open `/etc/fstab` and append the hardware mount layout to survive unexpected power losses or reboots safely:
   ```text
   UUID=YOUR_CAPTURED_UUID  /mnt/usb-storage  ext4  defaults,nofail  0  2
   ```
4. Build the data-processing pipeline path structure on the drive and pass local user directory ownership:
   ```bash
   sudo mount -a
   sudo mkdir -p /mnt/usb-storage/torrent_in_progress
   sudo mkdir -p /mnt/usb-storage/torrent_staging
   sudo mkdir -p /mnt/usb-storage/sanitized_complete
   sudo chown -R XXXXXXXX:XXXXXXXX /mnt/usb-storage/
   ```

---

## Step 8: Antivirus Installation & Pipeline Script Setup
1. Pull down the open-source threat scanning engine and update its virus database definitions:
   ```bash
   sudo apt install clamav clamav-daemon -y
   sudo systemctl stop clamav-freshclam
   sudo freshclam
   sudo systemctl start clamav-freshclam
   ```
2. Build the automated execution orchestration script document in your local home directory:
   ```bash
   nano ~/scan_and_move.sh
   ```
   Paste the following processing logic:
   ```bash
   #!/bin/bash
   TORRENT_PATH="$1"
   TORRENT_NAME="$2"

   STAGING_DIR="/mnt/usb-storage/torrent_staging"
   SANITIZED_DIR="/mnt/usb-storage/sanitized_complete"
   LOG_FILE="/home/XXXXXXXX/antivirus_scan.log"

   echo "--- $(date) ---" >> "$LOG_FILE"
   echo "Scanning file: $TORRENT_NAME" >> "$LOG_FILE"

   # Execute a recursive backend threat sweep
   clamscan -r --no-summary "$TORRENT_PATH" >> "$LOG_FILE" 2>&1
   SCAN_EXIT_CODE=$?

   if [ $SCAN_EXIT_CODE -eq 0 ]; then
       echo "CLEAN: $TORRENT_NAME passed. Moving to sanitized folder." >> "$LOG_FILE"
       mv "$TORRENT_PATH" "$SANITIZED_DIR/"
   elif [ $SCAN_EXIT_CODE -eq 1 ]; then
       echo "ALERT: VIRUS DETECTED IN $TORRENT_NAME! Isolating file." >> "$LOG_FILE"
       mv "$TORRENT_PATH" "$STAGING_DIR/${TORRENT_NAME}.INFECTED"
   else
       echo "ERROR: Scanner failed to execute properly on $TORRENT_NAME." >> "$LOG_FILE"
   fi
   ```
3. Save the script and authorize its execution capabilities across the host OS:
   ```bash
   chmod +x ~/scan_and_move.sh
   ```

---

## Step 9: Final qBittorrent Pipeline Mapping

1. Go back to your browser page and load up the web dashboard interface config panel (`http://xxx.xxx`).
2. Navigate to **Tools > Options > Downloads**:
   * Check **Keep incomplete torrents in:** and enter `/mnt/usb-storage/torrent_in_progress`
   * Set **Default Save Path** to `/mnt/usb-storage/torrent_staging`
3. Scroll to the very bottom of the window options page and check the box for **Run external program on torrent completion**.
4. Paste the execution automation shell script path string directly into the text box:
   ```text
   /home/XXXXXXXX/scan_and_move.sh "%F" "%N"
   ```
5. Click **Save**.

---

## 🔬 System Validation & Pipeline Verification

To ensure that your network boundaries are airtight and your sanitization automation scripts run cleanly in production, execute these final verification workflows.

### 1. The Active Socket Leak Audit
While a test download is actively pulling chunks, log into your server via SSH and execute the socket statistics engine:
```bash
sudo ss -tupn | grep qbittorrent
```
* **Success Criteria:** Every single external network socket must present a local connection origin starting with your VPN interface (e.g., `10.100.X.X%tun0`). Your local host management interface must remain isolated exclusively to the local socket bound to your `8080` port.

### 2. The Post-Download Sanitization Verification
Once a download reaches 100% completion, monitor your background orchestration logs to verify the ClamAV threat check and automatic data migration loop:
```bash
cat ~/antivirus_scan.log
```
* **Success Criteria:** The log must print a verification stamp detailing successful payload identification, followed by an immediate filesystem string response showing a safe move operation out of your staging area and into `/mnt/usb-storage/sanitized_complete/`.

Your physical-to-digital sanitization network environment is fully optimized, hard-bound, and structurally verified.


   
