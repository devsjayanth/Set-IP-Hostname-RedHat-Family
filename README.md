# 🐧Linux Network Manager Quick Guide
Quick guide to use Network Manager to edit the NIC parameters on linux servers.

⚠️ **Pre-Flight Warning (SSH Users):** Applying network changes over SSH can drop your session if there is a typo. Have console/KVM access available.

---

## Phase 1: Install NetworkManager & Eliminate Conflicts
In minimal Debian, legacy network managers will fight NetworkManager and prevent auto-boot connections. We must disable them.

**1. Install NetworkManager:**
```bash
# Debian / Ubuntu
sudo apt update && sudo apt install -y network-manager

# RHEL / CentOS / Rocky / Fedora
sudo dnf install -y NetworkManager
```

**2. Enable NetworkManager to start on boot:**
```bash
sudo systemctl enable --now NetworkManager
```
Identify your physical interface and check for existing profiles.

```bash
nmcli device status
nmcli connection show
```
---

## Phase 2: Fix "Unmanaged" State & Force Auto-Boot
NetworkManager will ignore interfaces (showing `unmanaged`) or refuse to auto-start if they are defined in legacy config files.

**1. Clean up legacy Debian interfaces file:**
Ensure `/etc/network/interfaces` **only** contains the loopback interface.
```bash
echo -e "auto lo\niface lo inet loopback" | sudo tee /etc/network/interfaces
```

**2. Force NetworkManager to manage all devices:**
```bash
sudo sed -i 's/managed=false/managed=true/g' /etc/NetworkManager/NetworkManager.conf 2>/dev/null || \
echo -e "\n[ifupdown]\nmanaged=true" | sudo tee -a /etc/NetworkManager/NetworkManager.conf
```

**3. Restart NM and force management via CLI:**
```bash
sudo systemctl restart NetworkManager
# Replace 'ens33' with your actual interface name
sudo nmcli device set ens33 managed yes
```

**✅ Verification:**
Run `nmcli device status`. Your interface **must** say `disconnected` or `connected` (NOT `unmanaged`).

---

## Phase 3: Discover Your Network State
Identify your physical interface and check for existing profiles.

```bash
nmcli device status
nmcli connection show
```
* **DEVICE column:** Your physical NIC name (e.g., `ens33`, `eth0`).
* **CONNECTION column:** 
  * Shows a name (e.g., `Wired connection 1`) ➔ Profile **exists**. Go to **Phase 4A**.
  * Shows `--` ➔ No profile exists. Go to **Phase 4B**.

---

## Phase 4: Configure the Static IP & Auto-Connect

Choose **Option A** or **Option B**. *Note the explicit `connection.autoconnect yes` flag, which guarantees it starts on boot.*

### Option A: Modify an EXISTING Connection Profile
```bash
# Replace "Wired connection 1" with your actual profile NAME
nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 10.0.0.140/24 \
  ipv4.gateway 10.0.0.2 \
  ipv4.dns "10.0.0.2 8.8.8.8" \
  ipv4.ignore-auto-dns yes \
  ipv4.may-fail no \
  connection.autoconnect yes
```

### Option B: Create a NEW Connection Profile
```bash
# Replace 'ens33' with your actual DEVICE name
nmcli connection add type ethernet ifname ens33 con-name ens33 \
  ipv4.method manual \
  ipv4.addresses 10.0.0.140/24 \
  ipv4.gateway 10.0.0.2 \
  ipv4.dns "10.0.0.2 8.8.8.8" \
  ipv4.ignore-auto-dns yes \
  ipv4.may-fail no \
  connection.autoconnect yes
```

---

## Phase 5: Apply, Verify & Persist

**1. Activate the connection:**
```bash
nmcli connection up "ens33"  # Use your profile NAME
```

**2. Verify the live configuration:**
```bash
ip -4 addr show ens33
ip route | grep default
ping -c 3 8.8.8.8
```

**3. Verify Auto-Boot Persistence (Crucial):**
Check that the profile is flagged to start automatically:
```bash
nmcli -g connection.autoconnect connection show "ens33"
```
*(It must output `yes`. If it says `no`, run: `nmcli con mod "ens33" connection.autoconnect yes`)*

**4. Disable conflicting legacy network services:**
```bash
sudo systemctl disable --now systemd-networkd 2>/dev/null
sudo systemctl disable --now networking 2>/dev/null  # Debian's ifupdown service
```

---

## 📋 Appendix A: Quick Edit Cheat Sheet
Always replace `"conn-name"` with your profile name.

| Task | Command |
| :--- | :--- |
| **Change IP only** | `nmcli con mod "conn-name" ipv4.addresses 10.0.0.50/24` |
| **Change Gateway** | `nmcli con mod "conn-name" ipv4.gateway 10.0.0.1` |
| **Change DNS** | `nmcli con mod "conn-name" ipv4.dns "1.1.1.1 8.8.8.8"` |
| **Add secondary IP** | `nmcli con mod "conn-name" +ipv4.addresses 10.0.0.51/24` |
| **Set MTU (Jumbo)** | `nmcli con mod "conn-name" 802-3-ethernet.mtu 9000` |
| **Disable IPv6** | `nmcli con mod "conn-name" ipv6.method disabled` |
| **Revert to DHCP** | `nmcli con mod "conn-name" ipv4.method auto ipv4.addresses "" ipv4.gateway ""` |

🔁 **Crucial:** After running *any* command above, apply it: `nmcli connection up "conn-name"`

---

## 🛠 Appendix B: Troubleshooting

**1. The connection still doesn't auto-start on reboot!**
* **Cause:** Another service is grabbing the interface on boot before NM.
* **Fix:** Ensure you ran the commands in Phase 1 Step 2. Double-check that `networking.service` and `systemd-networkd.service` are completely dead:
  ```bash
  systemctl status networking systemd-networkd
  ```
  If they are active, run `sudo systemctl mask networking systemd-networkd` to permanently kill them, then reboot.

**2. I locked myself out over SSH!**
* **Fix:** Access the server via hypervisor console (VMware/Proxmox) or cloud VNC. Revert to DHCP:
  ```bash
  nmcli connection modify "ens33" ipv4.method auto ipv4.addresses "" ipv4.gateway ""
  nmcli connection up "ens33"
  ```

**3. DNS isn't resolving after applying static IP**
* **Cause:** `systemd-resolved` is overriding `/etc/resolv.conf`.
* **Fix:** Force NetworkManager to take over DNS resolution:
  ```bash
  sudo ln -sf /run/NetworkManager/resolv.conf /etc/resolv.conf
  sudo systemctl restart NetworkManager
  ```
