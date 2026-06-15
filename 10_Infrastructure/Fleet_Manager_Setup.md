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
├── rsyncd.conf                 # rsync-Daemon-Konfiguration
├── rsyncd.secrets              # rsync-Passwörter (Host-only, gitignored, 600)
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
environment:
  - SECRET_ANSIBLE_PRIVATE_KEY=${SECRET_ANSIBLE_PRIVATE_KEY:?bitte via fleet-up starten}
  - SECRET_NTFY_TOKEN=${SECRET_NTFY_TOKEN:?bitte via fleet-up starten}
```

Die Secrets stehen **nicht** als Datei im Repo, sondern werden zur Laufzeit per `fleet-up`
aus einer age-verschlüsselten Datei in die Shell-Env injiziert (siehe **Secrets & Konfiguration**).
Der `:?`-Guard lässt ein versehentliches `docker compose up` (ohne geladene Env) bewusst
fehlschlagen, statt Kestra mit leeren Secrets zu starten.

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

### Kestra-Secrets (age-verschlüsselt, Runtime-Injektion)

Die Kestra-Secrets liegen **nicht** mehr im Repo (früher `.env_encoded`, git-crypt — entfernt).
Stattdessen:

| Artefakt | Ort | Inhalt |
|---|---|---|
| age-Identity (passphrase-geschützt) | `~/.config/age/keys.txt.age` | Root-of-Trust, Host-only, in keinem git |
| age-verschlüsselte Env | `~/.config/fleet/kestra.env.age` | `SECRET_ANSIBLE_PRIVATE_KEY`, `SECRET_NTFY_TOKEN` (base64), Host-only |

Variablen werden in Kestra als `{{ secret('ANSIBLE_PRIVATE_KEY') }}` bzw.
`{{ secret('NTFY_TOKEN') }}` referenziert (Kestra base64-**dekodiert** `SECRET_*`).

**Ablauf:**
- `fleet-genkey` (chezmoi, `~/.local/bin`) erzeugt ein neues Ansible-Keypair und schreibt die
  age-Env (privater Key nie im Klartext; gibt nur den Public Key aus).
- `fleet-up` (chezmoi) entschlüsselt die Identity (Passphrase-Prompt) und damit die Env zur
  Laufzeit und startet den Stack — **keine** Klartext-Env-Datei at rest.
- Reboots brauchen kein `fleet-up`: Docker behält die Env des erstellten Containers
  (`restart: unless-stopped`); `fleet-up` nur bei (Re)Create nötig.

**Ehrliche Grenze:** Während Kestra läuft, ist das Secret per `docker inspect` /
`docker exec … printenv` für jeden mit Docker-Zugriff lesbar — das ist inhärent bei
Env-Secrets und wird durch diesen Aufbau nicht verhindert. Der Schutz liegt in: nicht in git,
nicht als Klartext-Datei at rest, und Blast-Radius-Begrenzung (`from=`-Restriktion, s.u.).

### rsyncd.secrets

Host-only (gitignored, `chmod 600`). Format:

```
qnap_fleet:<PASSWORD>
```

> Hinweis: Diese Datei lag früher **im Klartext in der git-Historie** (kein git-crypt, da
> `.secrets` nicht von `.gitattributes` erfasst). Sie wurde aus dem Tracking entfernt,
> gitignored und das Passwort rotiert (auch im QNAP-rsync-Job). Der alte Blob bleibt in der
> git-Historie, bis optional per `git filter-repo` gepurged.

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

Der erzeugte Token muss anschließend **base64-kodiert** als `SECRET_NTFY_TOKEN` in die
age-Env (`~/.config/fleet/kestra.env.age`) auf dem Fleet Manager und Kestra via `fleet-up` neu gestartet werden
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
git-crypt unlock  # Inventories (inventory_*.ini) entschlüsseln
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

### 6. Secrets bereitstellen & Stack starten

Die Kestra-Secrets liegen **nicht** im Repo, sondern age-verschlüsselt unter
`~/.config/age/keys.txt.age` (Identity = Master-Key) und `~/.config/fleet/kestra.env.age`
(Secrets, inkl. ntfy-Token). `sudo apt install age` voranstellen.

**Fall A — Recovery (Regelfall: Backup vorhanden):** beide Dateien aus dem Backup
**zurückkopieren** — **nicht** neu erzeugen (sonst sind alle bestehenden Secrets verloren):
```bash
mkdir -p ~/.config/age ~/.config/fleet
# Quelle: restic-Restore (siehe "Restore aus Backup") ODER chezmoi (falls eingecheckt)
install -m600 <BACKUP>/keys.txt.age   ~/.config/age/keys.txt.age
install -m600 <BACKUP>/kestra.env.age ~/.config/fleet/kestra.env.age
```

**Fall B — nur echter Neuanfang (kein Backup mehr da):** neue Identity + neue Secrets erzeugen.
Die alten sind dann unwiederbringlich, und der Ansible-Key muss neu ausgerollt werden:
```bash
age-keygen | age -p -o ~/.config/age/keys.txt.age && chmod 600 ~/.config/age/keys.txt.age
fleet-genkey                          # neue age-Env + neuer Ansible-Key (gibt Public Key aus)
# danach Flow rotate_ansible_pubkey mit dem ausgegebenen Public Key
```

> Tipp: Wer Recovery vereinfachen will, kann `keys.txt.age` (passphrase-verschlüsselt) und
> `kestra.env.age` (an den Recipient verschlüsselt) in ein **privates** Repo (z.B. chezmoi)
> einchecken — Entschlüsselung braucht weiterhin die Passphrase. Sicherheit = Passphrase-Stärke
> + Repo-Vertraulichkeit; idealerweise Identity und Env nicht im selben Repo bündeln.

```bash
fleet-up        # entschlüsselt die Secrets (Passphrase) und startet den Stack
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

## Restore aus Backup (restic)

Der Fleet Manager sichert sich selbst per `fleet-backup` (restic) nach
`/var/backup/fleet_backups/fleet-mgr/` — dieses Verzeichnis zieht der QNAP-rsync mit. So
spielst du den Manager auf einem frischen Host aus diesem restic-Repo zurück.

### Voraussetzungen
```bash
sudo apt install age restic
```
- **Repo-Passwort** `/root/.restic_pw` = das **extern gesicherte** Passwort (liegt bewusst
  NICHT im Backup selbst):
  ```bash
  sudo sh -c 'umask 077; printf "%s" "<REPO_PW>" > /root/.restic_pw; chmod 600 /root/.restic_pw'
  ```
- **Zugriff auf das Repo**: das Verzeichnis `fleet-mgr/` vom QNAP zurückkopieren/mounten,
  z.B. nach `/var/backup/fleet_backups/fleet-mgr` (oder direkt auf den QNAP-Pfad zeigen).

### Restore
```bash
export RESTIC_REPOSITORY=/var/backup/fleet_backups/fleet-mgr   # oder QNAP-Pfad
export RESTIC_PASSWORD_FILE=/root/.restic_pw

sudo -E restic snapshots                      # Snapshots ansehen
sudo -E restic restore latest --target /tmp/restore   # erst in einen Zwischenordner
```
Der Snapshot enthält die Originalpfade:
`/home/decebu/docker-stacks/stacks/fleet-manager`, `/home/decebu/.config/age`,
`/home/decebu/.config/fleet`. Aus `/tmp/restore` an die Originalstellen kopieren (Rechte
erhalten):
```bash
sudo rsync -aAX /tmp/restore/home/decebu/docker-stacks/stacks/fleet-manager/ \
                /home/decebu/docker-stacks/stacks/fleet-manager/
sudo rsync -aAX /tmp/restore/home/decebu/.config/age/   /home/decebu/.config/age/
sudo rsync -aAX /tmp/restore/home/decebu/.config/fleet/ /home/decebu/.config/fleet/
sudo chown -R decebu:decebu ~/.config/age ~/.config/fleet
```
> Damit sind age-Identity **und** age-Env (Schritt 6, Fall A) bereits wiederhergestellt —
> direkt mit `fleet-up` weiter. `--target /` statt `/tmp/restore` würde direkt an die
> Originalstellen schreiben (nur auf wirklich frischem Host empfehlenswert).

### Danach Stack starten

`fleet-up` braucht die age-Passphrase, daher interaktiv (kein ssh-Einzeiler):

```bash
ssh fleet-mgr
fleet-up        # Passphrase eingeben -> Stack startet
```

Alle Kestra-Flows und Schedules sind sofort aktiv, da die `database.mv.db` vollständig wiederhergestellt wurde.

---

## Secret-Rotation

Die Kestra-Secrets liegen age-verschlüsselt in `~/.config/fleet/kestra.env.age` (nicht in git,
kein Klartext at rest) und werden zur Laufzeit per `fleet-up` injiziert. Rotation heißt daher:
Geheimnis erzeugen → in die age-Env schreiben (verschlüsselt an den Recipient) → `fleet-up`
(recreate) → verifizieren. Nur `fleet-up` braucht die Passphrase.

### ntfy-Token rotieren

Unkritisch und schnell — der Token hat nur Publish-Recht auf das Topic `fleet_backups`.

```bash
# 1. VPS: Schreibrecht sicherstellen + neuen Token erzeugen
docker exec ntfy ntfy access kuma-bot fleet_backups rw
docker exec ntfy ntfy token add kuma-bot      # -> tk_NEU

# 2. age-Env: nur die NTFY-Zeile ersetzen (entschlüsseln -> sed -> neu verschlüsseln,
#    kein Klartext-File; Passphrase nötig). <RECIPIENT> = age1… Public Recipient.
ID=~/.config/age/keys.txt.age; ENV=~/.config/fleet/kestra.env.age
age -d -i "$ID" "$ENV" \
  | sed "s|^SECRET_NTFY_TOKEN=.*|SECRET_NTFY_TOKEN=$(printf '%s' 'tk_NEU' | base64)|" \
  | age -r <RECIPIENT> -o "$ENV.new" && mv "$ENV.new" "$ENV" && chmod 600 "$ENV"

# 3. Kestra mit neuem Token neu erstellen
fleet-up

# 4. Verifizieren (über den WireGuard-Pfad) -> erwartet 200
docker exec fleet-wg-client sh -c \
  'curl -s -o /dev/null -w "%{http_code}\n" \
   -H "Authorization: Bearer tk_NEU" <NTFY_PUBLIC_ENDPOINT>/v1/account'

# 5. Alten Token serverseitig entwerten
docker exec ntfy ntfy token list kuma-bot
docker exec ntfy ntfy token remove kuma-bot tk_ALT
```

### Ansible-SSH-Key rotieren

Sicherheitskritisch (root auf allen Hosts via `become`). Vorgehen mit Überlappung — neuer
Public Key wird verteilt, **bevor** der alte entfernt wird. Mit `fleet-genkey`/`fleet-up`
ist kein `.env_encoded`-Editieren und kein manuelles base64 mehr nötig.

```bash
# 1. Neues Keypair erzeugen + age-Env schreiben (übernimmt bestehende Secrets wie ntfy aus
#    der age-Env, Passphrase nötig). Gibt NUR den neuen Public Key aus, Privatteil nie im Klartext.
fleet-genkey
```

**2. Pubkey ausrollen** — Kestra-UI → `dev.fleet_management` → `rotate_ansible_pubkey` →
**Execute**, Input `new_public_key` = der von `fleet-genkey` ausgegebene Public Key. Rollt mit
`from="10.10.30.10"` aus. Lauf muss grün sein (ein nicht erreichbarer Host = Rollout
unvollständig → Host fixen + erneut ausführen, idempotent).

```bash
# 3. Kestra mit neuem Key neu erstellen (Passphrase)
fleet-up
```

**4. Verifizieren:** Flow `hello_fleet` → `pong` auf allen Hosts (jetzt mit neuem Key).

**5. Erst NACH erfolgreichem Test:** alten Pubkey entfernen — Kestra-UI →
`remove_ansible_pubkey`, Input `old_public_key` = der **alte** Public Key. Den findest du via
Flow `diag_authkeys` (zeigt alle authorized_keys je Host): es ist die Zeile **ohne**
`from="10.10.30.10"`-Präfix. Der Flow hat einen Selbst-Aussperr-Schutz (vergleicht gegen den
aktiven Key).

> Falls der alte Key nicht mehr verfügbar ist, muss der neue Public Key alternativ per
> Konsolen-/Out-of-Band-Zugriff in `~kestra/.ssh/authorized_keys` jedes Hosts eingetragen
> werden.

#### Kestra-Flow `rotate_ansible_pubkey`

Übernimmt Schritt 2 (Public-Key-Rollout). Manuell ausgelöst (keine Trigger), Namespace
`dev.fleet_management`, ein Input `new_public_key`. Der Flow **addiert nur**
(`state=present`) und entfernt nie einen Key; der private Key wird nie berührt.

> **Wichtig (Pebble-Fallstrick):** Das Playbook darf **nicht inline** im Flow stehen.
> Kestra rendert `inputFiles`-Inhalte mit seiner Template-Engine (Pebble) und stolpert über
> Ansible-Ausdrücke wie `{{ lookup(...) }}` / `{{ inventory_hostname }}`
> (`Function or Macro [lookup] does not exist`). Kestras Pebble kennt auch **kein**
> `{% raw %}`. Lösung: das Playbook als **Namespace-File** ablegen und per
> `{{ read('...') }}` laden — `read()`-Inhalt wird nicht erneut gerendert. Die anderen
> `inputFiles` (`read()`, `secret()`, `inputs`) sollen ja gerendert werden und bleiben so.

Playbook-Datei `_files/fleet_rotate_pubkey.yaml` (Namespace-File neben den übrigen Playbooks):

```yaml
- name: Roll out new Ansible public key to fleet
  hosts: fleet
  become: true
  gather_facts: false
  tasks:
    - name: Ensure new public key present for user kestra
      ansible.posix.authorized_key:
        user: kestra
        state: present
        key: "{{ lookup('file', 'new_key.pub') }}"
        # Blast-Radius: Key nur von der Manager-VPN-IP nutzbar
        key_options: 'from="10.10.30.10"'
    - name: Report
      ansible.builtin.debug:
        msg: "OK: neuer Public Key auf {{ inventory_hostname }} hinterlegt."
```

Flow `rotate_ansible_pubkey`:

```yaml
id: rotate_ansible_pubkey
namespace: dev.fleet_management
description: |
  Rollt einen neuen Ansible-Public-Key auf alle Fleet-Hosts aus, authentifiziert mit dem
  AKTUELLEN Key. ADDIERT nur (state=present). Nach Erfolg neuen Key via fleet-genkey in die
  age-Env schreiben und Kestra via fleet-up neu erstellen.

inputs:
  - id: new_public_key
    type: STRING
    description: "Inhalt der neuen .pub-Datei (eine Zeile, z.B. 'ssh-ed25519 AAAA... kestra-fleet')"

tasks:
  - id: rollout_pubkey
    type: io.kestra.plugin.ansible.cli.AnsibleCLI
    taskRunner:
      type: io.kestra.plugin.scripts.runner.docker.Docker
      image: cytopia/ansible:latest-tools
      networkMode: "container:fleet-wg-client"
    env:
      ANSIBLE_HOST_KEY_CHECKING: "False"
    inputFiles:
      inventory.ini: "{{ read('inventory_fleet.ini') }}"
      ansible_key: "{{ secret('ANSIBLE_PRIVATE_KEY') }}"
      new_key.pub: "{{ inputs.new_public_key }}"
      rollout.yml: "{{ read('fleet_rotate_pubkey.yaml') }}"
    commands:
      - chmod 600 ansible_key
      - >-
        ansible-playbook -i inventory.ini rollout.yml
        --user kestra --private-key ansible_key
        -e "ansible_python_interpreter=/usr/bin/python3"
```

#### Kestra-Flow `remove_ansible_pubkey`

Gegenstück zu `rotate_ansible_pubkey`: entfernt einen alten Public Key (`state=absent`).
Erst **nach** erfolgreichem Key-Wechsel ausführen (neuer Key aktiv). Eingebauter
**Selbst-Aussperr-Schutz**: leitet aus dem aktiven Private Key per `ssh-keygen -y` den
zugehörigen Public Key ab und bricht ab, falls `old_public_key` diesem entspricht. Playbook
ebenfalls als Namespace-File (gleicher Pebble-Grund wie oben).

Playbook-Datei `_files/fleet_remove_pubkey.yaml`:

```yaml
- name: Remove old Ansible public key from fleet
  hosts: fleet
  become: true
  gather_facts: false
  tasks:
    - name: Ensure old public key absent for user kestra
      ansible.posix.authorized_key:
        user: kestra
        state: absent
        key: "{{ lookup('file', 'old_key.pub') }}"
    - name: Report
      ansible.builtin.debug:
        msg: "OK: alter Public Key auf {{ inventory_hostname }} entfernt (falls vorhanden)."
```

Flow `remove_ansible_pubkey`:

```yaml
id: remove_ansible_pubkey
namespace: dev.fleet_management
description: |
  Entfernt einen alten Ansible-Public-Key von allen Fleet-Hosts (state=absent), authentifiziert
  mit dem AKTUELLEN Key. Schutz gegen Selbst-Aussperrung: bricht ab, wenn der zu entfernende Key
  dem aktiv abgeleiteten Key entspricht. Erst NACH erfolgreichem Key-Wechsel ausfuehren.

inputs:
  - id: old_public_key
    type: STRING
    description: "Inhalt der ALTEN .pub-Datei (eine Zeile), die entfernt werden soll"

tasks:
  - id: remove_pubkey
    type: io.kestra.plugin.ansible.cli.AnsibleCLI
    taskRunner:
      type: io.kestra.plugin.scripts.runner.docker.Docker
      image: cytopia/ansible:latest-tools
      networkMode: "container:fleet-wg-client"
    env:
      ANSIBLE_HOST_KEY_CHECKING: "False"
    inputFiles:
      inventory.ini: "{{ read('inventory_fleet.ini') }}"
      ansible_key: "{{ secret('ANSIBLE_PRIVATE_KEY') }}"
      old_key.pub: "{{ inputs.old_public_key }}"
      remove.yml: "{{ read('fleet_remove_pubkey.yaml') }}"
    commands:
      - chmod 600 ansible_key
      # Selbst-Aussperr-Schutz: zu entfernender Key darf nicht der aktive Key sein
      - >-
        CUR=$(ssh-keygen -y -f ansible_key | awk '{print $1" "$2}');
        OLD=$(awk '{print $1" "$2}' old_key.pub);
        if [ "$CUR" = "$OLD" ]; then
        echo "ABBRUCH: old_public_key entspricht dem AKTUELL AKTIVEN Key - das wuerde dich aussperren.";
        exit 1; fi
      - >-
        ansible-playbook -i inventory.ini remove.yml
        --user kestra --private-key ansible_key
        -e "ansible_python_interpreter=/usr/bin/python3"
```

### Wann rotieren?

- **ntfy-Token:** bei `401`-Fehlern (siehe Troubleshooting) oder Verdacht auf Leak.
- **Ansible-Key:** bei Verdacht auf Kompromittierung, ausgeschiedenem Zugriff, oder wenn
  der Key (z.B. über Logs, Tool-Kontext, Screenshares) außerhalb der age-Env
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

# 2. age-Env: NTFY-Zeile ersetzen (entschlüsseln -> sed -> neu verschlüsseln; Passphrase nötig)
ID=~/.config/age/keys.txt.age; ENV=~/.config/fleet/kestra.env.age
age -d -i "$ID" "$ENV" \
  | sed "s|^SECRET_NTFY_TOKEN=.*|SECRET_NTFY_TOKEN=$(printf '%s' 'tk_NEU' | base64)|" \
  | age -r <RECIPIENT> -o "$ENV.new" && mv "$ENV.new" "$ENV" && chmod 600 "$ENV"

# 3. Kestra mit neuem Token neu erstellen
fleet-up

# 4. Verifizieren (über den WireGuard-Pfad, den Kestra nutzt): erwartet 200
docker exec fleet-wg-client sh -c \
  'curl -s -o /dev/null -w "%{http_code}\n" \
   -H "Authorization: Bearer tk_NEU" <NTFY_PUBLIC_ENDPOINT>/v1/account'
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
fleet-up kestra --force-recreate
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
