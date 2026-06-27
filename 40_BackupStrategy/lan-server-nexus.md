# LAN-Server-Backup: Nexus

Dieses Dokument beschreibt die Backup-Strategie fuer Nexus als aktuellen LAN-Server. Nexus ist nicht Teil der Fleet und wird daher separat abgesichert.

Der operative Dienstekatalog fuer Nexus steht in `../20_Server/nexus-dienste.md`.

## Ziel

Nexus soll als einzelner Homelab-Server wiederherstellbar sein, bevor weitere produktive Dienste wie paperless-ngx mit echten Daten betrieben werden.

## Aktueller Umsetzungsstand

Stand: 2026-06-27

Das erste Nexus-Backup ist eingerichtet und getestet.

| Bereich | Status |
|---|---|
| Restic | installiert, Repository initialisiert |
| QNAP-Ziel | per systemd Mount Unit eingebunden |
| Backup-Script | `/usr/local/sbin/nexus-backup` |
| systemd Timer | `nexus-backup.timer`, taeglich gegen 02:30 Uhr |
| Authentik-Dump | erfolgreich in `/var/backups/nexus/authentik/` |
| Manifest | erfolgreich in `/var/backups/nexus/manifest/` |
| Restore-Test | erfolgreich nach `/tmp/nexus-restore-test` |
| Snapshots | erster Snapshot erfolgreich erstellt |

Backup-Ziel:
- Restic-Backup von Nexus auf ein QNAP-Backupziel
- Restore-Test als Pflichtbestandteil
- Keine Secrets im Git
- Keine Abhaengigkeit von `wg0` oder Fleet-Management

Aktuelles Repository:
- Mountpoint: `/mnt/backup-nexus`
- Restic-Repository: `/mnt/backup-nexus/restic-repo`
- Restic-Passwortdatei: `/root/.restic_pw`
- SMB-Credentials: `/root/.config/nexus-backup/qnap-smb.credentials`

## Aktueller Schutzbedarf

| Bereich | Pfad / Objekt | Prioritaet | Grund |
|---|---|---|---|
| Docker Stacks | `/home/decebu/docker-stacks/` | Hoch | Compose-Dateien und Dienstkonfigurationen |
| WireGuard | `/etc/wireguard/` | Hoch | Schluessel und VPN-Konfigurationen, sofern weiter benoetigt |
| Traefik | Stack-Konfiguration und Zertifikate | Hoch | Reverse Proxy und TLS-Zustand |
| Authentik | Datenbank-Volume und Konfiguration | Hoch | IAM, Benutzer und Einstellungen |
| Unbound | Stack-Konfiguration | Mittel | Lokale DNS-Funktion |
| Docker Volumes | benoetigte Volumes unter `/var/lib/docker/volumes/` | Hoch | Persistente Dienstdaten |
| paperless-ngx | Media, Datenbank, Redis-Konfiguration, Consume, Export | Hoch | Kritische Dokumentenablage nach Installation |

Die konkrete Dienst-Matrix wird im Nexus-Dienstekatalog gefuehrt. Dieses Dokument beschreibt daraus die Backup-Strategie.

## Standard-Design

Nexus pusht Backups per Restic auf ein QNAP-Ziel.

Grundsaetze:
- Restic-Repository je Server, z. B. `nexusNG`
- Repository-Passwort in `/root/.restic_pw`, Rechte `600`
- QNAP-Zugangsdaten nicht im Git
- Backup-Job per systemd Timer
- Retention: 7 taegliche, 4 woechentliche und 6 monatliche Snapshots als Startwert
- Nach jedem Strukturwechsel Restore-Test durchfuehren

## Umsetzung

Angelegte Komponenten:
- `/usr/local/sbin/nexus-backup`
- `/etc/systemd/system/nexus-backup.service`
- `/etc/systemd/system/nexus-backup.timer`
- `/etc/systemd/system/mnt-backup\x2dnexus.mount`
- `/mnt/backup-nexus/`
- `/var/backups/nexus/authentik/`
- `/var/backups/nexus/manifest/`

Secret-Dateien:
- `/root/.restic_pw`
- `/root/.config/nexus-backup/qnap-smb.credentials`

Diese Dateien sind host-lokal, root-only und duerfen nicht ins Git.

## Include- und Exclude-Regeln

Start-Includes:
- `/home/decebu/docker-stacks/`
- `/var/backups/nexus/`

Start-Excludes:
- `.git`
- `node_modules`
- `*.log`
- `__pycache__`

Noch zu pruefen:
- ob Traefik-Zertifikate vollstaendig ueber `/home/decebu/docker-stacks/` erfasst sind
- ob `/etc/wireguard/` weiter benoetigt wird und in den Backupumfang aufgenommen werden soll
- welche Docker-Volumes spaeter fuer paperless-ngx konkret gesichert werden muessen

## Restore-Test

Ein Backup gilt erst als belastbar, wenn ein Restore-Test dokumentiert wurde.

Durchgefuehrter Test:
- Restore von `docker-stacks` und Manifest nach `/tmp/nexus-restore-test`
- Ergebnis: erfolgreich
- Wiederhergestellte Elemente: 128 Dateien/Verzeichnisse

Noch offen:
- Restore eines Authentik-Dumps in eine Testdatenbank
- Restore der Traefik-Konfiguration inklusive Zertifikatsstrategie
- Restore eines kleinen paperless-Testdatensatzes, bevor echte Dokumente importiert werden

## Paperless-Gate

paperless-ngx darf auf Nexus erst produktiv mit echten Dokumenten genutzt werden, wenn folgende Punkte erledigt sind:
- Backup-Pfade fuer Compose, PostgreSQL, Redis-Konfiguration, Media, Consume und Export dokumentiert
- erster Backup-Lauf erfolgreich
- Restore-Test mit Testdokument erfolgreich
- Aufbewahrung und QNAP-Ziel dokumentiert

Ein technischer Testbetrieb ohne echte Dokumente kann separat geplant werden, gilt aber nicht als Produktivstart.

## Offene Punkte

> **Offen:** Authentik-Dump in eine Testdatenbank restoren und Ergebnis dokumentieren.

> **Offen:** Traefik-Zertifikatspfad und `acme.json` gegen den Backupumfang pruefen.

> **Offen:** Klaeren, ob `/etc/wireguard/` weiterhin benoetigt und in das Backup aufgenommen werden soll.

> **Offen:** Monitoring oder Benachrichtigung fuer fehlgeschlagene Nexus-Backups einrichten.

> **Offen:** Paperless-spezifischen Restore-Test vor Produktivstart durchfuehren.
