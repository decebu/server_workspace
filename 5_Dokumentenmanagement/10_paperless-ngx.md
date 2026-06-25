# paperless-ngx Strategie

## Ziel
`paperless-ngx` dient als zentrales Dokumentenarchiv für die OCR-gestützte Volltextsuche und die langfristige Archivierung von Dokumenten.

Die Migration ersetzt ecoDMS schrittweise. Ab dem Schnittdatum `2020-01-01` wird der Bestand vollständig klassifiziert. Ältere Dokumente werden zunächst als suchbarer Altbestand übernommen.

## Klassifizierungsschema

### Korrespondenten
Die Korrespondenten werden aus den bestehenden ecoDMS-Ordnern abgeleitet.

Beispiele:

| Bereich | Korrespondenten |
|---|---|
| Versicherungen | Signal Iduna, Allianz, DKV, HDI, Schomacker, AXA, Württembergische, Standard Life, Yachtpool |
| Bank/Finanzen | DKB, Comdirect, Volksbank, Fidor, Postbank |
| Arbeit | Bosch |
| Behörden | Finanzamt, Deutsche Rentenversicherung |
| Energie | NetzeBW, Transnet BW, Immergrün, EnBW |
| Sonstige | Nach Bedarf ergänzen |

### Dokumenttypen
Die Dokumenttypen orientieren sich an der bestehenden ecoDMS-Pflege:
- Rechnung
- Entgeltabrechnung
- Vertrag
- Police
- Bescheid
- Bescheinigung
- Kontoauszug
- Kündigung
- Protokoll
- Dokumentation
- Nachweis
- Angebot
- Information

### Tags
Tags werden thematisch vergeben, mehrere Tags pro Dokument sind möglich.

Beispiele:

| Tag | Herkunft in ecoDMS | Dokumente ab 2020 |
|---|---|---|
| Gesundheit | Thomas, Tom, Ben, Claudi, Krankheiten und Ärzte | 163 |
| Versicherungen | Alle Versicherungsordner | 127 |
| Arbeit | Bosch | 107 |
| Immobilie-Perouse | Haus Perouse und Wohnung | 66 |
| Energie | Strom, Gas und Wasser | 65 |
| Finanzen | Alle Bank-Unterordner | 65 |
| KFZ | Auto | 45 |
| Immobilie-Callitectum | Projekt Callitectum | 19 |
| Steuern | Steuer | 12 |
| GbR-Stromversorgung | Stromversorgung Walter GbR | 10 |
| Telekommunikation | Telefon, Internet, TV und Rundfunk | 4 |
| Musikverein | Musikverein | 6 |
| Segeln | Hobby und Freizeit, Segeln | 2 |

### Custom Field
`Person` mit möglichen Werten `Thomas`, `Claudia`, `Tom`, `Ben`.

Das Feld wird nur für personenspezifische Dokumente gesetzt, zum Beispiel Renteninformationen, Gesundheitsunterlagen oder Dokumente zu Kindern. Es ersetzt keine thematischen Tags.

## Storage Path
Automatisiertes Dateisystemformat:
```
{created_year}/{correspondent}/{title}
```

## Migration

### Bestand ab 2020
Ziel ist eine vollständige Migration mit Klassifizierung für rund 808 Dokumente.

Geplantes Mapping:
- `hauptordner` und `ordner` werden auf Tags abgebildet.
- `dokumentenart` wird 1:1 als Dokumenttyp übernommen.
- Bei Versicherungen und Banken wird `ordner` als Korrespondent verwendet, sofern eindeutig.
- `bemerkung` wird als Titel übernommen.
- `datum` wird als Dokumentdatum übernommen.
- Es werden nur Dokumente mit `datum >= 2020-01-01` migriert.

Das Migrationsskript basiert konzeptionell auf `MichaelKirgus/ecodmstopaperlessngx-ng`. Vor Ausführung ist zu prüfen, ob Exportformat und API-Version zur lokalen Installation passen.

### Bestand vor 2020
Der Altbestand umfasst rund 650 Dokumente und wird als Bulk-Import übernommen:
- PDFs werden importiert.
- OCR erzeugt Volltextsuche.
- Tags, Korrespondenten und Dokumenttypen werden nicht flächendeckend nachgepflegt.
- Kontext entsteht später über Obsidian-Notizen und lokale Extraktion.

## Consumption Templates
Nach der Migration werden Regeln für neue Dokumente eingerichtet.

Beispiele:
- Titel enthält `Entgeltabrechnung`: Korrespondent `Bosch`, Typ `Entgeltabrechnung`, Tag `Arbeit`.
- Absender enthält `Signal Iduna`: Korrespondent `Signal Iduna`, Tag `Versicherungen`.
- Absender enthält `Finanzamt`: Korrespondent `Finanzamt`, Tag `Steuern`, Typ `Bescheid`.

Ziel ist, neue Eingangsdokumente möglichst automatisch mit Korrespondent, Dokumenttyp und Tags vorzubelegen. Manuelle Korrekturen bleiben möglich und sind insbesondere in der Anfangsphase eingeplant.

## Umgebung
Wichtige Umgebungsvariablen:
```yaml
PAPERLESS_FILENAME_FORMAT: "{created_year}/{correspondent}/{title}"
PAPERLESS_CONSUMER_RECURSIVE: true
PAPERLESS_OCR_LANGUAGE: deu+eng
PAPERLESS_TIME_ZONE: Europe/Berlin
```

## Offene Punkte
- **Offen:** Detailliertes Backup-Konzept für paperless-ngx in `40_BackupStrategy/` verlinken.
- **Offen:** Prüfung, welche Daten in paperless-ngx als Meta-Felder noch zusätzlich gespeichert werden sollten.
