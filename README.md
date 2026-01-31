# 📘 Hytale Dedicated Server for Proxmox LXC (Debian 12)

This guide explains **how to setup** a dedicated Hytale-Server on an **unprivileged Proxmox LXC** with:

- Stable Autostart
- Working Proxmox-Console
- Clean Network
- Crash-Recovery
- Update-Workflow

---

## 1️⃣ Create LXC (Proxmox)

### Template
- **Debian 12 Standard**

### Recommended Resources
- **CPU:** 8-16 Cores (depending on your hardware)
- **RAM:** 12-16 GB
- **Swap:** 0
- **Disk:** ≥ 16 GB
- **Unprivileged:** Yes
- **Hostname:** `hytale-server` (or whatever you like)
- **Features:** `nesting=yes`
- **Network:** DHCP (vmbr0)

> Start the container after creation.

---

## 2️⃣ Basic Container Setup

Select your Server-Node, open the Node-Console, and enter the container:

```bash
pct enter <containerID>
```

Enable the console and configure networking:

```bash
systemctl enable console-getty.service
systemctl add-wants multi-user.target console-getty.service

apt update && apt upgrade -y
apt purge -y ifupdown
systemctl mask networking.service
systemctl enable systemd-networkd
mkdir -p /etc/systemd/network
nano /etc/systemd/network/10-eth0.network
```

Paste the following configuration into the file:

```ini
[Match]
Name=eth0

[Network]
DHCP=yes
IPv6AcceptRA=yes
```

Save with `CTRL+X`, confirm with `Y`, then `ENTER`.

⚠️ **Action required:** Shutdown the container via Proxmox UI, wait a few seconds, and start it again. 

⚠️ You can now use the Container-Console directly. Log in with root & your password.

Install dependencies:
```bash
apt install -y ca-certificates curl wget unzip gnupg
```

---

## 3️⃣ Install Java (Temurin LTS)

Add the repository and install Java:

```bash
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | gpg --dearmor -o /usr/share/keyrings/adoptium.gpg
echo "deb [signed-by=/usr/share/keyrings/adoptium.gpg] https://packages.adoptium.net/artifactory/deb bookworm main" > /etc/apt/sources.list.d/adoptium.list

apt update
apt install -y temurin-25-jdk
```

Check if Java installation was successful:
```bash
java -version
```

---

## 4️⃣ Create Hytale-Directory & User

```bash
mkdir -p /opt/hytale
useradd -r -m -d /opt/hytale -s /usr/sbin/nologin hytale
chown -R hytale:hytale /opt/hytale
```

---

## 5️⃣ Install Hytale Downloader & Server

```bash
cd /opt/hytale
wget https://downloader.hytale.com/hytale-downloader.zip
unzip hytale-downloader.zip
chmod +x hytale-downloader-linux-amd64
mv hytale-downloader-linux-amd64 hytale-downloader
rm hytale-downloader.zip hytale-downloader-windows-amd64.exe
```

Download the game files:

> ⚠️ **Note:** You will need to authenticate with your Hytale Account before the download starts. Follow the prompots from hytale-downloader!
> 
> Go to [https://oauth.accounts.hytale.com/oauth2/device/verify](https://oauth.accounts.hytale.com/oauth2/device/verify) and enter the code shown in the console.

```bash
./hytale-downloader download-path game.zip
unzip 2026*.zip
rm 2026*.zip
```

---

## 6️⃣ Make script executable

```bash
chmod +x /opt/hytale/start.sh
chown -R hytale:hytale /opt/hytale
```

> ⚠️ **Note:** This is required after every update because the ZIP doesn't preserve the executable bit.

---

## 7️⃣ systemd-Service for Autostart & Crash-Recovery

### Create Service File
```bash
nano /etc/systemd/system/hytale.service
```

### Content
Paste the following:

```ini
[Unit]
Description=Hytale Server
After=network.target systemd-user-sessions.service

[Service]
Type=simple
User=hytale
WorkingDirectory=/opt/hytale
ExecStart=/opt/hytale/start.sh
Restart=always
RestartSec=15
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

Save with `CTRL+X`, confirm with `Y`, then `ENTER`.

### Enable Service
```bash
systemctl daemon-reload
systemctl enable hytale
```

---

## 8️⃣ Reboot-Test 

1. Shutdown Container through Proxmox UI.
2. Wait a few seconds.
3. Start Container again.

Check status:
```bash
systemctl is-system-running
systemctl status hytale  # (CTRL+C to exit)
ip a show eth0
```

**Expected results:**
- System: `running`
- Hytale: `active (running)`
- Network: Has IP

---

## 🔐 9️⃣ Authentication with your Hytale Account

You need to authenticate once to make the server work.

1. Stop the service and run manually:
   ```bash
   systemctl stop hytale
   cd /opt/hytale
   ./start.sh
   ```

2. Wait for the server to boot. You will see:
   > "Hytale has Booted" ... "No server token configured..."

3. In the console, type:
   ```bash
   /auth login device
   ```

4. Go to [https://oauth.accounts.hytale.com/oauth2/device/verify](https://oauth.accounts.hytale.com/oauth2/device/verify) and enter the code shown in the console.

5. After authentication is complete, type in the console:
   ```bash
   /auth persistence Encrypted
   ```
   ➡️ `auth.enc` created. **Never delete this file.**

6. Shutdown the manual server instance:
   ```bash
   /stop
   ```

7. Change File Owner to hytale:
   ```bash
   chown -R hytale:hytale /opt/hytale
   ```

8. Start the service again:
   ```bash
   systemctl start hytale
   ```

🎇🎊 **YOU ARE DONE! Have fun playing Hytale!** 🥳🎉

Default Server-Port is 5520

---

## 🔁 🔟 Test Crash-Recovery (optional)

We are going to crash the server intentionally to see if automatic restart works.

```bash
pkill -9 -f HytaleServer.jar
```

**Expected:** Automatic restart after ~15-30s.

---

## 🔄 1️⃣1️⃣ Update-Workflow

Run these commands to update the server:

```bash
systemctl stop hytale
cd /opt/hytale
./hytale-downloader download-path game.zip
unzip -o 2026*.zip
chmod +x /opt/hytale/start.sh
chown -R hytale:hytale /opt/hytale
rm 2026*.zip
systemctl start hytale
```

---

## 🧠 Important to remember

**Start Server** (Automatic on boot)
```bash
systemctl start hytale
```

**Stop Server**
```bash
systemctl stop hytale
```

**Restart Server**
```bash
systemctl restart hytale
```

**Server Status** (CTRL+C to exit)
```bash
systemctl status hytale
```

**Live-Logs**
```bash
journalctl -u hytale -f
```

**Show last lines of log**
```bash
journalctl -u hytale --no-pager
```

---

## ✅ Config

If you want to change settings (password, servername, etc.):

```bash
systemctl stop hytale
cd /opt/hytale/Server
nano config.json
```

Save with `CTRL+X`, confirm with `Y`, then `ENTER`.

```bash
systemctl start hytale
```

## 💾 Backups

Don't forget to setup your Proxmox backups 😊
I usually go for these settings:

**Proxmox UI:** `Datacenter → Backup → Add`

#### General
- **Schedule:** 04:00 (Daily at 4AM)
- **Mode:** `Suspend`
- **Selection Mode:** `Include selected VMs`  
  *(1 backup job per VM / container)*

#### Retention
- **Daily:** `3`
- **Weekly:** `4`
- **Monthly:** `2`

## Useful Links

https://support.hytale.com/hc/en-us/articles/45326769420827-Hytale-Server-Manual

## Join me!

If you want to join my Creative Hytale Server, feel free to! 

#### Server: 
```bash
tronnic-srv.duckdns.org:5521
```

If you play there and decide to leave at some point, I don't mind exporting the World / Playerdata for you 😃
