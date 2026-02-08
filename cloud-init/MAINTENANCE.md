# NTP Server Wartung & Monitoring

## Wartungsfenster

### Empfohlenes Monitoring-Wartungsfenster

```
Taeglich 02:55 - 03:35 UTC
```

Alle Wartungsaktivitaeten (Paketlisten, Upgrades, Dienst-Neustarts, Reboot)
sind in dieses Fenster gelegt. Ausserhalb dieses Fensters finden keine
geplanten Unterbrechungen statt.

### Zeitplan der automatischen Prozesse

| Zeit (UTC) | Prozess | NTP offline? | Haeufigkeit |
|---|---|---|---|
| 03:00 | `apt-daily.timer` (Paketlisten aktualisieren) | Nein | Taeglich |
| 03:10 | `apt-daily-upgrade.timer` (Pakete installieren) | Nein* | Taeglich |
| 03:25 | Automatischer Reboot (falls noetig) | **Ja, 1-2 Min** | 1-2x/Monat |

*\* Bei einem chrony-Paketupdate wird der Dienst durch needrestart kurz
neu gestartet (~1 Sek). Das ist fuer NTP-Clients transparent.*

Alle Timer laufen ohne `RandomizedDelaySec` — die Zeiten sind exakt,
nicht zufaellig verteilt. Konfiguriert ueber systemd Timer-Overrides in:

- `/etc/systemd/system/apt-daily.timer.d/override.conf`
- `/etc/systemd/system/apt-daily-upgrade.timer.d/override.conf`

### Wann wird tatsaechlich rebootet?

Ein Reboot um 03:25 UTC findet **nur** statt, wenn:
- Ein Kernel-Update installiert wurde (typisch 1-2x pro Monat)
- Ein anderes Paket einen Reboot erfordert (selten)

Die Datei `/var/run/reboot-required` zeigt an, ob ein Reboot ansteht:
```bash
cat /var/run/reboot-required        # Existiert nur wenn Reboot noetig
cat /var/run/reboot-required.pkgs   # Welches Paket den Reboot ausgeloest hat
```

---

## Monitoring-Checks

### NTP-Dienst pruefen

```bash
# Ist chrony synchronisiert?
chronyc tracking

# Upstream-Quellen und deren Status
chronyc sources -v

# Anzahl aktiver Clients (letzte Stunde)
chronyc clients

# Serverstats (Pakete beantwortet, gedroppt, etc.)
chronyc serverstats
```

### Wichtige Metriken fuer Statusmonitor

| Metrik | Kommando | Grenzwert |
|---|---|---|
| Sync-Status | `chronyc tracking \| grep "Leap status"` | Muss `Normal` sein |
| Stratum | `chronyc tracking \| grep "Stratum"` | Muss <= 3 sein |
| Root Delay | `chronyc tracking \| grep "Root delay"` | < 100 ms |
| Erreichbare Quellen | `chronyc sources \| grep "^\^\\*\\\|^\^\+"` | Min. 1 aktive Quelle |
| NTP-Port offen | `nc -uzw1 localhost 123` | Muss erreichbar sein |

### Einfacher Health-Check (fuer Monitoring-Scripte)

```bash
#!/bin/bash
# NTP Health Check - Exit 0 = OK, Exit 1 = Problem
LEAP=$(chronyc tracking 2>/dev/null | grep -c "Leap status.*Normal")
SOURCES=$(chronyc sources 2>/dev/null | grep -cE "^\^[\*\+]")

if [ "$LEAP" -eq 1 ] && [ "$SOURCES" -ge 1 ]; then
    exit 0
else
    exit 1
fi
```

---

## Logs

| Log | Pfad | Inhalt |
|---|---|---|
| Chrony Tracking | `/var/log/chrony/tracking.log` | Zeitsynchronisation |
| Chrony Measurements | `/var/log/chrony/measurements.log` | Messungen zu Quellen |
| Chrony Statistics | `/var/log/chrony/statistics.log` | Statistische Daten |
| Unattended-Upgrades | `/var/log/unattended-upgrades/` | Update-Verlauf |
| System (Reboot) | `journalctl --list-boots` | Reboot-Historie |

### Letzten Reboot pruefen

```bash
# Wann war der letzte Reboot?
who -b

# Alle Reboots auflisten
last reboot

# War der letzte Reboot durch unattended-upgrades?
grep -i reboot /var/log/unattended-upgrades/unattended-upgrades.log
```
