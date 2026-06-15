# Fleet Client Konfiguration

Diese Datei beschreibt, wie ein **Fleet-Client** (ein vom Fleet Manager verwalteter Host)
eingerichtet wird. Den Manager selbst (Kestra/Ansible/rsync-Aggregat) beschreibt
`Fleet_Manager_Setup.md`.

## Was ist ein Fleet-Client?

Ein Host, den der Fleet Manager per Ansible über das WireGuard-VPN (Range `10.10.30.x`)
erreicht und auf dem er automatisiert:

- OS-Updates ausführt (`update_fleet_devices` → `fleet_upgrade.yaml`)
- ein lokales Restic-Backup erstellt (`restic_backup_fleet` → `fleet_restic_backup.yaml`)
- dessen Restic-Repo anschließend auf den Manager zieht (`pull_backups_to_manager`)

Damit das funktioniert, muss jeder Client die unten genannten Voraussetzungen erfüllen und
im Inventory `inventory_fleet.ini` (Gruppe `[fleet]`) eingetragen sein.

---

## Voraussetzungen (jeder Client)

| Voraussetzung            | Detail                                                              |
|--------------------------|---------------------------------------------------------------------|
| Betriebssystem           | Debian-basiert (Raspberry Pi OS / Debian / Ubuntu)                  |
| Netzwerk                 | Erreichbar über WireGuard-VPN, feste VPN-IP `10.10.30.x`            |
| Python                   | `python3` unter `/usr/bin/python3` (für Ansible)                    |
| Ansible-Benutzer         | User `kestra` mit SSH-Key-Login + passwortloses `sudo`             |
| Restic-Passwortdatei     | `/root/.restic_pw` (Repo-Passwort, `chmod 600`, root)              |

Restic selbst muss **nicht** vorinstalliert sein — das Backup-Playbook installiert es bei
Bedarf und initialisiert das Repo unter `/var/backup/restic-repo`.

---

## 1. VPN-Anbindung

Der Client muss als WireGuard-Peer im VPN hängen und eine IP aus `10.10.30.x` haben.
Einrichtung siehe `VPN_Firezone_installation.md` bzw. `VPN_RPI_playbook.md`.

Test vom Fleet Manager aus:

```bash
docker exec fleet-wg-client ping -c1 10.10.30.<X>
```

---

## 2. Ansible-Benutzer `kestra`

Alle Flows verbinden sich als `--user kestra` und nutzen `become: true` (root). Der User
braucht daher SSH-Key-Login **und** passwortloses sudo.

```bash
# Auf dem Client (als root/sudo):
sudo useradd -m -s /bin/bash kestra

# SSH-Verzeichnis vorbereiten
sudo install -d -m 700 -o kestra -g kestra /home/kestra/.ssh

# Ansible-Public-Key hinterlegen (entspricht SECRET_ANSIBLE_PRIVATE_KEY auf dem Manager)
echo 'ssh-ed25519 AAAA... kestra-fleet' | sudo tee -a /home/kestra/.ssh/authorized_keys
sudo chmod 600 /home/kestra/.ssh/authorized_keys
sudo chown -R kestra:kestra /home/kestra/.ssh

# Passwortloses sudo (Ansible läuft nicht-interaktiv, ohne become-Passwort)
echo 'kestra ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/kestra
sudo chmod 440 /etc/sudoers.d/kestra
```

> Der hinterlegte Public Key muss zum privaten Ansible-Key des Managers passen. Wird der
> Key rotiert, übernehmen das die Flows `rotate_ansible_pubkey` / `remove_ansible_pubkey`
> (siehe `Fleet_Manager_Setup.md`, Abschnitt **Secret-Rotation**).

---

## 3. Restic-Passwortdatei

Das Backup-Playbook **bricht ab**, wenn `/root/.restic_pw` fehlt. Die Datei enthält das
Passwort des Restic-Repos und muss auf jedem Client identisch hinterlegt sein (sonst lassen
sich die gepullten Repos nicht gemeinsam verwalten).

```bash
# Auf dem Client (als root):
printf '%s' '<RESTIC_REPO_PASSWORD>' > /root/.restic_pw
chmod 600 /root/.restic_pw
```

Das Repo unter `/var/backup/restic-repo` wird beim ersten Lauf automatisch initialisiert
(`restic init`). Retention wird zentral im Playbook gesteuert: `--keep-daily 7 --keep-weekly 4`.

---

## 4. Inventory-Eintrag

Eintrag in `inventory_fleet.ini` (git-crypt, im Manager-Repo unter
`stacks/fleet-manager/kestra/dev/fleet_management/_files/`), Gruppe `[fleet]`:

```ini
10.10.30.<X> is_vps=false is_ha_client=false backup_paths="/home/decebu/docker-stacks /etc/wireguard"
```

### Host-Variablen

| Variable        | Werte / Beispiel                              | Bedeutung                                                        |
|-----------------|-----------------------------------------------|------------------------------------------------------------------|
| `backup_paths`  | `"/home/decebu/docker-stacks /etc/wireguard"` | Leerzeichen-getrennte Liste der zu sichernden Pfade (Pflicht)    |
| `is_vps`        | `true` / `false`                              | `true` aktiviert den Firezone-Postgres-Dump (s.u.)               |
| `is_ha_client`  | `true` / `false`                              | Markiert Home-Assistant-Clients (HA-Pfade in `backup_paths`)     |
| `pg_container`  | `firezone-postgres`                           | Nur VPS: Name des Postgres-Containers für `pg_dumpall`           |
| `ansible_port`  | `54322`                                       | Abweichender SSH-Port (nur falls nötig, z.B. VPS)                |

`backup_paths` ist der eigentliche Schalter dafür, **was** gesichert wird — die übrigen
Flags steuern Sonderlogik.

---

## 5. Rollentypen

### Standard-Client (RPi)

```ini
10.10.30.21 is_vps=false is_ha_client=false backup_paths="/home/decebu/docker-stacks /etc/wireguard"
```

### Home-Assistant-Client

```ini
10.10.30.20 is_vps=false is_ha_client=true backup_paths="/home/decebu/docker-stacks /etc/wireguard /home/decebu/homeassistant"
```

### VPS (Sonderfall)

Der VPS betreibt Firezone (WireGuard-Server) inkl. PostgreSQL. Zusätzlich zu den Pfaden wird
ein DB-Dump erstellt: `docker exec <pg_container> pg_dumpall -U postgres` wird per `--stdin`
in Restic geleitet.

```ini
10.10.30.1 ansible_port=54322 is_vps=true is_ha_client=false pg_container="firezone-postgres" backup_paths="/home/decebu/docker-stacks /etc/wireguard /etc/ufw"
```

Voraussetzung auf dem VPS: Docker läuft und der Container `firezone-postgres` ist vorhanden;
der User `kestra` darf `docker exec` ausführen (root via sudo/become ist gegeben).

---

## 6. Onboarding-Checkliste

1. [ ] Client ist im VPN, IP `10.10.30.<X>` vom Manager pingbar
2. [ ] User `kestra` angelegt, Ansible-Public-Key in `authorized_keys`
3. [ ] Passwortloses sudo für `kestra` (`/etc/sudoers.d/kestra`)
4. [ ] `/root/.restic_pw` vorhanden (`chmod 600`)
5. [ ] (nur VPS) Container `firezone-postgres` läuft
6. [ ] Inventory-Eintrag in `inventory_fleet.ini` mit korrekten Variablen
7. [ ] Erreichbarkeit testen: Flow `hello_fleet` (ansible ping) → `pong`
8. [ ] Erster Backup-Lauf: Flow `restic_backup_fleet` manuell starten, grün?
9. [ ] Pull prüfen: `/var/backup/fleet_backups/<host>/` auf dem Manager gefüllt?

---

## 7. Verifikation

```bash
# Erreichbarkeit (Manager-seitig, über WireGuard):
#   Kestra-UI -> dev.fleet_management -> hello_fleet -> Execute  (erwartet: pong)

# Auf dem Client: Repo + letzte Snapshots prüfen
sudo restic -r /var/backup/restic-repo --password-file /root/.restic_pw snapshots
```

---

## 8. Troubleshooting

### Host „unreachable" im Ansible-Lauf

```bash
# VPN-Handshake vom Manager prüfen:
docker exec fleet-wg-client wg show
docker exec fleet-wg-client ping -c1 10.10.30.<X>
# SSH-Login als kestra testen (vom Manager-netns):
docker exec fleet-wg-client ssh -i <key> kestra@10.10.30.<X> 'echo ok'
```

### „Die Datei /root/.restic_pw fehlt auf diesem Host!"

Passwortdatei anlegen (Abschnitt 3) — das Backup-Playbook bricht ohne sie bewusst ab.

### become/sudo schlägt fehl

`/etc/sudoers.d/kestra` prüfen (`kestra ALL=(ALL) NOPASSWD:ALL`, Datei `chmod 440`).
Ansible läuft nicht-interaktiv und kann kein become-Passwort eingeben.

### VPS: Postgres-Dump scheitert

```bash
# Container vorhanden + Name korrekt (== pg_container im Inventory)?
docker ps --format '{{.Names}}' | grep -i postgres
# Manuell testen:
docker exec firezone-postgres pg_dumpall -U postgres | head
```

---

## Verwandte Dokumente

- `Fleet_Manager_Setup.md` — Manager (Kestra/Ansible/rsync), Flows, Secret-Rotation
- `VPN_Firezone_installation.md`, `VPN_RPI_playbook.md` — VPN-Anbindung der Clients
