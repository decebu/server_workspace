# CLAUDE.md - server_workspace (Homelab Dokumentation)

## Zweck dieses Repos

Reines Dokumentations-Repository fuer das persoenliche Homelab.
Alle Inhalte sind Markdown-Dateien, Shell-Skripte und Konfigurationsbeispiele.
Es gibt **keinen lauffaehigen Code** in diesem Repo.

## Repo-Struktur

### Aktive Inhalte (meine eigenen)
- `20_Server/` - Nexus Homelab-Server, operative Umsetzung und Cluster-Zielbild
- `50_Dokumentenmanagement/` - Dokumentenmanagement, paperless-ngx, Obsidian, lokale LLM-Pipeline
- `10_Infrastructure/` - Netzwerk, VPN, DNS, Firewall, Raspberry Pi Edge Router
- `40_BackupStrategy/` - Backup-Strategie mit resticprofile, QNAP, Windows-Clients

### Template-Altlasten (aus public Template, weniger relevant)
- `1_Infrastruktur/` - Proxmox, pfSense, Terraform, Ansible-Notizen
- `2_Software/` - Docker, Portainer, GitHub-Notizen
- `3_Server/` - Portainer Stacks, Playbooks

## Verbundenes Code-Repo

Das eigentliche Homelab (Docker Stacks, Konfigurationen) liegt hier:

@/home/decebu/dev/decebu/docker-stacks/CLAUDE.md

Typische Aufgaben benoetigen beide Repos: Doku hier anpassen + Stack-Config dort aendern.

## Privacy-Regeln

Datei: `10_Infrastructure/privacy_rules.json`

Beim Arbeiten mit Inhalten aus diesem Repo gilt:
- Keine echten Public-IPs oder Domainnamen ausgeben (ersetzen durch `<PUBLIC_ENDPOINT>`)
- Keine Keys, Tokens, Passwoerter ausgeben (ersetzen durch `<SECRET>`, `<API_TOKEN>`, `<PASSWORD>`)
- Ausnahmen: Private RFC1918-Ranges (192.168.x, 10.x, 172.16-31.x) duerfen genannt werden

## Sprache & Stil

- Dokumentation ist auf Deutsch
- Kommunikation mit mir auf Deutsch
- Markdown-Dateien direkt bearbeiten, keine zusaetzlichen Wrapper-Dateien erstellen
- Keine Emojis ausser explizit gewuenscht
