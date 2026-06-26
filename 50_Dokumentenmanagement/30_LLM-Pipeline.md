# Lokale LLM-Pipeline

## Ziel
Eine lokale Pipeline zur Dokumentenanalyse soll Dokumente aus `paperless-ngx` automatisch auswerten und strukturierte Obsidian-Notizen erzeugen.

## Hardware
Geplantes Zielgerät: **Mac Mini M4 Pro mit 48GB Unified Memory**.

### Gründe
- Qwen2.5 32B kann vollständig im Unified Memory betrieben werden.
- Erwartete lokale Inferenzleistung: etwa 18 Token pro Sekunde.
- Geringe Leistungsaufnahme im Leerlauf, Zielgröße etwa 30 W.
- Leise, effizient und gut in ein Homelab integrierbar
- Keine Cloud-Datenweitergabe erforderlich

## Software
- Ollama als lokale Inferenzplattform
- Modell: Qwen2.5 32B (oder alternativ ein lokal verfügbarer LLM)
- Die Ollama-API soll lokal für Homelab-Dienste erreichbar sein, ohne Dokumentdaten an externe Dienste zu senden.

## Pipeline
### Stufe 1: Extraktion
- Bei neuen Dokumenten aus `paperless-ngx` wird OCR-Text ausgelesen.
- Ollama erzeugt eine strukturierte Obsidian-Note mit YAML-Frontmatter und einer kurzen Zusammenfassung.
- Die Verarbeitung läuft perspektivisch als Webhook bei neuen Dokumenten.

### Stufe 2: Reasoning
- Mehrere Obsidian-Notizen werden bei Bedarf zusammengeführt, um Fragen zu beantworten und Kontext aus mehreren Dokumenten zu liefern.
- Beispiele: Versicherungslücken, Fristenübersichten, Jahresvergleiche und Immobilienhistorie.

## Altbestand
Für die Dokumente vor 2020 ist ein einmaliger Batch-Lauf vorgesehen:
- OCR-Text aus `paperless-ngx` lesen.
- Minimale Obsidian-Notiz mit Datum, Korrespondent, Kurzfassung und paperless-Link erzeugen.
- Keine vollständige manuelle Nachklassifizierung erzwingen.

## Datenschutz
- Keine Original-PDFs verlassen das Heimnetz.
- OCR-Texte werden lokal verarbeitet.
- Externe Dienste erhalten keine Originaldokumente, keine Vertragsnummern und keine personenbezogenen Details, sofern dies nicht ausdrücklich freigegeben wird.

## Offene Punkte
- **Offen:** Netzwerkeinbindung des Mac Mini M4 Pro.
- **Offen:** Detaillierte Prozessbeschreibung für Webhooks oder Batch-Jobs.
