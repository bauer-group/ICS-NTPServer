# NTP Server — time.bauer-group.com

Oeffentlicher NTP-Server auf Basis von [Chrony](https://chrony-project.org/),
betrieben im [NTP Pool](https://www.ntppool.org/).

Upstream-Quellen: PTB (Braunschweig), TU Berlin, FAU Erlangen, FU Berlin.

## Deployment-Optionen

### 1. Cloud-Init (VM)

Vollstaendiges Setup einer dedizierten VM mit automatischen OS-Updates
und gehaertetem Chrony. Empfohlen fuer den Produktivbetrieb.

```bash
# Als user-data bei VM-Erstellung angeben
cloud-init/user-data.yml
```

Alternativ auf einem bestehenden System:

```bash
sudo bash cloud-init/install.sh
```

Zielplattform: Ubuntu 24.04+ / Debian 13+

Minimale Ressourcen: 1 vCPU / 1 GB RAM / 10 GB SSD
(Details siehe [SIZING.md](cloud-init/SIZING.md))

### 2. Docker

Leichtgewichtige Alternative fuer bestehende Container-Hosts.
Nutzt `network_mode: host` fuer nativen IPv4/IPv6-Support.

```bash
cd docker
docker compose up -d
```

## Wartungsfenster

Alle automatischen Wartungsarbeiten sind in ein Fenster gelegt:

```
Taeglich 02:55 - 03:35 UTC
```

| Zeit (UTC) | Prozess |
|---|---|
| 03:00 | Paketlisten aktualisieren |
| 03:10 | Pakete installieren |
| 03:25 | Reboot falls noetig (1-2x/Monat) |

Details, Monitoring-Checks und Health-Check-Script:
[MAINTENANCE.md](cloud-init/MAINTENANCE.md)

## Projektstruktur

```
.
├── cloud-init/
│   ├── user-data.yml       Cloud-Init Konfiguration
│   ├── install.sh          Fallback-Script fuer manuelle Installation
│   ├── SIZING.md           Ressourcen-Dokumentation (CPU/RAM/Disk)
│   └── MAINTENANCE.md      Wartungsfenster & Monitoring
├── docker/
│   └── docker-compose.yml  Docker Compose Konfiguration
├── LICENSE
└── README.md
```

## Chrony-Features

- **Rate Limiting** — Schutz vor NTP-Amplification-Angriffen (Pool-tauglich)
- **Client-Log** — Tracking von bis zu 2 Mio. Clients (128 MB)
- **Persistenz** — Drift- und Messdaten ueberleben Neustarts
- **Auto-Updates** — Taegliche OS-Updates mit automatischem Reboot
- **Firewall** — UFW mit ausschliesslich SSH + NTP (123/udp)
