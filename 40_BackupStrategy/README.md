# Backup-Strategie

Dieses Verzeichnis enthält die Backup-Dokumentation für das Homelab.

## Inhalt
- `clients/` – Backup-Profile und Konfigurationen für Clients
- `docs/` – Anleitungen für QNAP und Windows Backup
- `monitoring/` – Überwachung und Prüfung von Backup-Prozessen
- `lan-server-nexus.md` – LAN-Server-Backup für Nexus

## Strategie-Ebenen

Die Backup-Dokumentation unterscheidet bewusst zwischen mehreren Bereichen.

| Bereich | Status | Ziel |
|---|---|---|
| Fleet Management | Vorhandene Strategie | Automatisierte Backups und Pull-Aggregation für Fleet-Hosts |
| LAN-Server | Nexus Basis mit ntfy eingerichtet | Nexus und spätere LAN-Server mit Restic auf QNAP sichern |
| Windows-Clients | Vorhandene Basis | Restic/resticprofile für einzelne Clients |
| paperless-ngx | Gesperrt bis Backup steht | Dokumente, Datenbank und Konfiguration vor Produktivstart sichern |

## LAN-Server-Standard

Für Nexus und spätere LAN-Server gilt als Standard:
- Restic-Backup vom Server auf ein QNAP-Backupziel
- Restic-Passwort host-lokal, root-only, nicht im Git
- ntfy-Benachrichtigung fuer erfolgreiche und fehlgeschlagene Backup-Laeufe
- Include-/Exclude-Pfade dokumentieren
- Retention und Zeitplan dokumentieren
- Restore-Test dokumentieren
- Monitoring oder Statusmeldung vor produktiven kritischen Diensten klaeren

Nexus ist nicht Teil der Fleet. Die vorhandene Fleet-Backup-Strategie wird nicht als stillschweigende Grundlage für Nexus verwendet.

Aktueller Nexus-Stand:
- Restic-Repository eingerichtet
- systemd Timer eingerichtet
- Authentik-Dump und Manifest werden erstellt
- einfacher Restore-Test fuer `docker-stacks` und Manifest erfolgreich
- ntfy-Benachrichtigung fuer Erfolg und Fehler erfolgreich getestet
- dienstspezifische Restore-Tests sind noch offen

## Verbindung zur DMS-Strategie
`paperless-ngx` darf produktiv erst starten, wenn die Backup- und Restore-Anforderungen für Nexus und paperless-ngx dokumentiert sind.

Siehe hierzu auch:
- `lan-server-nexus.md`
- `../20_Server/nexus-dienste.md`
- `../50_Dokumentenmanagement/40_Backup-Verweis.md`

> **Offen:** Authentik-, Traefik- und paperless-spezifische Restore-Tests ergaenzen.
