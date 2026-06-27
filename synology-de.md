
---

# NetBird auf Synology NAS — Deinstallation, Reset & Neuregistrierung

> Ergänzte Anleitung basierend auf der offiziellen Dokumentation (docs.netbird.io/get-started/install/synology).  
> Die offizielle Anleitung ist unvollständig: der laufende Daemon-Prozess wird bei der Deinstallation nicht beendet, was bei einer Neuinstallation zu Fehlern führt.

---

## Voraussetzungen

SSH-Zugang zur Synology aktivieren:  
*Control Panel → Terminal & SNMP → Terminal → „Enable SSH Service" aktivieren*

Verbinden und Root-Rechte erlangen:
```bash
ssh user@192.168.178.10
sudo -i
```

---

## 1. Deinstallation (vollständig)

### Schritt 1 — NetBird-Verbindung trennen
```bash
netbird down
```

### Schritt 2 — Service deinstallieren
```bash
netbird service stop
netbird service uninstall
```

> **Wichtig (fehlt in der offiziellen Anleitung):**  
> `netbird service uninstall` beendet den bereits laufenden Daemon-Prozess **nicht**.  
> Er läuft als Waise im Speicher weiter und blockiert die Neuinstallation.

### Schritt 3 — Laufenden Prozess prüfen und beenden
```bash
ps aux | grep netbird
```

Wenn ein Prozess wie dieser erscheint:
```
root  32693  ... /usr/local/bin/netbird service run ...
```

Diesen hart beenden (PID aus der Ausgabe verwenden):
```bash
kill -9 32693
```

Nochmals prüfen ob der Prozess wirklich weg ist:
```bash
ps aux | grep netbird
```

Nur der `grep`-Prozess selbst darf noch erscheinen.

### Schritt 4 — Binary und Konfiguration entfernen
```bash
rm /usr/local/bin/netbird
rm -rf /var/lib/netbird/
```

### Schritt 5 — Alten Peer im NetBird-Dashboard löschen
*app.netbird.io → Peers → Diskstation → Delete*

> Bei einer Neuinstallation wird ein neuer WireGuard-Keypair generiert und ein neuer Peer angelegt. Der alte Peer bleibt sonst als inaktiver Eintrag im Dashboard stehen.

---

## 2. Neuinstallation

### Schritt 1 — tun-Modul prüfen
```bash
ls -la /dev/net/tun
```

Erwartete Ausgabe:
```
crw------- 1 root root 10, 200 ... /dev/net/tun
```

Falls das Verzeichnis oder die Datei fehlt, manuell anlegen:
```bash
if [ ! -d /dev/net ]; then mkdir -m 755 /dev/net; fi
mknod /dev/net/tun c 10 200
chmod 0755 /dev/net/tun
insmod /lib/modules/tun.ko
```

### Schritt 2 — NetBird installieren
```bash
curl -fsSL https://pkgs.netbird.io/install.sh | sudo sh
```

### Schritt 3 — Service starten
```bash
netbird service install
netbird service start
```

### Schritt 4 — Mit Setup Key verbinden
```bash
netbird up --setup-key DEIN-SETUP-KEY
```

Erwartete Ausgabe: `Connected`

### Schritt 5 — Verbindung prüfen
```bash
netbird status
```

```bash
ip addr show wt0
```

---

## 3. Reboot-Script einrichten (wichtig!)

Das tun-Modul wird nach einem DSM-Neustart nicht automatisch geladen.  
Ohne dieses Script verbindet sich NetBird nach einem Reboot nicht.

*Control Panel → Task Scheduler → Create → Triggered Task → User defined script*

- **Task Name:** NetBird Reboot
- **User:** root  
- **Event:** Boot-up  
- **Enable:** ✓

Script unter *Task Settings → Run command → User-defined script*:

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

## 4. Neue Peer-Gruppe zuweisen

Nach erfolgreicher Verbindung im NetBird-Dashboard:

*app.netbird.io → Peers → neuen Diskstation-Peer → Groups → „servers" zuweisen*

---

## Bekannte Probleme

**Problem:** Synology als Routing Peer für das eigene Subnetz (`192.168.178.0/24`) führt zu Routing-Konflikten.  
**Symptom:** QuickConnect und DDNS der Synology nicht mehr erreichbar.  
**Lösung:** Synology **nicht** als Routing Peer eintragen. Nur Home Assistant als einzigen Routing Peer für `192.168.178.0/24` verwenden. Die Synology ist über HA als Gateway trotzdem vollständig erreichbar.

---

## Zusammenfassung der Lücken in der offiziellen Anleitung

| Thema | Offiziell | Ergänzt |
|---|---|---|
| Daemon-Prozess bei Deinstallation | Nicht erwähnt | `kill -9 <PID>` nach `service uninstall` |
| Alter Peer im Dashboard | Erwähnt | Vor Neuinstallation löschen empfohlen |
| Routing-Konflikt | Nicht erwähnt | Synology nicht als Routing Peer |
| tun-Modul vor Installation prüfen | Nicht erwähnt | `ls -la /dev/net/tun` |
