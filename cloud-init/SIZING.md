# NTP Server Sizing: Warum 1 vCPU / 1 GB RAM / 10 GB SSD ausreicht

## Uebersicht

Ein dedizierter NTP-Server (chrony) benoetigt minimale Ressourcen.
NTP ist eines der leichtgewichtigsten Netzwerkprotokolle ueberhaupt:
Ein einzelnes Request/Response-Paar besteht aus **zwei UDP-Paketen a 48 Byte**.

Diese Dokumentation belegt, warum ein Minimal-Server fuer stabilen NTP-Betrieb
(inkl. NTP-Pool-Teilnahme) ausreichend dimensioniert ist.

---

## CPU: 1 vCPU

| Faktor | Detail |
|---|---|
| Protokoll | UDP, verbindungslos, kein Handshake |
| Paketgroesse | 48 Byte Request + 48 Byte Response |
| Verarbeitung | Reine Integer-Arithmetik (Zeitstempel), kein TLS, kein Parsing |
| Chrony | Single-Threaded, Event-basiert |
| Praxis | Ein einzelner Core verarbeitet **>100.000 NTP-Pakete/Sek** problemlos |

### Zum Vergleich

Typische NTP-Pool-Server in Deutschland bedienen 2.000-10.000 Clients/Sek.
Das entspricht <5% CPU-Last auf einem einzelnen modernen vCPU-Core.

Selbst die PTB-Zeitserver (ptbtime1-3.ptb.de), die zu den meistgenutzten
NTP-Servern in Deutschland gehoeren, laufen auf vergleichbar kleiner Hardware.

---

## RAM: 1 GB

### Speicherverbrauch im Detail

| Komponente | RAM | Anmerkung |
|---|---|---|
| Linux Kernel | ~30 MB | Minimal-Kernel ohne Module fuer Desktop/GPU |
| systemd + journald | ~40 MB | Init-System, Log-Daemon |
| sshd | ~5 MB | Einziger Zusatz-Dienst |
| chronyd (Prozess) | ~10 MB | Hauptprozess, RSS |
| chrony Client-Log | bis 128 MB | `clientloglimit 134217728` (konfigurierbar) |
| **Summe (idle)** | **~215 MB** | |
| apt upgrade (Peak) | +200 MB | Kurzzeitig waehrend taeglichem Auto-Update |
| **Summe (Peak)** | **~415 MB** | |

### Warum kein OOM-Risiko

- Keine weiteren Dienste (kein Docker, kein Webserver, kein Desktop)
- Peak durch `apt upgrade` dauert wenige Minuten (taeglich 03:00 Uhr)
- Client-Log waechst nur bei aktiver Pool-Teilnahme auf Maximum
- Selbst im Worst Case (voller Client-Log + apt upgrade): **~540 MB von 1024 MB**
- Verbleibende ~480 MB stehen fuer Filesystem-Cache zur Verfuegung

### Client-Log Skalierung

Der `clientloglimit`-Parameter steuert, wie viele Client-IPs chrony trackt:

| clientloglimit | RAM | Ca. Clients | Empfehlung |
|---|---|---|---|
| 33554432 (32 MB) | 32 MB | ~500.000 | Interner Betrieb |
| 67108864 (64 MB) | 64 MB | ~1.000.000 | Kleiner Pool-Server |
| 134217728 (128 MB) | 128 MB | ~2.000.000 | Aktuelle Konfiguration |

---

## Disk: 10 GB SSD

| Komponente | Speicher | Anmerkung |
|---|---|---|
| Ubuntu 24.04 Minimal | ~2.5 GB | Server-Image ohne Extras |
| Installierte Pakete | ~100 MB | chrony, ufw, unattended-upgrades |
| apt-Cache | ~200 MB | Wird woechentlich bereinigt (`AutocleanInterval 7`) |
| Chrony-Logs | ~50 MB | tracking, measurements, statistics |
| System-Logs (journald) | ~100 MB | Rotiert automatisch |
| **Summe** | **~3 GB** | |
| **Frei verfuegbar** | **~7 GB** | Grosszuegiger Puffer |

### Warum SSD statt HDD

NTP selbst hat keine Disk-I/O-Anforderungen. Die SSD ist relevant fuer:
- Schnelleren Boot nach automatischem Reboot (nach Kernel-Updates)
- Schnelleres `apt upgrade` (Package-Extraktion)
- Insgesamt sind die I/O-Anforderungen vernachlaessigbar

---

## Netzwerk

| Faktor | Detail |
|---|---|
| Bandbreite pro Client | ~96 Byte (Request + Response) |
| 10.000 Clients/Sek | ~7.5 Mbit/s |
| Typischer Pool-Server (DE) | 2.000-5.000 Clients/Sek = 1.5-3.8 Mbit/s |
| Empfehlung | 100 Mbit/s Anbindung genuegt voellig |

NTP erzeugt **keine Bursts** — der Traffic ist gleichmaessig verteilt,
da Clients ihre Abfragen zufaellig ueber ihr Poll-Intervall (64-1024 Sek) verteilen.

---

## Verfuegbarkeit

### Automatischer Reboot

Der Server startet automatisch um 03:00 Uhr neu, falls Kernel-Updates anstehen.
Ein NTP-Server-Neustart ist fuer Clients **transparent**:

- NTP-Clients haben mehrere Server konfiguriert (Redundanz)
- Clients puffern die letzte bekannte Zeitdifferenz lokal (`driftfile`)
- Ein Ausfall von 1-2 Minuten (Reboot-Dauer) ist fuer NTP-Clients irrelevant
- Chrony synchronisiert sich nach Boot innerhalb von Sekunden (`makestep 1.0 3`)

### Kein Single Point of Failure

NTP-Clients sollten immer mehrere Server konfigurieren. Dieser Server ist
eine Quelle von mehreren — ein kurzer Ausfall durch Reboot oder Wartung
hat keine Auswirkung auf die Zeitsynchronisation der Clients.

---

## Fazit

| Ressource | Minimum | Empfohlen | Unsere Konfig |
|---|---|---|---|
| vCPU | 1 | 1 | 1 |
| RAM | 512 MB | 1 GB | 1 GB |
| Disk | 5 GB | 10 GB | 10 GB SSD |
| Netzwerk | 10 Mbit/s | 100 Mbit/s | abhaengig vom Hoster |

Ein NTP-Server ist einer der ressourcensparendsten Netzwerkdienste ueberhaupt.
Die gewaehlte Dimensionierung (1/1/10) bietet genuegend Headroom fuer
Pool-Betrieb und automatische OS-Updates ohne Stabilitaetsrisiko.
