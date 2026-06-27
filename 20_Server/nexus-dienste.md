# Nexus Dienstekatalog

Dieses Dokument beschreibt die aktuell bekannten Dienste auf Nexus (`nexusNG`). Es ist die operative Grundlage fuer Backup, Restore, Hardening und spaetere Migration in ein N+1-Zielbild.

## Statusklassen

| Status | Bedeutung |
|---|---|
| Produktiv | Aktiv genutzt und betriebskritisch |
| Infrastruktur | Basisdienst fuer andere Dienste |
| Test | Demo oder Entwicklung, nicht produktiv |
| Platzhalter | Stack definiert, aber nicht gestartet oder nicht vollstaendig konfiguriert |
| Altlast | Laeuft oder ist vorhanden, aber ohne klares Zielbild |

## Laufende Dienste

| Dienst | Status | Betriebsart | Zweck | Zugriff | Datenhaltung | Backup |
|---|---|---|---|---|---|---|
| Traefik | Infrastruktur | Docker | Reverse Proxy und TLS-Terminierung | Ports 80/443 auf allen Interfaces | Bind-Mounts fuer Konfiguration und Zertifikate | Ja |
| Authentik Server | Produktiv | Docker | Identity Provider / SSO | Intern via Traefik | PostgreSQL-Volume, ggf. Medien | Ja |
| Authentik Worker | Produktiv | Docker | Hintergrundaufgaben fuer Authentik | Kein direkter Zugriff | Gemeinsame Authentik-Daten | Ja |
| Authentik PostgreSQL | Infrastruktur | Docker | Datenbank fuer Authentik | Intern, Port 5432 | Docker Volume `authentik_database` | Ja |
| Unbound | Infrastruktur | Docker | Rekursiver DNS-Resolver | Port 53 TCP/UDP auf allen Interfaces | Bind-Mounts fuer Konfiguration | Ja, niedrige Prioritaet |
| socket-proxy | Infrastruktur | Docker | Begrenzter Docker-Socket-Zugriff fuer Traefik | Intern, Port 2375 | Keine persistenten Nutzdaten | Nein |
| nginx-demo | Test | Docker | Demo- oder Test-Endpunkt | Intern via Traefik | Keine persistenten Nutzdaten | Nein |
| WireGuard `wg0` | Altlast | System / Kernel-Interface | VPN-Relikt, nicht Fleet-Zugehoerigkeit | Route `10.10.30.0/24` | `/etc/wireguard/` | Ja, solange aktiv |
| SSH | Infrastruktur | Systemd | Fernzugang | Port 22 auf allen Interfaces | Keine Nutzdaten | Nein |

## Kritische Dienste

### Traefik

- Zweck: Einziger HTTP-/HTTPS-Eintrittspunkt fuer Webdienste auf Nexus.
- Betriebsart: Docker Container `traefik:v3`, zusammen mit `socket-proxy`.
- Compose-Pfad: `/home/decebu/docker-stacks/stacks/traefik/docker-compose.yml`.
- Ports: 80 und 443 auf allen Interfaces.
- Persistente Daten: statische Traefik-Konfiguration, dynamische Konfiguration, Zertifikatsdatei `acme.json`.
- Secrets: DNS-Provider-Token fuer Zertifikatsausstellung, nicht im Git dokumentieren.
- Backup: Konfiguration und `acme.json` sichern.
- Restore: Stack-Konfiguration wiederherstellen, Zertifikate einspielen oder kontrolliert neu ausstellen.

> **Offen:** Zertifikatspfad und konkrete Traefik-Bind-Mounts aus Compose-Datei dokumentieren.

> **Offen:** Monitoring fuer Zertifikatsablauf und Traefik-Erreichbarkeit festlegen.

### Authentik

- Zweck: Identity Provider fuer Homelab-Dienste, SSO und zentrale Benutzerverwaltung.
- Betriebsart: Docker Container fuer Server, Worker und PostgreSQL.
- Compose-Pfad: `/home/decebu/docker-stacks/stacks/authentik/docker-compose.yml`.
- Zugriff: ausschliesslich via Traefik, kein direkter externer Container-Port.
- Persistente Daten: Docker Volume `authentik_database`, ggf. Medien- oder Upload-Pfade.
- Secrets: Datenbankpasswort und Authentik Secret Key, nicht im Git dokumentieren.
- Backup: PostgreSQL-Dump plus relevante Medien-/Konfigurationspfade.
- Restore: Datenbankdump einspielen, Stack starten, Login und OIDC/OAuth-Flows pruefen.

> **Offen:** Automatischen PostgreSQL-Dump fuer Authentik festlegen.

> **Offen:** Medienpfade und eventuelle Uploads pruefen.

### Unbound

- Zweck: Lokaler rekursiver DNS-Resolver fuer Nexus und das Server-/Infra-Netz.
- Betriebsart: Docker Container `mvance/unbound-rpi`.
- Compose-Pfad: `/home/decebu/docker-stacks/stacks/unbound/docker-compose.yml`.
- Ports: 53 TCP/UDP auf allen Interfaces.
- Persistente Daten: Konfigurationsdateien per Bind-Mount.
- Secrets: Keine bekannt.
- Backup: Konfiguration ueber `docker-stacks` sichern.
- Restore: Konfiguration wiederherstellen und Container starten.

> **Offen:** Lokale DNS-Overrides und interne Zonen dokumentieren.

> **Offen:** Image-Pinning pruefen, da `latest` nicht reproduzierbar ist.

### WireGuard `wg0`

- Zweck: Aktiver oder ehemaliger VPN-Tunnel mit Adresse `10.10.30.21/32`.
- Einordnung: Altlast beziehungsweise Relikt. Nexus ist nicht Teil der Fleet.
- Betriebsart: System / Kernel-Interface, vermutlich `wg-quick`.
- Konfiguration: `/etc/wireguard/wg0.conf`.
- Persistente Daten: Private Keys, Peer-Konfiguration und ggf. Preshared Keys.
- Backup: Solange `wg0` aktiv bleibt, muss `/etc/wireguard/` gesichert werden.
- Restore: Entweder Konfiguration wiederherstellen oder neue Keys erzeugen und Peers aktualisieren.

> **Offen:** Klaeren, ob `wg0` entfernt, deaktiviert oder dokumentiert stillgelegt wird.

> **Offen:** Autostart von `wg0` pruefen.

## Nicht laufende oder Platzhalter-Stacks

| Stack | Pfad | Status | Einordnung |
|---|---|---|---|
| adguard-home | `/home/decebu/docker-stacks/stacks/adguard-home/` | Platzhalter | Alternative DNS-Loesung, aktuell nicht aktiv |
| authellia | `/home/decebu/docker-stacks/stacks/authellia/` | Platzhalter | Alternative IAM-Loesung, aktuell nicht aktiv |
| caddy | `/home/decebu/docker-stacks/stacks/caddy/` | Platzhalter | Alternative zu Traefik |
| fleet-manager | `/home/decebu/docker-stacks/stacks/fleet-manager/` | Platzhalter | Nicht fuer Nexus-Backup verwenden |
| influxdb | `/home/decebu/docker-stacks/stacks/influxdb/` | Platzhalter | Zeitreihendatenbank fuer spaeteres Monitoring |
| victoriametrics | `/home/decebu/docker-stacks/stacks/victoriametrics/` | Platzhalter | Alternative fuer Monitoring-Metriken |
| firezone | `/home/decebu/docker-stacks/stacks/firezone/` | Platzhalter | VPN-Management, Status unklar |
| fleet-client-ha | `/home/decebu/docker-stacks/stacks/fleet-client-ha/` | Platzhalter | Nicht als aktueller Nexus-Dienst behandeln |
| vps-monitoring | `/home/decebu/docker-stacks/stacks/vps-monitoring/` | Platzhalter | Monitoring-Kontext, Status unklar |
| portainer | `/home/decebu/docker-stacks/portainer/` | Platzhalter | Portainer ist nicht laufend |
| watchtower | `/home/decebu/docker-stacks/watchtower/` | Platzhalter | Automatische Updates nicht aktiv |
| knx | `/home/decebu/docker-stacks/knx/` | Platzhalter | Image nicht gesetzt |
| monitoring | `/home/decebu/docker-stacks/monitoring/` | Platzhalter | Image nicht gesetzt |
| mqtt | `/home/decebu/docker-stacks/mqtt/` | Platzhalter | Image nicht gesetzt |

## Backup-Matrix

| Dienst | Muss gesichert werden | Prioritaet | Backup-Art | Restore-Test |
|---|---|---|---|---|
| Authentik PostgreSQL | Ja | Hoch | `pg_dump` plus Volume-/Konfigurationssicherung | Dump erstellt, Restore offen |
| Authentik Medien/Konfiguration | Ja | Hoch | Datei-Backup | Pflicht |
| Traefik `acme.json` | Ja | Mittel | Datei-Backup | Pflicht |
| Traefik-Konfiguration | Ja | Mittel | Git/Datei-Backup | Pflicht |
| `/home/decebu/docker-stacks/` | Ja | Hoch | Restic-Datei-Backup | Erfolgreich getestet |
| `/etc/wireguard/` | Ja, solange aktiv | Hoch | Verschluesseltes Datei-Backup | Pflicht, falls weiter genutzt |
| Unbound-Konfiguration | Ja | Mittel | Git/Datei-Backup | Einfacher Restore-Test |
| nginx-demo | Nein | Keine | Keine | Nicht erforderlich |
| socket-proxy | Nein | Keine | Aus Compose reproduzierbar | Nicht erforderlich |

## Aktueller Backup-Stand

Stand: 2026-06-27

- Nexus-Backup per Restic auf QNAP ist eingerichtet.
- `nexus-backup.timer` laeuft taeglich gegen 02:30 Uhr.
- Authentik PostgreSQL-Dump wird vor dem Backup erzeugt.
- Manifest wird vor dem Backup erzeugt.
- Restore-Test fuer `docker-stacks` und Manifest war erfolgreich.
- Der Restore-Test liegt temporaer unter `/tmp/nexus-restore-test`.

## Offene Punkte

> **Offen:** Authentik-Dump in eine Testdatenbank restoren.

> **Offen:** Compose-Bind-Mounts der kritischen Dienste exakt aus den Compose-Dateien dokumentieren.

> **Offen:** Traefik-Zertifikatspfad und `acme.json` gegen den Backupumfang pruefen.

> **Offen:** `nginx-demo` klaeren und bei fehlendem Zweck entfernen.

> **Offen:** Automatische Container-Updates klaeren: Watchtower gewuenscht oder bewusst deaktiviert?

> **Offen:** Sicherheitsupdates klaeren, da automatische Update-Timer auffaellig waren.
