# 🛠️ Static IP & Hostname Setup (RHEL/Fedora)
Steps for configuring a static IP and changing the hostname on RHEL/Fedora-based systems. It uses nmcli and hostnamectl for reliable, persistent changes, providing exact commands, expected outputs, quick verification steps.

*Copy-paste ready. Uses `ens192` as the working example. Swap values if your network differs.*

## 🔑 Example Values Used
| Setting | Example Value |
|---------|---------------|
| Connection | `ens192` |
| IP/Subnet | `10.0.0.50/24` |
| Gateway | `10.0.0.1` |
| DNS | `8.8.8.8,1.1.1.1` |
| Hostname (FQDN) | `web01.example.com` |

---

## 1️⃣ Find Connection Name
```bash
nmcli connection show
```
**Expected Output:**
```text
NAME      UUID ... TYPE      DEVICE
ens192    a1b2c3 ... ethernet  ens192
lo        000000 ... loopback  lo
```
✅ Use the exact `NAME` column. We'll use `ens192`.

---

## 2️⃣ Set Static IP
*(Note: `modify` commands return **no output** on success)*
```bash
sudo nmcli connection modify ens192 ipv4.addresses 10.0.0.50/24
sudo nmcli connection modify ens192 ipv4.gateway 10.0.0.1
sudo nmcli connection modify ens192 ipv4.dns "8.8.8.8,1.1.1.1"
sudo nmcli connection modify ens192 ipv4.method manual
sudo nmcli connection modify ens192 connection.autoconnect yes
sudo nmcli connection up ens192
```
**Expected Output (`up` only):**
```text
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/3)
```

**One-Liner Alternative:**
```bash
sudo nmcli con mod ens192 ipv4.addresses 10.0.0.50/24 ipv4.gateway 10.0.0.1 ipv4.dns "8.8.8.8,1.1.1.1" ipv4.method manual connection.autoconnect yes && sudo nmcli con up ens192
```

---

## 3️⃣ Change Hostname
```bash
sudo hostnamectl set-hostname web01.example.com
```
```bash
sudo nano /etc/hosts   # or: sudo vi /etc/hosts
```
**Edit this line to:**
```text
127.0.0.1  localhost  web01.example.com  web01
```
💡 *Save/Exit:* `nano` → `Ctrl+O` → `Enter` → `Ctrl+X` | `vi` → `Esc` → `:wq` → `Enter`

---

## 4️⃣ Verify
```bash
ip -4 addr show ens192
```
**Expected:**
```text
2: ens192: ... UP ...
    inet 10.0.0.50/24 brd 10.0.0.255 scope global noprefixroute ens192
```
```bash
hostname -f
```
**Expected:**
```text
web01.example.com
```
```bash
ping -c 2 8.8.8.8
```
**Expected:**
```text
2 packets transmitted, 2 received, 0% packet loss
```

---

## 🔁 Revert to DHCP
```bash
sudo nmcli con mod ens192 ipv4.method auto && sudo nmcli con up ens192
sudo hostnamectl set-hostname localhost.localdomain
```
---
✅ **Done.** Reboot optional but recommended for full service propagation.
