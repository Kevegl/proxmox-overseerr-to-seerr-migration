# 🚀 Overseerr to Seerr Migration Guide (Proxmox LXC)
Last validated: February 2026 | Node.js v22 | Seerr v3.0.1

This guide provides a proven, step-by-step walkthrough to migrate from the deprecated **Overseerr** to its active fork **Seerr** within a Proxmox LXC environment.

---

## 📖 Table of Contents
* [⚠️ Prerequisites](#️-prerequisites--troubleshooting)
* [🛠 Step 1: Prepare the LXC](#-step-1-prepare-the-lxc)
* [⚙️ Step 2: System Setup](#️-step-2-system-setup)
* [📦 Step 3: Migration & Data Transfer](#-step-3-migration--data-transfer)
* [🏗️ Step 4: Build from Source](#️-step-4-build-from-source)
* [⚖️ Step 5: Finalize Permissions](#️-step-5-finalize-permissions)
* [🚀 Step 6: Service Configuration](#-step-6-service-configuration)
* [🏁 Step 7: Post-Migration Steps](#-step-7-post-migration-steps)

---

## ⚠️ Prerequisites & Troubleshooting

The build process for Seerr is resource-heavy. Before starting, ensure your LXC meets these requirements:

* **Disk Space:** Minimum **20GB** free space is highly recommended. (Build fails with `ERR_PNPM_ENOSPC` if space is insufficient).
* **RAM (Build Phase):** Temporarily increase to **8GB (8192 MiB)** if your host system allows it. Minimum 4GB + Swap is required to avoid "Out of Memory" errors.
* **Node.js:** Version **22.x** is required.

---

## 🛠 Step 1: Prepare the LXC

1. **Shutdown Old LXC:** Power off your original Overseerr LXC (e.g., ID 106) to prevent IP and Tailscale conflicts.
2. **Clone or Backup:** Create a clone of the offline LXC to a new ID (e.g., 115).
3. **Handle Bind Mounts:** If cloning fails due to `mp0` (mount points), you must edit the container's configuration file **on the Proxmox Host (Node)**.
   * Path: `/etc/pve/lxc/106.conf`
   * Action: Comment out the line starting with `mp0:` (add a `#` at the beginning).
4. **Adjust Resources:** In Proxmox, increase **Root Disk** (min. 20GB), **Memory** (e.g., 8192 MiB), and **Swap** (e.g., 2048 MiB) based on your host's capacity.
5. **Start New LXC:** Power on the new LXC (115).

---

## ⚙️ Step 2: System Setup

```bash
# Install Node.js 22 and Build Essentials
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt-get install -y nodejs build-essential git

# Install Pnpm globally
npm install -g pnpm
```

---

## 📦 Step 3: Migration & Data Transfer

```bash
cd /opt
mv overseerr overseerr_old

# Clone the repository
git clone https://github.com/seerr-team/seerr.git overseerr

# Copy your existing database and settings
cp -r /opt/overseerr_old/config /opt/overseerr/
```

---

## 🏗️ Step 4: Build from Source

```bash
cd /opt/overseerr
CYPRESS_INSTALL_BINARY=0 pnpm install --frozen-lockfile

# Adjust the max-old-space-size according to your assigned RAM (e.g., 4096 for 8GB RAM)
export NODE_OPTIONS="--max-old-space-size=4096"

# Run the build process
pnpm build
```

---

## ⚖️ Step 5: Finalize Permissions

```bash
# Set ownership (assuming root)
chown -R root:root /opt/overseerr

# Ensure config directory is writable
chmod -R 755 /opt/overseerr/config
```

---

## 🚀 Step 6: Service Configuration

1. Edit the service file: `nano /etc/systemd/system/overseerr.service`
2. Update the content:

```ini
[Unit]
Description=Seerr Service
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/overseerr
ExecStart=/usr/bin/node dist/index.js
Restart=always
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

3. **Reload and start the service:**

```bash
systemctl daemon-reload
systemctl enable overseerr
systemctl restart overseerr
```

---

## 🏁 Step 7: Post-Migration Steps

1. **Verify:** Open `http://[LXC-IP]:5055` in your browser.
2. **Scale Down:** Reduce LXC RAM back to its normal operating size (e.g., **2048 MiB**).
3. **Prevent Auto-Start:** Set **old LXC (106) -> Options -> Start at boot** to `No`.
4. **IP Reassignment:** Assign the old static IP to the new LXC if desired.
5. **Restore Bind Mounts (mp0):** To restore your media mount from the old container to the new one, run this command **on the Proxmox Host shell**:
   ```bash
   # Copy the mount point line from ID 106 to ID 115
   grep "mp0" /etc/pve/lxc/106.conf >> /etc/pve/lxc/115.conf
   ```
6. **Tailscale:** Connection will resume. Keep the old LXC off.
7. **Cleanup:** Once stable, you can remove old data: `rm -rf /opt/overseerr_old`.
