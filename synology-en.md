
---

# NetBird on Synology NAS — Uninstallation, Reset & Re-Registration

> Supplementary guide based on the official documentation (docs.netbird.io/get-started/install/synology).
> The official guide is incomplete: the running daemon process is not terminated during uninstallation, which causes errors during reinstallation.

---

## Prerequisites

Enable SSH access on your Synology:
*Control Panel → Terminal & SNMP → Terminal → Check „Enable SSH Service" → Apply*

Connect via SSH and gain root privileges:
```bash
ssh user@192.168.178.10
sudo -i
```

---

## 1. Uninstallation (complete)

### Step 1 — Disconnect NetBird
```bash
netbird down
```

### Step 2 — Uninstall the service
```bash
netbird service stop
netbird service uninstall
```

> **Important (missing from the official guide):**
> `netbird service uninstall` does **not** terminate the already running daemon process.
> It continues running as an orphan in memory and will block a fresh installation.

### Step 3 — Check for and kill the orphaned process
```bash
ps aux | grep netbird
```

If a process like this appears:
```
root  32693  ... /usr/local/bin/netbird service run ...
```

Terminate it using the PID from the output:
```bash
kill -9 32693
```

Verify the process is gone — only the grep process itself should remain:
```bash
ps aux | grep netbird
```

### Step 4 — Remove binary and configuration files
```bash
rm /usr/local/bin/netbird
rm -rf /var/lib/netbird/
```

### Step 5 — Delete the old peer from the NetBird dashboard
*app.netbird.io → Peers → Diskstation → Delete*

> A fresh installation generates a new WireGuard keypair and registers a new peer. The old peer will otherwise remain as an inactive entry in the dashboard.

---

## 2. Reinstallation

### Step 1 — Verify the tun module
```bash
ls -la /dev/net/tun
```

Expected output:
```
crw------- 1 root root 10, 200 ... /dev/net/tun
```

If the file or directory is missing, create it manually:
```bash
if [ ! -d /dev/net ]; then mkdir -m 755 /dev/net; fi
mknod /dev/net/tun c 10 200
chmod 0755 /dev/net/tun
insmod /lib/modules/tun.ko
```

### Step 2 — Install NetBird
```bash
curl -fsSL https://pkgs.netbird.io/install.sh | sudo sh
```

### Step 3 — Start the service
```bash
netbird service install
netbird service start
```

### Step 4 — Connect using a Setup Key
```bash
netbird up --setup-key YOUR-SETUP-KEY
```

Expected output: `Connected`

### Step 5 — Verify the connection
```bash
netbird status
ip addr show wt0
```

---

## 3. Configure the Reboot Script (important!)

The tun module is not loaded automatically after a DSM reboot.
Without this script, NetBird will not reconnect after a restart.

*Control Panel → Task Scheduler → Create → Triggered Task → User defined script*

- **Task Name:** NetBird Reboot
- **User:** root
- **Event:** Boot-up
- **Enable:** checked

Paste the following script under *Task Settings → Run command → User-defined script*:

```sh
#!/bin/sh

if [ ! -c /dev/net/tun ]; then
  if [ ! -d /dev/net ]; then
    mkdir -m 755 /dev/net
  fi
  mknod /dev/net/tun c 10 200
  chmod 0755 /dev/net/tun
fi

if !(lsmod | grep -q "^tun\s"); then
  insmod /lib/modules/tun.ko
fi
```

---

## 4. Assign the new peer to a group

After a successful connection, go to the NetBird dashboard:

*app.netbird.io → Peers → new Diskstation peer → Groups → assign „servers"*

---

## Known Issues

**Problem:** Configuring the Synology as a routing peer for its own subnet (`192.168.178.0/24`) causes routing conflicts.
**Symptom:** Synology QuickConnect and DDNS become unreachable from the internet.
**Solution:** Do **not** add the Synology as a routing peer. Use only Home Assistant as the single routing peer for `192.168.178.0/24`. The Synology remains fully accessible as a resource through HA as the gateway.

> This issue has been reported at [GitHub Issue #4950](https://github.com/netbirdio/netbird/issues/4950).

---

## Summary of gaps in the official documentation

| Topic | Official docs | This guide |
|---|---|---|
| Daemon process during uninstall | Not mentioned | `kill -9 <PID>` required after `service uninstall` |
| Old peer in dashboard | Mentioned | Recommended to delete before reinstalling |
| Routing conflict | Not mentioned | Do not use Synology as routing peer |
| Verify tun module before install | Not mentioned | `ls -la /dev/net/tun` |
