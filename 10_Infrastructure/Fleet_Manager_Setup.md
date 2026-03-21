# Fleet Manager Dokumentation

## Übersicht

Der Fleet Manager ist ein Raspberry Pi in Heimnetz 2 (192.168.71.10), der folgende Aufgaben übernimmt:

- **Zentrales Backup-Aggregat**: Zieht Restic-Repositories von allen Fleet-Clients per Ansible/rsync
- **Fleet-Automatisierung**: Führt OS-Updates, Backup-Runs und Backup-Pulls auf allen Fleet-Hosts via Kestra + Ansible aus
- **Backup-Bereitstellung**: Stellt die aggregierten Backups per rsync-Daemon für das QNAP bereit
- **VPN-Client**: Ist per WireGuard ins VPN eingebunden, um alle Heimnetze zu erreichen

---

## Architektur

```
                    ┌─────────────────────────────────────────┐
                    │  Fleet Manager RPI (192.168.71.10)      │
                    │                                         │
                    │  ┌──────────────┐                       │
                    │  │ fleet-wg-    │  Port 8080 (Kestra)   │
                    │  │ client       │  Port 873  (rsync)    │
                    │  │ (WireGuard)  │◄──────────────────────┼── Netzwerk
                    │  └──────┬───────┘                       │
                    │         │ network_mode: service         │
                    │  ┌──────▼───────┐                       │
                    │  │ fleet-kestra │  Ansible Playbooks    │
                    │  │ (Kestra)     │──────────────────────►│── VPN → Fleet-Hosts
                    │  └──────────────┘                       │
                    │  ┌──────────────┐                       │
                    │  │ fleet-rsync- │                       │
                    │  │ server       │◄── QNAP pull backup   │
                    │  └──────────────┘                       │
                    └─────────────────────────────────────────┘
```

### Netzwerk-Details

| Komponente         | IP / Port         | Beschreibung                              |
|--------------------|-------------------|-------------------------------------------|
| Fleet Manager RPI  | 192.168.71.10     | Host-IP im Heimnetz 2                     |
| WireGuard VPN-IP   | 10.10.20.x        | VPN-Peer (Heimnetz 2 Gateway)             |
| DNS im Container   | 192.168.70.3      | Interner DNS-Resolver (AdGuard/Unbound)   |
| Kestra Web-UI      | :8080             | Geteilt über WireGuard-Container-Netzwerk |
| rsync-Daemon       | :873              | Read-only Bereitstellung für QNAP         |

Kestra hat **kein eigenes Netzwerk-Interface** — es nutzt `network_mode: "service:wireguard"`. Alle Ports laufen über den WireGuard-Container.

---

## Verzeichnisstruktur auf dem Host

```
/home/decebu/docker-stacks/stacks/fleet-manager/
├── docker-compose.yml          # Stack-Definition
├── application.yaml            # Kestra-Konfiguration (Docker-Plugin)
├── .env_encoded                # Secrets (git-crypt verschlüsselt im Repo)
├── rsyncd.conf                 # rsync-Daemon-Konfiguration
├── rsyncd.secrets              # rsync-Passwörter (nicht im Repo)
├── wg-client/                  # WireGuard-Client-Config (Bind-Mount)
│   ├── coredns/
│   │   └── Corefile            # CoreDNS-Config
│   ├── templates/
│   │   ├── peer.conf           # WireGuard Peer-Template
│   │   └── server.conf         # WireGuard Server-Template
│   └── wg_confs/
│       └── wg0.conf            # Aktive WireGuard-Konfiguration (mit Keys!)
└── kestra/                     # Kestra-Datenbankverzeichnis (Bind-Mount, 97MB)
    ├── database.mv.db          # H2-Datenbank (Flows, Executions, Schedules)
    ├── dev/                    # Kestra-Namespace "dev"
    │   └── fleet_management/
    │       └── _files/         # Ansible Playbooks & Inventories
    ├── company/
    └── tutorial/

/var/backup/                    # Backup-Aggregat (908MB+)
└── fleet_backups/
    └── <hostname>/             # Restic-Repo je Fleet-Client
        └── snapshots/
```

---

## Container-Details

### fleet-wg-client (WireGuard)

```yaml
image: linuxserver/wireguard
ports:
  - "8080:8080"   # Kestra Web-UI
dns:
  - 192.168.70.3
volumes:
  - ./wg-client:/config
  - /lib/modules:/lib/modules
sysctls:
  - net.ipv4.conf.all.src_valid_mark=1
```

Stellt das Netzwerk-Namespace für alle anderen Container bereit. Die eigentliche WireGuard-Config (`wg0.conf`) wird von linuxserver/wireguard aus dem `/config`-Verzeichnis geladen und enthält den Private Key sowie den Peer (VPN-Server).

### fleet-kestra (Kestra)

```yaml
image: kestra/kestra:latest-full
volumes:
  - ./kestra:/app/data
  - /var/run/docker.sock:/var/run/docker.sock
  - /var/backup/fleet_backups:/var/backup/fleet_backups
  - ./application.yaml:/etc/config/application.yaml
group_add:
  - 984   # docker-Gruppe auf dem Host
network_mode: "service:wireguard"
env_file:
  - .env_encoded
```

`kestra:latest-full` enthält bereits alle benötigten Plugins inklusive Ansible. Kestra startet im `server local`-Modus (kein Kubernetes). Die `database.mv.db` in `./kestra` enthält alle Flows, Schedules und Execution-History.

**Wichtig:** `group_add: 984` ist die Docker-GID auf dem Host — muss nach Neuinstallation geprüft werden:
```bash
getent group docker | cut -d: -f3
```

### fleet-rsync-server (rsync)

```yaml
image: instrumentisto/rsync-ssh
ports:
  - "873:873"
volumes:
  - /var/backup:/data:ro
  - ./rsyncd.conf:/etc/rsyncd.conf:ro
  - ./rsyncd.secrets:/etc/rsyncd.secrets:ro
```

Stellt `/var/backup` read-only als rsync-Modul `fleet_backups` bereit. Authentifizierung per User `qnap_fleet` mit Passwort aus `rsyncd.secrets`.

---

## Secrets & Konfiguration

### .env_encoded

Git-crypt-verschlüsselt im Repo. Enthält folgende Variablen für Kestra:

| Variable                  | Beschreibung                                   |
|---------------------------|------------------------------------------------|
| `SECRET_ANSIBLE_PRIVATE_KEY` | Base64-kodierter SSH-Private-Key für Ansible |
| `SECRET_NTFY_TOKEN`          | API-Token für ntfy-Benachrichtigungen        |

Diese werden in Kestra als `{{ secret('ANSIBLE_PRIVATE_KEY') }}` bzw. `{{ secret('NTFY_TOKEN') }}` referenziert.

### rsyncd.secrets

Nicht im Repo, muss manuell erstellt werden:

```
qnap_fleet:<PASSWORD>
```

Dateirechte zwingend: `chmod 600 rsyncd.secrets`

### wg-client/wg_confs/wg0.conf

Enthält den WireGuard Private Key und Preshared Key. Diese Datei wird von linuxserver/wireguard automatisch geladen. Bei Neuinstallation muss entweder:
- die Datei aus dem Backup wiederhergestellt werden, **oder**
- ein neuer Peer beim VPN-Server registriert werden (siehe `VPN_Firezone_installation.md`)

---

## Kestra Flows

Alle Flows liegen im Namespace `dev.fleet_management`. Die Ansible Playbooks sind als Kestra-Files (`_files`) hinterlegt und werden direkt im Kestra-Container ausgeführt.

### fleet_upgrade

**Zweck:** OS-Sicherheitsupdates auf allen Fleet-Hosts
**Schedule:** Konfiguriert in Kestra
**Playbook:** `fleet_upgrade.yaml`

- Führt `apt dist-upgrade` auf allen Hosts in `inventory_fleet.ini` durch
- Prüft ob Reboot nötig ist
- Meldet Status pro Host zurück an Kestra

### fleet_restic_backup

**Zweck:** Restic-Backup auf jedem Fleet-Host lokal ausführen
**Playbook:** `fleet_restic_backup.yaml`

- Installiert Restic falls nicht vorhanden
- Initialisiert Restic-Repo unter `/var/backup/restic-repo` falls nicht vorhanden
- Passwort-Datei muss auf jedem Host unter `/root/.restic_pw` liegen
- Sonderpfad VPS: Erstellt zusätzlich Postgres-Dump (Firezone-DB)
- Retention: 7 daily, 4 weekly
- Meldet Status für ntfy-Benachrichtigung

### fleet_pull_backup

**Zweck:** Restic-Repos von Fleet-Hosts zum Fleet Manager ziehen
**Playbook:** `fleet_pull_backup.yaml`

- Zieht per `ansible.posix.synchronize` (rsync) die Repos von jedem Host
- Quelle auf Fleet-Host: `/var/backup/restic-repo/`
- Ziel auf Fleet Manager: `/var/backup/fleet_backups/<hostname>/`
- `delegate_to: localhost` — rsync läuft im Kestra-Container, nicht auf dem Zielhost

### Inventories

Zwei verschlüsselte Inventory-Dateien (git-crypt):

- `inventory_fleet.ini` — Die zu verwaltenden Fleet-Clients
- `inventory_homelab.ini` — Alle Homelab-Hosts

---

## Zugriffsrechte /var/backup

Dies ist der kritische Punkt. Kestra läuft als Container-User (UID 1000), startet aber zusätzlich einen **Ansible-Container** via Docker-Socket. Dieser Ansible-Container läuft ebenfalls mit UID 1000.

Damit der Pull-Backup-Playbook in `/var/backup/fleet_backups/<hostname>/` schreiben kann:

```bash
# /var/backup muss decebu (UID 1000) gehören:
sudo chown -R decebu:decebu /var/backup/fleet_backups

# /var/backup selbst gehört root, fleet_backups dem User:
ls -la /var/backup/
# drwxr-xr-x  root   root     /var/backup/
# drwxr-xr-x  decebu decebu   /var/backup/fleet_backups/
```

Der Bind-Mount im Kestra-Container (`/var/backup/fleet_backups:/var/backup/fleet_backups`) stellt sicher, dass der UID-1000-User innerhalb des Ansible-Containers auf das Verzeichnis schreiben kann, weil die Eigentümerschaft auf Host-Ebene passt.

---

## Neuinstallation (von Grund auf)

### 1. Voraussetzungen

- Raspberry Pi mit Raspberry Pi OS (64-bit, Debian Bookworm)
- User `decebu` angelegt (UID 1000)
- Docker CE installiert, `decebu` in `docker`-Gruppe
- SSH-Zugriff via `fleet-mgr` in `~/.ssh/config`

```
Host fleet-mgr
    HostName 192.168.71.10
    User decebu
```

### 2. Repo klonen

```bash
ssh fleet-mgr
git clone <REPO_URL> /home/decebu/docker-stacks
cd /home/decebu/docker-stacks
git-crypt unlock  # .env_encoded und Inventories entschlüsseln
```

### 3. /var/backup anlegen

```bash
sudo mkdir -p /var/backup/fleet_backups
sudo chown -R decebu:decebu /var/backup/fleet_backups
```

### 4. rsyncd.secrets erstellen

```bash
echo "qnap_fleet:<PASSWORD>" > /home/decebu/docker-stacks/stacks/fleet-manager/rsyncd.secrets
chmod 600 /home/decebu/docker-stacks/stacks/fleet-manager/rsyncd.secrets
```

### 5. Docker-GID prüfen und anpassen

```bash
getent group docker | cut -d: -f3
```

In `docker-compose.yml` unter `kestra.group_add` die GID eintragen (aktuell `984`).

### 6. Stack starten

```bash
cd /home/decebu/docker-stacks/stacks/fleet-manager
docker compose up -d
```

Kestra Web-UI: http://192.168.71.10:8080

### 7. WireGuard-Verbindung prüfen

```bash
docker exec fleet-wg-client wg show
```

Sollte den Peer und den letzten Handshake anzeigen. Falls die WireGuard-Config fehlt oder der Peer neu registriert werden muss, siehe `VPN_Firezone_installation.md`.

---

## Restore aus Backup (dd-Image)

Falls der RPI neu eingerichtet werden muss und ein dd-Image der alten Partition vorliegt:

```bash
# Image mounten (Offset der Linux-Partition ermitteln):
sudo fdisk -l /pfad/zum/image.img
# offset = start_sector * 512
sudo mount -o loop,offset=<OFFSET> /pfad/zum/image.img /mnt/rpi

# Alternativ: Partition direkt mounten wenn als Block-Device verfügbar
sudo mount /dev/sdX2 /mnt/rpi
```

### Dateien übertragen

```bash
# fleet-manager Stack-Verzeichnis (inkl. Kestra-Datenbank, WireGuard-Keys, Secrets):
rsync -aAXv --progress \
  /mnt/rpi/home/decebu/docker-stacks/stacks/fleet-manager/ \
  fleet-mgr:/home/decebu/docker-stacks/stacks/fleet-manager/

# /var/backup (Backup-Daten, braucht sudo auf Ziel):
rsync -aAXv --progress \
  --rsync-path="sudo rsync" \
  /mnt/rpi/var/backup/ \
  fleet-mgr:/var/backup/
```

`-aAX` erhält Permissions, ACLs und Extended Attributes exakt wie im Original — das löst das Zugriffsrechteproblem für Kestra/Ansible automatisch mit.

### Danach Stack starten

```bash
ssh fleet-mgr "cd /home/decebu/docker-stacks/stacks/fleet-manager && docker compose up -d"
```

Alle Kestra-Flows und Schedules sind sofort aktiv, da die `database.mv.db` vollständig wiederhergestellt wurde.

---

## Troubleshooting

### Kestra kann nicht in /var/backup schreiben

```bash
# Auf dem Fleet Manager Host:
ls -la /var/backup/
# fleet_backups muss decebu:decebu gehören
sudo chown -R decebu:decebu /var/backup/fleet_backups
```

### WireGuard verbindet nicht

```bash
docker logs fleet-wg-client
docker exec fleet-wg-client wg show
# wg0.conf in ./wg-client/wg_confs/ prüfen
```

### Docker-GID stimmt nicht (Kestra kann keinen Ansible-Container starten)

```bash
# Aktuelle Docker-GID ermitteln:
getent group docker | cut -d: -f3
# In docker-compose.yml unter kestra.group_add anpassen, dann:
docker compose up -d --force-recreate kestra
```

### rsync-Verbindung vom QNAP schlägt fehl

```bash
# Secrets-Datei prüfen (muss 600 sein):
ls -la /home/decebu/docker-stacks/stacks/fleet-manager/rsyncd.secrets
# Daemon-Log:
docker logs fleet-rsync-server
```