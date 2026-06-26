# DMS-Strategie Übersicht

## Ziel
Diese Dokumentation beschreibt die Umsetzung einer Dokumentenmanagement-Strategie im persönlichen Homelab.

Das Ziel ist eine zweistufige Architektur:
- `paperless-ngx` als Dokumentenarchiv für Speicherung, OCR und Volltextsuche
- `Obsidian` als Wissenslayer für Kontext, Aufgaben und Zusammenhänge

```text
Obsidian Vault
  Wissen, Kontext, Fristen, Zusammenhänge
        |
        | Deep Links
        v
paperless-ngx
  Dokumentenarchiv, OCR, Volltextsuche
```

Zusätzlich wird eine lokale LLM-Pipeline auf einem Mac Mini M4 Pro vorgesehen, um Dokumente automatisiert zu analysieren und strukturierte Notizen zu erzeugen.

## Ausgangslage
- Stand der Strategie ist Juni 2026.
- ecoDMS ist seit rund 10 Jahren im Einsatz und soll wegen laufender Lizenzkosten perspektivisch ersetzt werden.
- Der Bestand umfasst 1.456 Dokumente aus den Jahren 2006 bis 2026.
- Der Migrationsfokus liegt auf 808 Dokumenten ab dem Schnittdatum `2020-01-01`.
- Der Altbestand vor 2020 umfasst rund 650 Dokumente und wird zunächst nur grob importiert.
- Die bisherige Ordnerstruktur ist historisch gewachsen und soll durch Korrespondenten, Dokumenttypen, Tags und ein Personenfeld ersetzt werden.
- Rund 41 Prozent der Dokumente liegen in einem unspezifischen Bereich `Allgemeines`.
- Gesundheitsdokumente sind auf fünf verschiedene Ordner verteilt.
- Immobilienunterlagen liegen in drei verschiedenen Hauptordnern.
- Personen wurden bisher teilweise als Ordner statt als Metadaten geführt.

## Rollenverteilung
- `paperless-ngx` beantwortet: Wo liegt das Dokument und was steht wörtlich darin?
- `Obsidian` beantwortet: Was bedeutet das Dokument, welche Fristen entstehen daraus und welche Zusammenhänge bestehen?
- Die lokale LLM-Pipeline erzeugt strukturierte Notizen aus OCR-Texten, ohne Originaldokumente an externe Dienste zu senden.

## Datenschutzprinzip
- Original-PDFs verlassen das Heimnetz nicht.
- OCR-Texte werden lokal verarbeitet.
- Cloud-APIs sind für die Dokumentenextraktion nicht vorgesehen.
- Falls später externe Dienste genutzt werden, dürfen nur abstrahierte Obsidian-Notizen ohne Vertragsnummern, sensible Personendaten oder Originaldokumente weitergegeben werden.

## Phasen

### Phase 1: Basisaufbau
- paperless-ngx installieren und konfigurieren
- Klassifizierungsschema für Dokumente definieren
- Migration der Dokumente ab 2020 mit Klassifizierung
- Bulk-Import des Bestands vor 2020 ohne vollständige Klassifizierung
- Obsidian Vault-Struktur anlegen
- Erste Templates und Dataview-Register einrichten

### Phase 2: Lokale LLM-Verarbeitung
- Mac Mini M4 Pro als lokaler Inference-Node einrichten
- Ollama/Qwen2.5 32B als lokale Pipeline installieren
- Extraktion neuer und alter Dokumente in Obsidian-Notes automatisieren
- Webhook- oder Batch-Prozesse für neue Dokumente implementieren

## Umsetzungsreihenfolge

```text
Phase 1 - Jetzt, ohne Mac Mini
  - paperless-ngx Docker aufsetzen
  - Tag-Schema und Korrespondenten anlegen
  - Migrationsskript für 808 Dokumente ab 2020 ausführen
  - Bulk-Import von rund 650 Dokumenten vor 2020 unklassifiziert durchführen
  - Obsidian Vault-Struktur anlegen
  - Vertragsregister manuell befüllen, zunächst etwa 15 bis 20 Verträge

Phase 2 - Nach Mac Mini Anschaffung
  - Ollama und Qwen2.5 32B einrichten
  - Extraktion Altbestand zu Obsidian-Notizen umsetzen
  - Extraktion Neubestand zu Obsidian-Notizen verfeinern
  - Webhook für neue Dokumente einrichten
```

## Nicht Bestandteil dieser Strategie
- Die vollständige Backup-Strategie wird in `40_BackupStrategy/` geführt.
- Homelab-Infrastruktur, NAS, Netzwerk und Dienste werden in separaten Infrastruktur-Dokumenten beschrieben.
- Obsidian Sync und mobiler Zugriff bleiben offen.

## Offene Punkte
- **Offen:** Backup-Strategie für paperless-ngx – wird im Homelab-Konzept behandelt.
- **Offen:** Mac Mini M4 Pro Netzwerkeinbindung – folgt nach Anschaffung.
- **Offen:** Obsidian Sync / mobiler Zugriff.
