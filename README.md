# 🛠️ RedHat/Fedora: Static IP & Hostname Setup 

Steps for configuring a persistent static IP and changing the system hostname on RHEL/Fedora-based servers. Uses `nmcli` and `hostnamectl` for reliable changes, with exact commands, expected outputs, and targeted fixes for common edge cases.

---

## 🔑 Example Values Used
| Setting | Value |
|---------|-------|
| Connection | `ens192` |
| IP/Subnet | `10.0.0.50/24` |
| Gateway | `10.0.0.1` |
| DNS | `8.8.8.8,1.1.1.1` |
| Hostname (FQDN) | `<new-hostname-here>.example.com` |
| Short Hostname | `<new-hostname-here>` |

---

## 1️⃣ Find Your Connection Name
```bash
nmcli connection show
```
**Expected Output:**
```text
NAME      UUID                                  TYPE      DEVICE
ens192    5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03  ethernet  ens192
lo        00000000-0000-0000-0000-000000000000  loopback  lo
```
✅ Use the exact `NAME` column value. We'll use `ens192`.

---

## 2️⃣ Set Static IP
> 💡 `modify` commands return **no output** on success. Errors only appear if syntax/values are wrong.

```bash
sudo nmcli connection modify ens192 ipv4.addresses 10.0.0.50/24
```
```bash
sudo nmcli connection modify ens192 ipv4.gateway 10.0.0.1
```
```bash
sudo nmcli connection modify ens192 ipv4.dns "8.8.8.8,1.1.1.1"
```
```bash
sudo nmcli connection modify ens192 ipv4.method manual
```
## ✅Apply
```bash
sudo nmcli connection modify ens192 connection.autoconnect yes
```
```bash
sudo nmcli connection up ens192
```
**Expected Output (`up` only):**
```text
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/3)
```

🔹 *Note on `nmcli connection down`:* It's intentionally omitted. `modify` updates the saved profile, and `up` safely reloads it without a manual drop. Adding `down` causes an unnecessary network gap. Use it only if troubleshooting stuck states.

**One-Liner Alternative:**
```bash
sudo nmcli con mod ens192 ipv4.addresses 10.0.0.50/24 ipv4.gateway 10.0.0.1 ipv4.dns "8.8.8.8,1.1.1.1" ipv4.method manual connection.autoconnect yes && sudo nmcli con up ens192
```

---

## 🏠Change Hostname
```bash
sudo hostnamectl set-hostname <new-hostname-here>
sudo systemctl restart sshd
```
```bash
sudo nano /etc/hosts   # or: sudo vi /etc/hosts
```
**Edit the `127.0.0.1` line to:**
```text
127.0.0.1  localhost localhost.localdomain web01.example.com web01
```
💡 *Save/Exit:* `nano` → `Ctrl+O` → `Enter` → `Ctrl+X` | `vi` → `Esc` → `:wq` → `Enter`

---

## ✅Verify
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
<new-hostname-here>
```
```bash
ping -c 2 8.8.8.8
```
**Expected:**
```text
2 packets transmitted, 2 received, 0% packet loss
```

---

## 🔁 Revert to DHCP / Default
```bash
sudo nmcli con mod ens192 ipv4.method auto && sudo nmcli con up ens192
sudo hostnamectl set-hostname localhost.localdomain
# Clean /etc/hosts: remove <new-hostname-here>.example.com & <new-hostname-here> from the 127.0.0.1 line
```
---

## ✅ Final Notes
- Run all commands as `root` or with `sudo`.
- Changes are **persistent** across reboots.
- Reboot is **optional** but recommended.
