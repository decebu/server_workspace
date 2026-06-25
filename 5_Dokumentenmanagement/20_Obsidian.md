# Obsidian Vault Strategie

## Ziel
Obsidian wird als Wissenslayer genutzt, um Dokumente aus `paperless-ngx` kontextuell zu verknüpfen und Aufgaben, Fristen und persönliche Notizen zu verwalten.

`paperless-ngx` bleibt das Archiv. Obsidian enthält keine vollständigen Dokumentkopien, sondern Kontext, Register, Fristen, Zusammenfassungen und Deep Links auf die Dokumente.

## Struktur
Empfohlene Verzeichnisstruktur:

```
vault/
├── 00 Dashboard.md
├── 10 Verträge/
│   ├── _Register.md
│   ├── Signal Iduna BU.md
│   └── Allianz KFZ.md
├── 20 Immobilien/
│   ├── Perouse/
│   │   ├── Übersicht.md
│   │   ├── Renovierung 2024.md
│   │   └── Handwerker.md
│   └── Callitectum/
│       ├── WEG Protokolle.md
│       └── Jahresabrechnungen.md
├── 30 Familie/
│   ├── Thomas.md
│   ├── Claudia.md
│   ├── Tom.md
│   └── Ben.md
├── 40 Steuern/
│   ├── 2024.md
│   └── _Checkliste Template.md
├── 50 Finanzen/
│   ├── Überblick.md
│   ├── GbR Stromversorgung.md
│   └── Depot-Snapshot 2024.md
└── _Templates/
    ├── Vertrag.md
    ├── Steuer-Jahr.md
    └── Immobilie-Projekt.md
```

### Beispiele
- `00 Dashboard.md` – Startseite mit Dataview-Übersichten und offenen Fristen
- `10 Verträge/` – Vertragsregister, Vertragsnotizen
- `20 Immobilien/` – Projekte Perouse und Callitectum
- `30 Familie/` – Personenbezogene Dokumente und Ereignisse
- `40 Steuern/` – Steuerjahr-Notizen und Checklisten
- `50 Finanzen/` – Finanzüberblick und Konten

## Templates und Metadata
Beispiel-Metadaten:
```markdown
---
korrespondent: Signal Iduna
typ: BU-Versicherung
laufzeit_bis: 2031-12-31
kuendigungsfrist_monate: 3
monatlich: 87.50
person: Thomas
paperless: <PAPERLESS_ENDPOINT>/documents/?correspondent=Signal+Iduna
tags: [Versicherungen, aktiv]
---
Notizen und persönlicher Kontext hier.
```

## Workflow
- Neue Verträge und relevante Dokumente erhalten eine Obsidian-Notiz mit Kontext und Fristen.
- Registerseiten nutzen Dataview für Übersichten zu laufenden Verträgen, Steuerjahren und Immobilienprojekten.
- Deep Links verweisen auf `paperless-ngx`; Original-PDFs werden nicht im Vault dupliziert.
- Für den Altbestand vor 2020 können minimalistische Notizen aus OCR-Texten erzeugt werden.
- Das Vertragsregister wird zu Beginn manuell mit etwa 15 bis 20 relevanten Verträgen befüllt.
- Bestehende Dokumente bleiben in `paperless-ngx`; Obsidian bildet Bedeutung, Aufgaben und Zusammenhänge ab.

## Plugins
Empfohlene Obsidian-Plugins:
- Dataview
- Templater
- Tasks
- Calendar
- Obsidian URI

## Offene Punkte
- **Offen:** Obsidian Sync / mobiler Zugriff.
- **Offen:** Konkrete Regeln zur Pflege der manuellen Notizen.
