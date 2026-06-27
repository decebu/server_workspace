# Dokumentenmanagement

Dieses Verzeichnis enthält die Dokumentation für die geplante DMS-Strategie im Homelab.

Die Inhalte umfassen:
- `paperless-ngx` als Dokumentenarchiv und OCR-Lösung
- `Obsidian` als Wissenslayer und Kontextmanagement
- `LLM-Pipeline` als lokale Dokumentenanalyse auf einem Mac Mini M4 Pro
- Migrationsstrategie von `ecoDMS`
- Datenschutzprinzip: Dokumente verlassen nicht das Heimnetz

Die Strategie basiert auf dem vollständigen ecoDMS-Export mit 1.456 Dokumenten aus den Jahren 2006 bis 2026. Der aktive Migrationsfokus liegt auf 808 Dokumenten ab `2020-01-01`; der ältere Bestand wird als suchbarer Altbestand übernommen.

## Strukturentscheidung
Die DMS-Strategie wird als Top-Level-Bereich `50_Dokumentenmanagement/` geführt.

Begründung: Das Thema ist kein reines Software-Setup und auch kein reiner Serverbetrieb. Es verbindet Dokumentenarchiv, Wissensmanagement, lokale LLM-Verarbeitung, Datenstrategie, Datenschutz und eine spätere Hardware-Entscheidung. Eine Ablage in `2_Software/` oder `3_Server/` würde diese fachliche Klammer aufsplitten. Die Backup-Frage wird nur verlinkt und bleibt in `40_BackupStrategy/`.

## Produktiv-Gate
paperless-ngx darf erst produktiv mit echten Dokumenten starten, wenn die Backup-Strategie für Nexus und paperless-ngx dokumentiert und ein Restore-Test durchgeführt wurde.

Siehe:
- `../40_BackupStrategy/lan-server-nexus.md`
- `40_Backup-Verweis.md`

## Struktur
- `00_Übersicht.md` - Gesamtstrategie und Phasenplan
- `10_paperless-ngx.md` - Klassifizierung, Migration, Konfiguration
- `20_Obsidian.md` - Vault-Struktur, Templates, Workflow
- `30_LLM-Pipeline.md` - Lokale Ollama-Strategie, Extraktion, Datenschutz
- `40_Backup-Verweis.md` - Hinweis auf Backup-Integration in `40_BackupStrategy/`
