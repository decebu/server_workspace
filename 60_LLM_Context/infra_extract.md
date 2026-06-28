# Homelab- und VPN-Infrastruktur — anonymisierter Kontext-Extract

> **Zweck.** Dieses Dokument ist ein bewusst anonymisierter Auszug meiner Homelab-, VPN- und
> Sicherheits-Architektur. Es dient als Kontext für ein externes LLM, mit dem ich die Umsetzung
> **neuer Dienste oder Server** unter Einhaltung meiner Sicherheitsprinzipien durchdenke
> (z. B. „Wie mache ich Dienst X für Modell/Client Y erreichbar, ohne meine Isolation zu brechen?").
>
> **Es enthält keine echten IPs, Domains, URLs, Schlüssel, Tokens, Passwörter, Hostnamen,
> Benutzernamen oder Standort-/Personennamen.** Alle solchen Werte sind durch Platzhalter ersetzt.
> Technische Produkt-/Herstellernamen (WireGuard, Docker, Traefik, Authentik, Kestra, Ansible,
> restic, Unbound, Firezone, paperless-ngx, Ollama, Ubiquiti/UDM, FRITZ!Box, QNAP, IONOS,
> Cloudflare …) bleiben erhalten, weil sie technischer Kontext und nicht sensibel sind.
>
> **Hinweis fürs LLM:** Antworte auf Platzhalter-Ebene. Wenn du z. B. eine Firewall-Regel oder
> Route vorschlägst, benenne Quelle/Ziel über die Platzhalter (`NET-IOT`, `HOST-HOMELAB-SRV`,
> `OVERLAY-MOBILE` …); ich übersetze das lokal in echte Werte zurück.

---

## 0. Anonymisierungs-Konventionen (Platzhalter-Typen)

Alle folgenden Platzhalter stehen für reale, hier nicht genannte Werte. Gleiche Platzhalter =
gleiches reales Objekt (konsistent über das ganze Dokument).

| Platzhalter-Typ | Bedeutung |
|---|---|
| `NET-<ROLLE>` | Ein lokales Subnetz/VLAN nach Funktion (z. B. `NET-IOT`). Reale Oktette/CIDR-Werte sind entfernt; CIDR-**Größe** (`/24`, `/25`) ist sachlich, aber generisch. |
| `OVERLAY-<ROLLE>` | Ein WireGuard-Overlay-Segment (Tunnel-Adressraum), z. B. `OVERLAY-MOBILE`. |
| `HOST-<ROLLE>` | Ein konkreter Host nach Rolle, z. B. `HOST-HOMELAB-SRV`. |
| `NAS`, `HA-PROD`, `HA-PROD-NET` | Speicher-Appliance bzw. die Home-Assistant-Produktivinstanz und ihr Netz. |
| `<ADMIN_USER>`, `<AUTOMATION_USER>` | Linux-Benutzer (interaktiver Admin bzw. nicht-interaktiver Ansible-Automationsuser). |
| `<NTFY_BOT_USER>`, `<RSYNC_USER>` | Service-Accounts für Push-Benachrichtigung bzw. rsync. |
| `<PORT-WG-A/B/FZ/C>`, `<PORT-SSH-ALT>` | Nicht-Standard-(Custom-)Ports. Well-known Ports (22, 53, 80, 443, 5432, 873, 8080, 8123, 2375, 9100) sind real genannt. |
| `<PUBLIC_ENDPOINT>`, `<DDNS_HOSTNAME>` | Öffentlich erreichbarer Endpunkt / dynamischer DNS-Hostname. |
| `<NTFY_PUBLIC_ENDPOINT>`, `<KESTRA_PUBLIC_ENDPOINT>` | Öffentliche Service-URLs. |
| `<SECRET>`, `<API_TOKEN>`, `<PASSWORD>` | Schlüssel/PSK / Token / Passwort. |

---

## 1. Überblick (eine Absatz-Zusammenfassung)

Ein **zentraler Cloud-VPS** (`HOST-EDGE-VPS`, Ubuntu LTS, bei IONOS) ist der **WireGuard-Hub**
einer Sterntopologie. Er verbindet mehrere **Heimnetze** (ein Hauptstandort mit Enterprise-Gateway
und vielen VLANs, plus mehrere Remote-Standorte hinter Consumer-Routern) sowie **mobile Clients**
(über Firezone) zu einem Overlay. Der **Homelab-Server** (`HOST-HOMELAB-SRV`, Raspberry Pi 5,
Docker Compose) betreibt die Kern-Webdienste hinter einem Reverse Proxy (Traefik) mit zentralem
IAM (Authentik) und lokalem DNS (Unbound). Ein separater **Fleet-Manager** (`HOST-FLEET-MGR`)
automatisiert Updates und Backups einer Geräte-Flotte über ein eigenes Management-Overlay
(Kestra + Ansible). Backups laufen mit **restic** auf eine **NAS** (QNAP). Durchgängige Prinzipien:
**strikte Netzsegmentierung + Allow-List-Firewalls**, **Secrets nie im Klartext/Git**,
**Daten-Souveränität** (sensible Originaldaten verlassen das Heimnetz nicht), **Reproduzierbarkeit**
(Dienste aus Doku + Compose + Backup wiederherstellbar, Restore-Test als Pflicht-Gate).

---

## 2. Netz-Topologie & Segmentierung

### 2.1 Standorte (Sites)

| Site | Gateway-Klasse | Rolle | Vertrauen |
|---|---|---|---|
| `SITE-HOME-MAIN` (HN1) | Enterprise-Gateway/Firewall (Ubiquiti UDM Pro) | Hauptstandort, voll segmentiert, Hub-seitig | hoch (Kern) |
| `SITE-HOME-B` (HN2) | Consumer-Router (FRITZ!Box) + RPi-WireGuard-Gateway | Sekundärstandort; beherbergt `HOST-FLEET-MGR` | reduziert (als „unsicher" behandelt) |
| `SITE-REMOTE-A/B` | Consumer-Router (FRITZ!Box) | entfernte Standorte Dritter, nur Anbindung an Hauptstandort | niedrig |
| `SITE-SETUP` | externe FRITZ!Box (VPS ist hier WG-**Client**) | reines Einrichtungs-/Durchleitungsnetz, **einseitig** | niedrig, isoliert |

### 2.2 VLANs am Hauptstandort (`SITE-HOME-MAIN`)

| VLAN-Rolle | Funktion | CIDR-Größe |
|---|---|---|
| `NET-MGMT` | nur Netzwerkgeräte (Gateway, Switches, APs) — **keine** Server/Clients | /24 |
| `NET-PROD` | produktive Familien-/Nutzerclients | /24 |
| `NET-GUEST` | Gäste | /24 |
| `NET-IOT` | IoT-Geräte (**hier liegt `HA-PROD`**) | /24 |
| `NET-KNX` | KNX-Gebäudeautomation | /24 |
| `NET-WORK` | Arbeits-/Admin-Clients (Quelle für Admin-Zugriffe) | /25 |
| `NET-SERVER` | Server-/Infra-/DNS-VLAN (Homelab-Server, lokaler DNS, geplanter Cluster) | /25 |

Designregeln:
- **Management ist separat** und enthält keine Server/Workloads.
- **Server liegen in `NET-SERVER`**, nicht im Client-Netz.
- **Admin-Zugriff** (SSH/HTTPS) auf Server erfolgt aus `NET-WORK` über definierte Firewall-Regeln.
- `HA-PROD` läuft **isoliert in `NET-IOT`** und ist nur über gezielte Firewall-Freigaben erreichbar.

---

## 3. VPN-Architektur

### 3.1 Topologie

- **Hub:** `HOST-EDGE-VPS` terminiert mehrere WireGuard-Interfaces (eines je angebundenem
  Standort) plus Firezone für mobile Clients. **Sterntopologie**, kein Vollmesh.
- **Spokes:**
  - Hauptstandort: das Enterprise-Gateway ist WG-**Client** zum Hub (`OVERLAY-HUB`).
  - Sekundär-/Remote-Standorte: je ein Gateway (RPi bzw. Consumer-Router) als WG-Peer.
  - Mobile Clients: über **Firezone** (eigener WG-Dienst auf dem VPS, eigener Adressraum
    `OVERLAY-MOBILE`).
  - Management: eigenes **`OVERLAY-FLEET`** für die verwaltete Flotte (s. Abschnitt 6).
- **Sonderfall `SITE-SETUP`:** Hier ist der VPS ausnahmsweise WG-**Client** zu einer externen
  FRITZ!Box. Wichtig: deren mitgelieferte Client-Config setzt `AllowedIPs = 0.0.0.0/0` (Full
  Tunnel) — das wird **nicht** übernommen, nur das Setup-Subnetz, damit nicht der gesamte
  VPS-Traffic durch die Gegenstelle läuft.

### 3.2 Overlay-Segmente

| Overlay | Zweck |
|---|---|
| `OVERLAY-HUB` | VPS ⇄ Hauptstandort-Gateway |
| `OVERLAY-REMOTE-B` | VPS ⇄ Sekundärstandort-Gateway (RPi) |
| `OVERLAY-MOBILE` | Firezone-Clients (Laptop, Tablet, Handy) |
| `OVERLAY-FLEET` | Management-Overlay: Fleet-Manager ⇄ Fleet-Clients |

### 3.3 Routing-/Tunnel-Prinzipien

- **AllowedIPs bewusst breit** (ganze Subnetze) in den `.conf`-Dateien, um sie selten ändern zu
  müssen; die **Feinsteuerung passiert in der Firewall** (Gateway + VPS), nicht über AllowedIPs.
- **Split-Tunnel:** an den Remote-Standorten läuft der Internetverkehr lokal über den eigenen
  Router; **nur interne Zielnetze** gehen durch den Tunnel.
- **Echtes Routing ohne NAT** zwischen Hauptstandort und Sekundärstandort (MASQUERADE dort
  entfernt → echte Quell-IPs sichtbar). **NAT bleibt** nur für Firezone- und Docker-Netze.
- WG-Peering mit **PresharedKey** zusätzlich zum Keypair; `PersistentKeepalive` bei
  Client-Peers hinter NAT/CGNAT.

---

## 4. Firewall- & Trust-Modell (Kernprinzipien)

Die Anbindung ist absichtlich **default-deny + Whitelist**. Zwei Durchsetzungsebenen:
das Enterprise-Gateway am Hauptstandort und nftables auf VPS und RPi-Gateways.

### 4.1 Trust-Matrix (wer darf wohin)

| Quelle | Erlaubtes Ziel | Sonst |
|---|---|---|
| Hauptstandort (`SITE-HOME-MAIN`) | alle anderen Netze | — |
| Remote-Standorte (`SITE-HOME-B`, `SITE-REMOTE-*`) | **nur** Hauptstandort, und dort nur definierte Hosts/Ports (Gateway-Admin, `NAS`, `HA-PROD`) | untereinander **kein** Zugriff |
| Mobile (`OVERLAY-MOBILE`) | Hauptstandort (Gateway, `NAS`, `HA-PROD`) | **kein** Zugriff auf Remote-Standorte |
| `SITE-SETUP` | — (einseitig: Hauptstandort darf hinein, Rückrichtung blockiert) | policy drop |

Typische freigegebene Ziel-Dienste in den Allow-Regeln: Gateway-Admin (22/443), `NAS`
(22/445/5000/5001), `HA-PROD` (8123). Drop-All-Regeln stehen **immer ganz unten**.

### 4.2 VPS-Firewall (nftables)

- `policy drop` auf input und forward; nur `established,related` + Loopback + explizite Allows.
- Eingehend nur die WireGuard-UDP-Ports (`<PORT-WG-A/B/FZ>`); **Public-SSH ist deaktiviert** —
  SSH nur im Tunnel.
- Forward streng per Interface→Ziel-Whitelist (z. B. „aus Sekundärstandort nur zu Gateway,
  `NAS`, `HA-PROD`").

### 4.3 RPi-Gateway-Firewall (nftables)

- `policy drop` auf **input, forward UND output** (Output-Whitelist: nur DNS, WG-Traffic zum Hub,
  HTTP/HTTPS).
- Input: SSH nur aus dem lokalen Subnetz bzw. der Management-Tunnel-IP; Monitoring (node_exporter
  9100) nur aus der Monitoring-Quelle.
- Forward: nur lokale Quellen zu den definierten Zielhosts am Hauptstandort.

### 4.4 Fallback-Regel

- Immer einen **Admin-Pfad mit vollen Rechten** (Mobile/Firezone) einplanen, um sich bei einer
  fehlerhaften Regel nicht auszusperren.

---

## 5. Hosts & Rollen

| Host (Platzhalter) | HW/Plattform | Rolle / Dienste | Netz | Status |
|---|---|---|---|---|
| `HOST-EDGE-VPS` | Cloud-VPS (Ubuntu LTS, IONOS) | WireGuard-Hub; Firezone (Docker, eigener WG-Port `<PORT-WG-FZ>`, PostgreSQL, Caddy-UI); ntfy-Server (Auth, deny-all); nftables policy-drop; Public-SSH zu | Internet-Edge + alle Overlays | produktiv |
| `HOST-HOMELAB-SRV` | Raspberry Pi 5, 8 GB, NVMe, Debian, Docker Compose | Traefik (Reverse Proxy/TLS); Authentik (Server+Worker+PostgreSQL, IAM/SSO); Unbound (lokaler DNS); socket-proxy; nginx-demo; **Altlast** wg0 | `NET-SERVER` | produktiver Einzelknoten |
| `HOST-PVE-NODE-1..3` | Raspberry Pi 5 (geplant) | künftiger Proxmox-Cluster (N+1-Zielbild) | `NET-SERVER` | geplant |
| `HOST-FLEET-MGR` | Raspberry Pi, Docker | Kestra + Ansible (über WG-Container-Namespace), rsync-Aggregat, ntfy-Push | `SITE-HOME-B` + `OVERLAY-FLEET` | produktiv (außerhalb der „Fleet" selbst) |
| `HOST-HA-CLIENT` | Raspberry Pi | Home-Assistant-Host, Fleet-Client (Backup/Update durch Manager) | `OVERLAY-FLEET` | produktiv |
| `HOST-FLEET-CLIENT-n` | Raspberry Pi / Debian | generischer Fleet-Client | `OVERLAY-FLEET` | produktiv |
| `NAS` | QNAP | Backup-Ziel (SMB/rsync, read-only-Bereitstellung), DSM-Dienste | `NET-PROD` | produktiv |
| `HA-PROD` | (auf HA-Host) | **Home-Assistant-Produktivinstanz, isoliert** | `NET-IOT` | produktiv, isoliert |
| `HOST-LLM-NODE` | Mac Mini M4 Pro, 48 GB (geplant) | lokale LLM-Inferenz (Ollama + Qwen2.5 32B) für Dokumentenanalyse, **kein** Cloud-Egress | `NET-SERVER` (offen) | geplant |

> Hinweis Konsolidierung: Die Quell-Doku enthält mehrere zeitlich versetzte Adressschemata, die
> sich teils überlappen. Dieses Dokument bildet **ein** kohärentes Rollenmodell ab. Bekannte
> Altlasten/Sonderfälle sind als solche markiert (z. B. `wg0` auf `HOST-HOMELAB-SRV` ist ein
> Relikt und **nicht** Teil der Fleet; doppelte Subnetze werden ggf. per NAT am VPS umgeschrieben).

---

## 6. Fleet-Management (Automatisierung)

- `HOST-FLEET-MGR` betreibt **Kestra** (Orchestrierung) + **Ansible** und ein **restic-Aggregat**.
- **Netzwerk-Kniff:** Kestra hat kein eigenes Netz-Interface (`network_mode: service:wireguard`);
  die von Kestra dynamisch gestarteten Ansible-Container hängen im Namespace des
  WireGuard-Containers (`networkMode: container:<wg>`). Dadurch erreichen die Playbooks die
  Fleet-Hosts über `OVERLAY-FLEET`, ohne dass Kestra selbst geroutet werden muss.
- **Flows (per Schedule + Flow-Trigger):**
  - täglich: restic-Backup **lokal auf jedem Client** → anschließend **Pull** der Repos auf den
    Manager (rsync) → zentrale **ntfy**-Statusmeldung.
  - wöchentlich: `apt`-Updates auf Fleet- und LAN-Geräten.
  - Diagnose-Flow (Ansible ping) und Key-Rotations-Flows.
- **Fleet-Client-Anforderungen:** nicht-interaktiver User `<AUTOMATION_USER>` mit
  SSH-Key-Login + passwortlosem sudo; Restic-Repo-Passwort als root-only Datei; Eintrag im
  (git-crypt-verschlüsselten) Inventory. Der hinterlegte Public Key ist mit
  `from="<MGMT-TUNNEL-IP>"` auf die Manager-VPN-IP **eingeschränkt** (Blast-Radius).
- **Manager-Self-Backup:** bewusst **anders** (host-lokaler systemd-Timer statt Kestra), weil der
  Kestra-Container die age-Identity/Repo-Passwörter nicht sehen soll.

---

## 7. DNS-Architektur

- **Lokal rekursiv:** Unbound (Container auf `HOST-HOMELAB-SRV`, bindet 53/tcp+udp) als interner
  Resolver für `NET-SERVER`. AdGuard ist als Alternative vorgesehen (aktuell Platzhalter).
- **Extern autoritativ:** Cloudflare verwaltet die öffentlichen Domains (DNS, Proxy/DDoS,
  DNS-Challenge für Zertifikate). Der Domain-Provider gibt die Nameserver an Cloudflare ab.
- **Bekannte Stolperstelle:** ein DNS-Resolver in einer Docker-Bridge ist je nach VLAN-/Routing-
  Sichtbarkeit nicht aus allen VLANs nutzbar; Lösung ist Betrieb des Resolvers direkt im Zielnetz
  (nicht hinter Docker-Bridge-Magie).

---

## 8. Backup-Strategie

Zwei getrennte Standards (bewusst nicht vermischt):

**LAN-Server-Standard (z. B. `HOST-HOMELAB-SRV`):**
- restic-Push vom Server auf `NAS` (SMB-Mount). Repo-Passwort host-lokal, root-only, **nicht im
  Git**. App-konsistente Vorstufen: PostgreSQL-Dump (Authentik) + Manifest **vor** dem Lauf.
- systemd-Timer (täglich), Retention 7 täglich / 4 wöchentlich / (6 monatlich als Startwert),
  **ntfy**-Meldung bei Erfolg/Fehler. **Restore-Test ist Pflicht.**

**Fleet-Standard:**
- restic läuft **lokal auf jedem Client** (Ansible-getrieben), Repos werden per rsync auf den
  Manager gezogen (Aggregat), die `NAS` zieht das Aggregat read-only per rsync-Daemon.

**Gates:** Ein neuer kritischer Dienst (z. B. paperless-ngx) darf erst produktiv mit echten Daten
laufen, wenn **dienst-spezifischer Backup-Umfang + erfolgreicher Restore-Test** dokumentiert sind.

---

## 9. Sicherheitsprinzipien (meine Randbedingungen)

Diese Prinzipien sind der Maßstab, an dem jeder neue Dienst geprüft wird.

### 9.1 Secrets

1. **Möglichst nicht in Git** — auch git-crypt schützt nur Remote, lokal liegt der Wert im Klartext.
2. **Kein Klartext at rest** — bevorzugt **age-verschlüsselt**, erst zur Laufzeit per Injektion in
   die Container-Env entschlüsselt (`${VAR:?fehlt}`-Guard lässt Start ohne Secret bewusst scheitern).
3. **Entscheidungsbaum „wo lebt ein Secret?":** Service-Secret → age + Runtime-Injektion;
   muss versioniert ins Repo → git-crypt (nur wenn nötig); host-lokales Daemon-Secret → root-only
   Datei `chmod 600`; Boot-kritisches Mount-Secret → gitignored Host-Datei.
4. **Blast-Radius begrenzen:** SSH-Keys mit `from="…"` an die erwartete Quelle binden; kleinster
   Token-Scope; eingeschränktes sudo statt `NOPASSWD: ALL` wo praktikabel.
5. **Rotation mit Überlappung:** neues Geheimnis **neben** dem alten gültig machen, verifizieren,
   dann altes entwerten — kein Aussperren (inkl. Selbst-Aussperr-Schutz in den Rotations-Flows).
6. **Root-of-Trust extern sichern** (age-Identity + Passphrase, root-only Repo-Passwörter) —
   liegen bewusst **nicht** im eigenen Backup.
7. **Ehrliche Grenze:** laufende Env-Secrets sind per `docker inspect`/`printenv` für jeden mit
   Docker-Zugriff lesbar — der Aufbau reduziert „in Git" und „Klartext at rest", nicht das.

### 9.2 Host-Hardening (Zero Trust, auch bei möglichem physischem Zugriff)

- SSH nur Key-Login, kein Root-Login, fail2ban/sshguard, möglichst nur über Tunnel.
- Minimale Pakete/Dienste; ungenutzte Daemons (`rpcbind`, Desktop-Dienste, `cloud-init` …)
  prüfen/abschalten; aktuelle Kernel-/Security-Updates; Zeitsynchronisation.
- nftables strikt Allow-List (policy drop), Kernel-/sysctl-Hardening (keine ICMP-Redirects, kein
  Source-Routing, SYN-Cookies, Logging verdächtiger Pakete).
- Physisch: Full-Disk-Encryption (LUKS), `/tmp` & `/var/tmp` mit `noexec,nosuid,nodev`,
  Bootloader-Absicherung; Remote-Sperre via Key-Revoke an der VPS-Firewall denkbar.
- Logging/Monitoring (node_exporter, journald-Forward) zur Nachvollziehbarkeit.

### 9.3 Daten-Souveränität

- **Sensible Originaldaten verlassen das Heimnetz nicht** (z. B. Original-PDFs, Vertragsnummern,
  personenbezogene Details).
- Lokale Verarbeitung bevorzugt (lokale LLM-Inferenz auf `HOST-LLM-NODE` statt Cloud-API).
- Externe Dienste/APIs erhalten höchstens **abstrahierte** Inhalte ohne sensible Identifikatoren —
  und nur nach bewusster Freigabe.

### 9.4 Reproduzierbarkeit & Exposition

- Dienste müssen aus **Doku + Compose + Backup** wiederherstellbar sein; Images möglichst pinnen
  (kein blindes `latest`).
- **Exposition-Default:** intern **nur über Reverse Proxy (Traefik)**, **keine** direkten
  Container-Ports nach außen; Remote-Zugriff über das bestehende VPN/Firezone statt neuer
  Port-Forwards.

---

## 10. Checkliste „neuer Dienst / neuer Server"

Vor der Umsetzung jeden Punkt beantworten:

- [ ] **Platzierung:** In welches Netz/auf welchen Host? (Sensibel → `NET-SERVER`/isoliert;
      IoT-nah → `NET-IOT`; nie unnötig ins Client-/Management-Netz.)
- [ ] **Datenklasse:** Berührt der Dienst sensible Daten? Falls ja → **lokal halten**, kein
      Cloud-Egress, externe Schnittstellen nur abstrahiert.
- [ ] **Exposition:** Intern via Traefik? Remote via VPN/Firezone? **Kein** neuer Port-Forward
      ohne zwingenden Grund; keine direkten Container-Ports.
- [ ] **Firewall/Trust:** Aus welcher Trust-Zone kommt der Zugriff? Passende Allow-Regel als
      Whitelist (Quelle→Ziel→Port), Drop-All bleibt unten.
- [ ] **Secrets:** Nach Entscheidungsbaum (Default: age + Runtime-Injektion, nicht in Git),
      kleinster Scope, Blast-Radius via `from=`/Scope begrenzt, Rotationsweg bekannt.
- [ ] **Backup-Gate:** Backup-Umfang + **Restore-Test** dokumentiert, **bevor** echte Daten rein.
- [ ] **Reproduzierbarkeit:** Compose/Config versioniert, Image gepinnt, im Backup enthalten
      (oder bewusst ausgenommen).
- [ ] **Hardening:** Host erfüllt 9.2; keine neuen offenen Daemons/Ports.
- [ ] **Fallback:** Admin-Pfad bleibt erreichbar (kein Aussperren).

---

## 11. Aufgaben-Vorlage für das externe LLM

Kopiere dieses Dokument **plus** den ausgefüllten Block unten ins Pro-LLM.

```text
KONTEXT: siehe obenstehenden Infrastruktur-Extract (Abschnitte 1–10).

NEUER DIENST / NEUE FRAGE
- Dienst/Vorhaben:        <kurz, was ich aufsetzen will>
- Wo soll er laufen:      <Host/Netz, oder „offen — bitte empfehlen">
- Wer/was muss zugreifen: <Quelle(n): NET-*, OVERLAY-*, externe Modelle/Clients, …>
- Gewünschte Exposition:  <intern via Traefik / Remote via VPN / öffentlich — und warum>
- Sensible Daten?:        <ja/nein, welche Klasse>
- Vorgaben aus Doku:      <relevante Herstellerempfehlung, die ich NICHT blind umsetzen will>

AUFGABE
1. Prüfe 2–3 Umsetzungsvarianten gegen meine Sicherheitsprinzipien (Abschnitt 9) und die
   Checkliste (Abschnitt 10).
2. Nenne je Variante: Netz-Platzierung, Firewall-/Routing-Änderungen (auf Platzhalter-Ebene),
   Exposition, Secrets-Handhabung, Backup-/Restore-Implikation, Blast-Radius.
3. Gib eine Empfehlung mit Begründung und eine knappe Schritt-für-Schritt-Skizze.
4. Markiere offene Annahmen und was ich gegenprüfen muss.
```

### 11.1 Ausgefülltes Beispiel (AI-MCP-Server für Home Assistant)

```text
NEUER DIENST / NEUE FRAGE
- Dienst/Vorhaben:        MCP-Server als Add-on auf HA-PROD, damit On-Demand-LLMs auf Home
                          Assistant zugreifen können.
- Wo soll er laufen:      Auf HA-PROD (in NET-IOT, vollständig isoliert).
- Wer/was muss zugreifen: Externe/On-Demand-Modelle (außerhalb des Heimnetzes) als MCP-Client.
- Gewünschte Exposition:  So wenig wie möglich. Doku schlägt vor, MCP-Traffic über einen
                          bestehenden Reverse Proxy / Webhook-Proxy (Nabu Casa o. ä.) nach außen
                          zu reichen — das will ich NICHT blind tun.
- Sensible Daten?:        Ja — HA steuert/liest das ganze Smart Home; Zugriff = hoher Blast-Radius.
- Vorgaben aus Doku:      „Webhook Proxy Add-on routet MCP über bestehenden Reverse Proxy, kein
                          separater Tunnel/Port-Forward nötig; AI-Client mit Webhook-URL konfigurieren."

AUFGABE
1. Bewerte, ob/wie der MCP-Server erreichbar gemacht werden kann, OHNE die NET-IOT-Isolation von
   HA-PROD aufzuweichen und ohne neuen öffentlichen Port-Forward.
2. Vergleiche u. a.:
   (a) Zugriff ausschließlich über das bestehende VPN/Firezone (Modell-Client kommt „von innen"),
   (b) MCP hinter Traefik auf HOST-HOMELAB-SRV als authentifizierter Proxy (Authentik/SSO davor),
       der nach NET-IOT nur den MCP-Port zu HA-PROD freigibt,
   (c) der von der Doku vorgeschlagene Webhook-/Reverse-Proxy-Weg nach außen.
3. Für jede Variante: Firewall-/Routing-Änderung (Platzhalter-Ebene), Authentifizierung,
   Secrets-Handhabung (MCP-Token nach Abschnitt 9.1), Blast-Radius wenn der Zugang leakt,
   und Datenschutz (was sieht das externe Modell?).
4. Empfehlung + Begründung + Schritt-für-Schritt-Skizze. Offene Annahmen markieren.
```

---

*Ende des Extracts. Reale Werte stehen ausschließlich in der lokalen, nicht eingecheckten
`_legend.local.md`.*
