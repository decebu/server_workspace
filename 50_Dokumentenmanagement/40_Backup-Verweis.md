# Backup-Verweis für DMS

## Kontext
paperless-ngx speichert Dokumente und Metadaten, die Teil der Homelab-Backup-Strategie sein sollten.

## Empfehlung
- `paperless-ngx` Daten und Konfiguration sollten als Teil der allgemeinen Homelab-Backups behandelt werden.
- Die vollständige Backup-Strategie wird in `40_BackupStrategy/` dokumentiert.
- Dieser Bereich enthält bewusst nur den Verweis. Retention, Restore-Tests, Datenbank-Dumps und Medienverzeichnisse werden separat in der Backup-Strategie ausgearbeitet.
- Produktivbetrieb mit echten Dokumenten ist erst nach dokumentiertem paperless-Backup und erfolgreichem paperless-Restore-Test freigegeben.

## Offen
> **Offen:** Backup-Strategie und Restore-Test fuer paperless-ngx in `../40_BackupStrategy/lan-server-nexus.md` konkretisieren.
