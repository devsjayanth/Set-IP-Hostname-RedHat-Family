# 🐧Linux Network Manager Quick Guide
Here is the ultimate, comprehensive Standard Operating Procedure (SOP). I have significantly expanded the **"Unmanaged State"** section, as this is the #1 reason `nmcli` fails on minimal, cloud, or freshly installed servers.
**Supported Distros:** Debian, Ubuntu, RHEL, CentOS, Rocky, Alma, Fedora.

⚠️ **Pre-Flight Warning (SSH Users):** Applying network changes over SSH can drop your session if there is a typo. Have console/KVM access available, or keep a backup DHCP plan ready (see Troubleshooting).

---

## Phase 1: Install & Enable NetworkManager
NetworkManager is default on RHEL, but often missing on minimal Debian/Ubuntu servers.

**For Debian / Ubuntu:**
```bash
sudo apt update
sudo apt install -y network-manager
sudo systemctl enable --now NetworkManager
```

**For RHEL / CentOS / Rocky / Fedora:**
```bash
# Usually pre-installed, but run to ensure:
sudo dnf install -y NetworkManager
sudo systemctl enable --now NetworkManager
```

---

## Phase 2: Diagnosing & Fixing the "Unmanaged" State (CRITICAL)
If you run `nmcli device status` and see **`unmanaged`** under the STATE column, NetworkManager is explicitly ignoring your hardware. **You cannot assign an IP until this is fixed.**
```
nmcli device status
```
### 🔍 Why does this happen?
1. **Debian/Ubuntu:** The interface is defined in the legacy `/etc/network/interfaces` file, so NM backs off.
2. **RHEL/CentOS:** The legacy `ifcfg-*` script has `NM_CONTROLLED=no`.
3. **Cloud Providers (AWS/GCP/Azure) & Proxmox:** `cloud-init` or custom udev rules mark the primary NIC as unmanaged to prevent NM from breaking the cloud metadata network.

### 🛠 The 4-Step Fix for "Unmanaged" Interfaces

**Step 1: Tell NetworkManager to manage `ifupdown` interfaces (Debian/Ubuntu)**
```bash
sudo sed -i 's/managed=false/managed=true/g' /etc/NetworkManager/NetworkManager.conf 2>/dev/null || \
echo -e "\n[ifupdown]\nmanaged=true" | sudo tee -a /etc/NetworkManager/NetworkManager.conf
```

**Step 2: Remove the interface from legacy configuration files**
*For Debian/Ubuntu:*
```bash
# Replace 'ens33' with your actual interface name
sudo sed -i '/ens33/d' /etc/network/interfaces 2>/dev/null
```
*For RHEL/CentOS (Legacy):*
```bash
# Check if legacy scripts exist and ensure NM_CONTROLLED=yes
sudo sed -i 's/NM_CONTROLLED=no/NM_CONTROLLED=yes/g' /etc/sysconfig/network-scripts/ifcfg-* 2>/dev/null
```

**Step 3: Check for hidden override files (Common in Cloud VMs)**
Cloud images often hide "unmanaged" rules in drop-in directories.
```bash
# Look for files forcing the device to be unmanaged
grep -r "unmanaged" /etc/NetworkManager/conf.d/ /usr/lib/NetworkManager/conf.d/ 2>/dev/null
```
*If you find a file (e.g., `10-globally-managed-devices.conf`), edit it and change `unmanaged-devices` to `none`, or simply delete the file.*

**Step 4: Restart NM and Force Management via CLI**
```bash
sudo systemctl restart NetworkManager

# The magic bullet: Force NM to manage the device via CLI
sudo nmcli device set ens33 managed yes
```

**✅ Verification:**
Run `nmcli device status`. The STATE for your interface **must** now say `disconnected` or `connected`. If it still says `unmanaged`, reboot the server.

---

## Phase 3: Discover Your Network State
Now that the device is managed, identify your physical interface and check for existing profiles.

```bash
nmcli device status
nmcli connection show
```

**How to read the output:**
* **DEVICE column:** Your physical NIC name (e.g., `ens33`, `eth0`).
* **CONNECTION column:** 
  * If it shows a name (e.g., `Wired connection 1`), a profile **exists**. ➔ Go to **Phase 4A**.
  * If it shows `--`, no profile exists. ➔ Go to **Phase 4B**.

---

## Phase 4: Configure the Static IP

Choose **Option A** or **Option B** based on your findings in Phase 3.

### Option A: Modify an EXISTING Connection Profile
*Use this if `nmcli connection show` listed a profile name for your device.*

```bash
# Replace "Wired connection 1" with your actual profile NAME
nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 10.0.0.140/24 \
  ipv4.gateway 10.0.0.2 \
  ipv4.dns "10.0.0.2 8.8.8.8" \
  ipv4.ignore-auto-dns yes \
  ipv4.may-fail no
```

### Option B: Create a NEW Connection Profile
*Use this if you got `Error: unknown connection` or the CONNECTION column showed `--`.*

```bash
# Replace 'ens33' with your actual DEVICE name
nmcli connection add type ethernet ifname ens33 con-name ens33 \
  ipv4.method manual \
  ipv4.addresses 10.0.0.140/24 \
  ipv4.gateway 10.0.0.2 \
  ipv4.dns "10.0.0.2 8.8.8.8" \
  ipv4.ignore-auto-dns yes \
  connection.autoconnect yes
```

*(Note: Update the IP, Subnet CIDR `/24`, Gateway, and DNS to match your specific network.)*

---

## Phase 5: Apply and Verify

**1. Activate the connection:**
```bash
# Use the profile NAME (e.g., "Wired connection 1" or "ens33")
nmcli connection up "ens33"
```

**2. Verify the configuration:**
```bash
# Check that the device is now "connected"
nmcli device status

# Verify the IP is assigned to the interface
ip -4 addr show ens33

# Verify routing and DNS
ip route | grep default
cat /etc/resolv.conf

# Test connectivity
ping -c 3 10.0.0.2
ping -c 3 8.8.8.8
```

---

## 📋 Appendix A: Quick Edit Cheat Sheet
Once your profile is created, use these quick commands for targeted edits. Always replace `"conn-name"` with your profile name.

| Task | Command |
| :--- | :--- |
| **Change IP only** | `nmcli con mod "conn-name" ipv4.addresses 10.0.0.50/24` |
| **Change Gateway** | `nmcli con mod "conn-name" ipv4.gateway 10.0.0.1` |
| **Change DNS** | `nmcli con mod "conn-name" ipv4.dns "1.1.1.1 8.8.8.8"` |
| **Add secondary IP** | `nmcli con mod "conn-name" +ipv4.addresses 10.0.0.51/24` |
| **Remove secondary IP** | `nmcli con mod "conn-name" -ipv4.addresses 10.0.0.51/24` |
| **Set MTU (Jumbo)** | `nmcli con mod "conn-name" 802-3-ethernet.mtu 9000` |
| **Disable IPv6** | `nmcli con mod "conn-name" ipv6.method disabled` |
| **Revert to DHCP** | `nmcli con mod "conn-name" ipv4.method auto ipv4.addresses "" ipv4.gateway ""` |

🔁 **Crucial:** After running *any* command above, you must apply it:
```bash
nmcli connection up "conn-name"
```

---

## 🛠 Appendix B: Troubleshooting & Recovery

**1. I locked myself out over SSH!**
* **Fix:** Access the server via physical console, hypervisor VM console (VMware/Proxmox), or cloud provider VNC/Web Console. Run the following to revert to DHCP and get back online:
  ```bash
  nmcli connection modify "ens33" ipv4.method auto ipv4.addresses "" ipv4.gateway ""
  nmcli connection up "ens33"
  ```

**2. "Error: Device 'ens33' not managed" when running `connection up`**
* **Cause:** The "unmanaged" fix in Phase 2 didn't fully clear the cache.
* **Fix:** Run `sudo nmcli device set ens33 managed yes`, followed by `sudo systemctl restart NetworkManager`, then try `nmcli connection up "ens33"` again.

**3. DNS isn't resolving after applying static IP**
* **Cause:** `systemd-resolved` or `resolvconf` is overriding `/etc/resolv.conf`.
* **Fix:** Force NetworkManager to take over DNS resolution:
  ```bash
  sudo ln -sf /run/NetworkManager/resolv.conf /etc/resolv.conf
  sudo systemctl restart NetworkManager
  ```
