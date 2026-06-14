# Fleet Manager Dokumentation

## Übersicht

Der Fleet Manager ist ein Raspberry Pi in Heimnetz 2 (192.168.71.10), der folgende Aufgaben übernimmt:

- **Fleet-Automatisierung**: Führt OS-Updates, Restic-Backup-Runs und Backup-Pulls auf allen Fleet-Hosts via Kestra + Ansible aus
- **Zentrales Backup-Aggregat**: Zieht die Restic-Repositories aller Fleet-Clients per rsync auf den Manager
- **Benachrichtigung**: Meldet den Status der Flows per ntfy-Push
- **Backup-Bereitstellung**: Stellt die aggregierten Backups per rsync-Daemon für das QNAP bereit
- **VPN-Client**: Ist per WireGuard ins VPN eingebunden, um alle Fleet-Hosts (10.10.30.x) zu erreichen

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
                    │  │ fleet-kestra │  startet Ansible-     │
                    │  │ (Kestra)     │  Container via Docker  │
                    │  └──────┬───────┘                       │
                    │         │ networkMode: container:        │
                    │         │ fleet-wg-client                │
                    │  ┌──────▼───────┐                       │
                    │  │ cytopia/     │──────────────────────►│── VPN (10.10.30.x) → Fleet-Hosts
                    │  │ ansible      │  ansible-playbook      │
                    │  └──────────────┘                       │
                    │  ┌──────────────┐                       │
                    │  │ fleet-rsync- │◄── QNAP pull backup   │
                    │  │ server       │                       │
                    │  └──────────────┘                       │
                    └─────────────────────────────────────────┘
```

Der zentrale Trick: Sowohl der Kestra-Container als auch die von Kestra **dynamisch
gestarteten Ansible-Container** hängen sich in den Netzwerk-Namespace des
WireGuard-Containers. Dadurch erreichen die Ansible-Playbooks die Fleet-Hosts über das
VPN (10.10.30.x), ohne dass Kestra ein eigenes Netzwerk-Interface besitzt.

### Netzwerk-Details

| Komponente         | IP / Port         | Beschreibung                              |
|--------------------|-------------------|-------------------------------------------|
| Fleet Manager RPI  | 192.168.71.10     | Host-IP im Heimnetz 2                     |
| WireGuard VPN-Range| 10.10.30.x        | Fleet-Hosts (Peers) über VPN              |
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
└── kestra/                     # Kestra-Datenbankverzeichnis (Bind-Mount)
    ├── database.mv.db          # H2-Datenbank (Flows, Executions, Schedules)
    ├── dev/                    # Kestra-Namespace "dev"
    │   └── fleet_management/
    │       └── _files/         # Ansible Playbooks & Inventories (Namespace-Files)
    │           ├── fleet_restic_backup.yaml
    │           ├── fleet_pull_backup.yaml
    │           ├── fleet_upgrade.yaml
    │           ├── inventory_fleet.ini
    │           └── inventory_homelab.ini
    ├── company/
    └── tutorial/

/var/backup/                    # Backup-Aggregat
└── fleet_backups/
    └── <hostname>/             # Restic-Repo je Fleet-Client (gepullt)
        └── ...
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

Kestra führt die Ansible-Playbooks nicht selbst aus, sondern startet pro Task über den
gemounteten Docker-Socket einen **Ansible-Container** (`cytopia/ansible:latest-tools`) mit
`networkMode: "container:fleet-wg-client"`, damit dieser über das VPN auf die Fleet-Hosts
zugreifen kann.

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
| `SECRET_ANSIBLE_PRIVATE_KEY` | Base64-kodierter SSH-Private-Key für Ansible (User `kestra` auf den Fleet-Hosts) |
| `SECRET_NTFY_TOKEN`          | ntfy-Access-Token für Benachrichtigungen     |

Referenzierung in Kestra als `{{ secret('ANSIBLE_PRIVATE_KEY') }}` bzw. `{{ secret('NTFY_TOKEN') }}`.

**Wichtig zur Kodierung:** Kestra erwartet `SECRET_*`-Variablen **base64-kodiert** und
dekodiert sie beim Zugriff über `secret(...)`. Der Wert in `.env_encoded` ist also der
base64-kodierte Token, nicht der Klartext-Token. Beispiel zum Erzeugen:

```bash
printf '%s' 'tk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx' | base64
# Ergebnis als Wert von SECRET_NTFY_TOKEN eintragen
```

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

Alle Flows liegen im Namespace `dev.fleet_management`. Die Ansible-Playbooks und Inventories
sind als Kestra-Namespace-Files (`_files/`) hinterlegt und werden in den Flows per
`{{ read('<datei>') }}` in den Ansible-Container geladen.

> Hinweis: Die Doku benennt die Flows nach ihrer **Kestra-Flow-ID** (nicht nach dem
> Playbook-Dateinamen). Das jeweils ausgeführte Playbook ist je Flow angegeben.

### Ablauf-Kette (Schedule + Flow-Trigger)

```
täglich 02:00  ─► restic_backup_fleet ─(bei Abschluss)─► pull_backups_to_manager
                        │                                        │
                        └──────────────┬─────────────────────────┘
                                       ▼ (bei Abschluss, jeweils)
sonntags 03:00 ─► update_fleet_devices ─► fleet_backup_notification ─► ntfy-Push
sonntags 03:00 ─► update_lan_devices
```

### restic_backup_fleet

- **Trigger:** Schedule `cron: "0 2 * * *"` (täglich 02:00)
- **Playbook:** `fleet_restic_backup.yaml` (Inventory: `inventory_fleet.ini`, `--user kestra`)
- **Task:** `run_ansible_backup` (`io.kestra.plugin.ansible.cli.AnsibleCLI`)

Führt das Restic-Backup **lokal auf jedem Fleet-Host** aus:
- Installiert Restic, falls nicht vorhanden
- Prüft Passwort-Datei `/root/.restic_pw` (Playbook bricht ab, falls sie fehlt)
- Initialisiert das Repo unter `/var/backup/restic-repo`, falls nicht vorhanden
- Sonderfall VPS (`is_vps=true`): erstellt zusätzlich einen Postgres-Dump
  (`pg_dumpall` der Firezone-DB) und leitet ihn per `--stdin` in Restic
- Sichert die je Host definierten `backup_paths`
- `restic check` (Integritätsprüfung)
- Retention: `--keep-daily 7 --keep-weekly 4 --prune`

### pull_backups_to_manager

- **Trigger:** Flow-Trigger `after_backup_completion` — startet nach Abschluss von
  `restic_backup_fleet`
- **Playbook:** `fleet_pull_backup.yaml`
- **Task:** `run_pull_playbook` (`AnsibleCLI`)

Zieht die Restic-Repos von jedem Fleet-Host auf den Manager:
- Quelle auf Fleet-Host: `/var/backup/restic-repo/`
- Ziel auf Fleet Manager: `/var/backup/fleet_backups/<hostname>/`
- per `ansible.posix.synchronize` (rsync, `mode: pull`, `--archive --delete`)
- Das Zielverzeichnis wird `delegate_to: localhost` angelegt — der rsync-Pull läuft also
  im Ansible-Container (= Fleet Manager), nicht auf dem Zielhost

### update_fleet_devices

- **Trigger:** Schedule `cron: "0 3 * * 0"` (sonntags 03:00)
- **Playbook:** `fleet_upgrade.yaml`
- **Task:** `run_ansible_updates` (`AnsibleCLI`)
- **Beschreibung:** „Führt apt upgrade auf der gesamten Flotte durch"

Führt `apt update` + `upgrade: dist` + `autoremove` + `autoclean` auf allen Fleet-Hosts
aus und prüft per `/var/run/reboot-required`, ob ein Reboot nötig ist.

### update_lan_devices

- **Trigger:** Schedule `cron: "0 3 * * 0"` (sonntags 03:00)
- **Task:** `run_lan_update` (`AnsibleCLI`)

Update-Lauf für die LAN-Geräte (separat von der eigentlichen Fleet).

### fleet_backup_notification

- **Trigger:** Drei Flow-Trigger, jeweils mit Bedingung
  `ExecutionStatus in [SUCCESS, FAILED, WARNING]`:
  - `on_backup_completion` → nach `restic_backup_fleet`
  - `on_update_completion` → nach `update_fleet_devices`
  - `on_backup_pull_completion` → nach `pull_backups_to_manager`
- **Task:** `ntfy_send` (`io.kestra.plugin.core.http.Request`, `POST`)

Zentraler Benachrichtigungs-Flow. Reagiert auf den Abschluss der drei o.g. Flows und
schickt eine ntfy-Push an das Topic `fleet_backups`. Titel und Tags werden dynamisch nach
Status/Quell-Flow gesetzt, der Body enthält Flow-ID, Status, Execution-ID und einen
Deep-Link in die Kestra-UI. Siehe Abschnitt **ntfy-Integration**.

> Diese Trennung (Backup-Flows ohne eigene ntfy-Logik, eine zentrale Notification)
> ersetzt die früheren Inline-ntfy-Tasks in den einzelnen Backup-Flows.

### hello_fleet

- **Trigger:** keiner (manuell)
- **Task:** `check_remote_rpi` (`AnsibleCLI`, `ansible -m ping`)

Diagnose-Flow zum Prüfen der SSH/VPN-Erreichbarkeit der Fleet-Hosts.

### Inventories

Zwei git-crypt-verschlüsselte Inventory-Dateien:

- `inventory_fleet.ini` — die zu verwaltenden Fleet-Clients
- `inventory_homelab.ini` — alle Homelab-Hosts

Aufbau `inventory_fleet.ini` (Gruppe `[fleet]`, alle über VPN 10.10.30.x):

| Host         | Rolle                                  | Host-Variablen                                  |
|--------------|----------------------------------------|-------------------------------------------------|
| 10.10.30.1   | VPS (Firezone, WireGuard, UFW)         | `is_vps=true`, `ansible_port=54322`, `pg_container=firezone-postgres` |
| 10.10.30.20  | Client + Home Assistant                | `is_ha_client=true`                             |
| 10.10.30.21  | Client                                 | —                                               |
| 10.10.30.23  | Client                                 | —                                               |

Pro Host steuert `backup_paths` die zu sichernden Pfade (z.B.
`/home/decebu/docker-stacks /etc/wireguard`), beim HA-Client zusätzlich das
Home-Assistant-Verzeichnis.

---

## ntfy-Integration

Die Benachrichtigungen laufen über einen **ntfy-Server, der nicht auf dem Fleet Manager,
sondern im `vps-monitoring`-Stack auf dem VPS** betrieben wird.

### ntfy-Server (VPS)

- Image `binwiederhier/ntfy`, erreichbar unter `<NTFY_PUBLIC_ENDPOINT>`
- `--auth-default-access=deny-all` + `--enable-login=true` → **jeder Zugriff erfordert
  Authentifizierung**
- Auth-DB: `/etc/ntfy/user.db` (Bind-Mount)
- Topic für die Fleet-Benachrichtigungen: `fleet_backups`
- Service-User für Kestra: `kuma-bot`

### Anbindung von Kestra

Der Flow `fleet_backup_notification` sendet per HTTP-POST an
`<NTFY_PUBLIC_ENDPOINT>/fleet_backups` mit Bearer-Auth:

```yaml
tasks:
  - id: ntfy_send
    type: io.kestra.plugin.core.http.Request
    uri: "<NTFY_PUBLIC_ENDPOINT>/fleet_backups"
    method: POST
    headers:
      Authorization: "Bearer {{ secret('NTFY_TOKEN') | trim }}"
      Title: "Fleet {{ trigger.flowId == 'restic_backup_fleet' ? 'Backup' : 'Update' }}: {{ trigger.state }}"
      Tags:  "{{ trigger.state == 'SUCCESS' ? 'green_circle' : 'red_circle' }},{{ trigger.flowId == 'restic_backup_fleet' ? 'floppy_disk' : 'arrow_up' }}"
    body: |
      Fleet-Management Bericht ({{ now() | date("HH:mm") }}):
      Der Flow '{{ trigger.flowId }}' wurde abgeschlossen.
      Status: {{ trigger.state }}
      Execution-ID: {{ trigger.executionId }}
      Logs: <KESTRA_PUBLIC_ENDPOINT>/ui/executions/{{ trigger.namespace }}/{{ trigger.flowId }}/{{ trigger.executionId }}
    contentType: text/plain
```

Das `| trim` entfernt ein evtl. an den Token angehängtes Newline (entsteht beim
base64-Kodieren mit `echo`).

### Token verwalten (auf dem VPS)

```bash
# Schreibrecht des Users auf das Topic prüfen/setzen
docker exec ntfy ntfy access kuma-bot fleet_backups rw

# Neuen Access-Token erzeugen
docker exec ntfy ntfy token add kuma-bot
# -> "token tk_... created for user kuma-bot"

# Benutzer / bestehende Rechte ansehen
docker exec ntfy ntfy user list
docker exec ntfy ntfy access kuma-bot
```

Der erzeugte Token muss anschließend **base64-kodiert** als `SECRET_NTFY_TOKEN` in
`.env_encoded` auf dem Fleet Manager hinterlegt und Kestra neu gestartet werden
(siehe Abschnitt **Secrets** und **Troubleshooting → ntfy 401**).

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

### 8. ntfy-Anbindung prüfen

Auf dem Fleet Manager (Pfad, den Kestra nutzt — über den WireGuard-Container):

```bash
docker exec fleet-wg-client sh -c \
  'curl -s -o /dev/null -w "%{http_code}\n" \
   -H "Authorization: Bearer <TOKEN>" \
   <NTFY_PUBLIC_ENDPOINT>/v1/account'
# Erwartet: 200
```

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

## Secret-Rotation

Beide für Kestra relevanten Secrets liegen base64-kodiert in `.env_encoded`
(`SECRET_NTFY_TOKEN`, `SECRET_ANSIBLE_PRIVATE_KEY`) und werden per `env_file` eingelesen.
Eine Rotation besteht immer aus: neues Geheimnis erzeugen → an der Quelle hinterlegen →
`.env_encoded` aktualisieren → Kestra neu starten → verifizieren → committen.

> Hinweis: Beim base64-Kodieren `printf '%s'` (kein `echo`) verwenden, damit kein
> Trailing-Newline mitkodiert wird. Der ntfy-Task entfernt es zwar zusätzlich per `| trim`,
> beim Ansible-Key wäre ein angehängtes Newline aber unkritisch bis störend — sauber ist
> ohne.

### ntfy-Token rotieren

Unkritisch und schnell — der Token hat nur Publish-Recht auf das Topic `fleet_backups`.

```bash
# 1. VPS: Schreibrecht sicherstellen + neuen Token erzeugen
docker exec ntfy ntfy access kuma-bot fleet_backups rw
docker exec ntfy ntfy token add kuma-bot
#    -> "token tk_... created for user kuma-bot"

# 2. Fleet Manager: neuen Token base64-kodieren
printf '%s' 'tk_NEUER_TOKEN' | base64

# 3. In .env_encoded den Wert von SECRET_NTFY_TOKEN ersetzen
#    SECRET_NTFY_TOKEN=<base64-ergebnis>

# 4. Kestra neu starten
cd /home/decebu/docker-stacks/stacks/fleet-manager && docker compose up -d kestra

# 5. Verifizieren (über den WireGuard-Pfad, den Kestra nutzt) -> erwartet 200
docker exec fleet-wg-client sh -c \
  'curl -s -o /dev/null -w "%{http_code}\n" \
   -H "Authorization: Bearer tk_NEUER_TOKEN" <NTFY_PUBLIC_ENDPOINT>/v1/account'

# 6. Alten Token serverseitig entwerten (Liste ansehen, dann gezielt löschen)
docker exec ntfy ntfy token list kuma-bot
docker exec ntfy ntfy token remove kuma-bot tk_ALTER_TOKEN

# 7. .env_encoded committen (git-crypt verschlüsselt automatisch)
git -C /home/decebu/docker-stacks add stacks/fleet-manager/.env_encoded
git -C /home/decebu/docker-stacks commit -m "chore(fleet): ntfy-Token rotiert"
```

### Ansible-SSH-Key rotieren

Sicherheitskritisch: Der Key gibt SSH-Zugriff als User `kestra` (mit `become`/root via
Playbooks) auf **alle** Fleet-Hosts inkl. VPS. Vorgehen mit Überlappung — neuer Public Key
wird verteilt, **bevor** der alte entfernt wird, damit man sich nicht aussperrt.

```bash
# 1. Neues ed25519-Keypair erzeugen (z.B. lokal in einem temp-Verzeichnis)
ssh-keygen -t ed25519 -f ./fleet_ansible_key -C "kestra-fleet" -N ""
#    erzeugt fleet_ansible_key (privat) + fleet_ansible_key.pub (öffentlich)
```

**2. Neuen Public Key auf allen Fleet-Hosts ausrollen** — solange der alte Key noch gültig
ist. Am einfachsten per Ansible-Ad-hoc mit dem *alten* Key (auf einem Host mit Ansible,
Inventory `inventory_fleet.ini`):

```bash
ansible fleet -i inventory_fleet.ini \
  --user kestra --private-key ./alter_ansible_key --become \
  -m ansible.posix.authorized_key \
  -a "user=kestra state=present key='$(cat ./fleet_ansible_key.pub)'"
```

```bash
# 3. Privaten Key base64-kodieren (einzeilig, ohne Umbrüche)
base64 -w0 ./fleet_ansible_key

# 4. In .env_encoded den Wert von SECRET_ANSIBLE_PRIVATE_KEY ersetzen
#    SECRET_ANSIBLE_PRIVATE_KEY=<base64-ergebnis>

# 5. Kestra neu starten
cd /home/decebu/docker-stacks/stacks/fleet-manager && docker compose up -d kestra
```

**6. Verifizieren:** Den Flow `hello_fleet` in der Kestra-UI manuell ausführen
(`ansible -m ping`). Alle Hosts müssen mit dem neuen Key erreichbar sein (`pong`).

```bash
# 7. Erst NACH erfolgreichem Test: alten Public Key auf allen Hosts entfernen
ansible fleet -i inventory_fleet.ini \
  --user kestra --private-key ./fleet_ansible_key --become \
  -m ansible.posix.authorized_key \
  -a "user=kestra state=absent key='$(cat ./alter_ansible_key.pub)'"

# 8. .env_encoded committen
git -C /home/decebu/docker-stacks add stacks/fleet-manager/.env_encoded
git -C /home/decebu/docker-stacks commit -m "chore(fleet): Ansible-SSH-Key rotiert"

# 9. Lokale Klartext-Kopien des privaten Keys sicher löschen
shred -u ./fleet_ansible_key ./alter_ansible_key 2>/dev/null; rm -f ./fleet_ansible_key.pub ./alter_ansible_key.pub
```

> Falls der alte Key nicht mehr verfügbar ist, muss der neue Public Key alternativ per
> Konsolen-/Out-of-Band-Zugriff in `~kestra/.ssh/authorized_keys` jedes Hosts eingetragen
> werden.

### Wann rotieren?

- **ntfy-Token:** bei `401`-Fehlern (siehe Troubleshooting) oder Verdacht auf Leak.
- **Ansible-Key:** bei Verdacht auf Kompromittierung, ausgeschiedenem Zugriff, oder wenn
  der Key (z.B. über Logs, Tool-Kontext, Screenshares) außerhalb von `.env_encoded`
  sichtbar war.

> Härtungshinweis: Frühere Flow-Revisionen enthielten einen `debug_secret`-Task, der den
> ntfy-Token per `echo` in die Execution-Logs geschrieben hat. Solche Debug-Tasks vor
> produktiver Nutzung entfernen — gerenderte Secrets bleiben sonst in der `database.mv.db`
> erhalten. Eine Rotation entwertet den geloggten alten Token.

---

## Troubleshooting

### ntfy 401 Unauthorized (Benachrichtigung schlägt fehl)

Symptom in den Kestra-Logs des Tasks `ntfy_send`:

```
{"code":40101,"http":401,"error":"unauthorized"}
```

Ursache ist fast immer ein **serverseitig nicht mehr gültiger Token** (regeneriert, oder
`user.db` zurückgesetzt), während das Kestra-Secret noch den alten Token enthält.

**Diagnose** (read-only, ohne eine Push zu senden) am ntfy-Endpoint `/v1/account`:

```bash
# anonym -> 200 (Server erreichbar), mit ungültigem/altem Token -> 401
curl -s -o /dev/null -w "%{http_code}\n" <NTFY_PUBLIC_ENDPOINT>/v1/account
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer <TOKEN>" <NTFY_PUBLIC_ENDPOINT>/v1/account
```

- anonym `200` + Token `200` → Token gültig (Problem liegt woanders, z.B. Topic-Recht → dann
  `403` beim Publish, nicht `401`)
- anonym `200` + Token `401` → **Token ungültig** → neuen Token erzeugen

**Fix:**

```bash
# 1. Auf dem VPS: neuen Token erzeugen (Schreibrecht sicherstellen)
docker exec ntfy ntfy access kuma-bot fleet_backups rw
docker exec ntfy ntfy token add kuma-bot

# 2. Auf dem Fleet Manager: Token base64-kodieren und in .env_encoded eintragen
printf '%s' 'tk_NEUER_TOKEN' | base64
#   -> als Wert von SECRET_NTFY_TOKEN setzen

# 3. Kestra neu starten (liest .env_encoded neu ein)
cd /home/decebu/docker-stacks/stacks/fleet-manager && docker compose up -d kestra

# 4. Verifizieren (über den WireGuard-Pfad, den Kestra nutzt): erwartet 200
docker exec fleet-wg-client sh -c \
  'curl -s -o /dev/null -w "%{http_code}\n" \
   -H "Authorization: Bearer tk_NEUER_TOKEN" <NTFY_PUBLIC_ENDPOINT>/v1/account'
```

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

### Fleet-Host nicht erreichbar (Ansible)

```bash
# hello_fleet-Flow in der Kestra-UI manuell ausführen (ansible ping)
# oder VPN-Handshake prüfen:
docker exec fleet-wg-client wg show
```
