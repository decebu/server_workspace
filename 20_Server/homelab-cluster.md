# Energiesparendes Home-Lab – Raspberry Pi 5 Cluster  
## Phase 1 – Node 1 Basis-Setup & NVMe-Boot

Stand: Phase 1 (Node 1), NVMe-Boot aktiv, WLAN noch an.

---

## 1. Zielsetzung Phase 1

Ziel dieser Phase:

- Ersten Raspberry Pi 5 (8 GB, NVMe) als **Cluster-Node #1** grundlegend in Betrieb nehmen.
- **Stabiles Boot** von NVMe sicherstellen.
- **Aktuellen Bootloader** (EEPROM) einspielen.
- **Netzwerk-Anbindung im Server-VLAN (70)** herstellen.
- DNS-/Netzwerk-Besonderheiten verstehen und dokumentieren.
- Noch **keine** Proxmox-Installation, noch **kein** Cluster, nur Basis-OS.

---

## 2. Hardware-Basis

- **Node 1 Hardware:**
  - Raspberry Pi 5 – 8 GB
  - NVMe SSD (~480–500 GB) über NVMe-HAT (PCIe x1)
  - Aktiver Kühler (Lüfter), kein zusätzlicher Heatsink notwendig
  - Aktuelle Bootloader-Firmware (Stand: 05.11.2025)
- **Netzteil (temporär für Phase 1):**
  - Anker IQ USB-A Multi-Port Netzteil
  - Hinweis: Für Testbetrieb ausreichend, **nicht** für finalen 24/7-Cluster-Betrieb empfohlen (Unterspannungsrisiko bei Last).

---

## 3. Netzwerk-Topologie (relevant für Node 1)

### 3.1 VLAN-Übersicht (aus Sicht des Clusters)

- VLAN 10 – `192.168.10.0/24` – **Management-Netzwerk (Netzwerkgeräte, UDM, Switches)**  
- VLAN 20 – `192.168.20.0/24` – Norden (Reserve / unklar genutzt)
- VLAN 30 – `192.168.30.0/24` – `DECEBU_PROD` – Familiennetzwerk
- VLAN 40 – `192.168.40.0/24` – `DECEBU_GUEST` – Gäste
- VLAN 50 – `192.168.50.0/24` – `DECEBU_IOT` – IoT
- VLAN 51 – `192.168.51.0/24` – `DECEBU_KNX` – KNX-Netz
- VLAN 60 – `192.168.60.0/25` – `DECEBU_WORK` – Arbeits-/Admin-Clients
- VLAN 70 – `192.168.70.0/25` – `DECEBU_DNS` – **Server-/Infra-/DNS-VLAN** (Unbound + Cluster)

### 3.2 Design-Entscheidungen

- **Management-VLAN (10)**: nur für Netzwerkgeräte (UDM, Switches, APs).  
  → **Keine** Server/Cluster-Nodes dort.
- **Server-/Infra-VLAN (70)**:  
  → Hier sollen laufen:
  - Proxmox Nodes (Pi-Cluster, später evtl. NUCs)
  - DNS/Unbound (bis zur Migration vom QNAP)
  - Weitere Infrastruktur-Dienste (später: HA, MQTT, Vaultwarden, Grafana, InfluxDB, etc.)
- **Admin-Zugriff** aus VLAN 60:
  - SSH / HTTPS auf Nodes im VLAN 70 über definierte Firewall-Regeln.

### 3.3 Geplantes IP-/FQDN-Schema (Cluster-relevant)

- Lokale DNS-Zone: `home.decebu.com`
- Geplante FQDNs:
  - `pve-node1.home.decebu.com` → **192.168.70.11**
  - `pve-node2.home.decebu.com` → 192.168.70.12
  - `pve-node3.home.decebu.com` → 192.168.70.13
- Aktueller Stand Node 1 (Phase 1):
  - Läuft via DHCP im VLAN 70 (z. B. `192.168.70.187`)
  - Statische IP wird in **Phase 2** gesetzt.

---

## 4. DNS-Situation (Unbound / QNAP / Übergangsphase)

- **Unbound** läuft aktuell:
  - In einem **Docker-Container** auf dem **QNAP**.
  - DNS-Server-IP im VLAN 70: `192.168.70.6`.

- Beobachtetes Problem:
  - Clients in VLAN 30 (Familiennetz) → `192.168.70.6` als DNS → **funktioniert** (google.com o. ä. wird aufgelöst).
  - Raspberry Pi 5 in VLAN 70 → `192.168.70.6` als DNS → **kann externe Domains NICHT auflösen**.
  - Ursache: Kombination aus **Docker-Netzwerkmodus** (vermutlich `bridge`) und VLAN-/Routing-Regeln → Container nicht direkt im VLAN 70 sichtbar / erreichbar.

- Temporäre Lösung für Phase 1:
  - Am Pi wurde der DNS auf **externen Resolver** (z. B. `8.8.8.8` / UDM-Auto-DNS) umgestellt, damit:
    - `apt update` und Paketinstallation funktionieren.
    - Firmware-/Bootloader-Updates möglich sind.
  - Plan: Unbound später in einem LXC/Container/VM direkt auf dem Proxmox-Cluster betreiben (im VLAN 70, ohne Docker-Bridge-Magie).

---

## 5. Node 1 – Phase 1 Ablauf (chronologisch)

### 5.1 Erstaufbau & Netz

1. **Hardware-Montage**:
   - Raspberry Pi 5 in Stack-/Tower-Konfiguration.
   - NVMe-HAT montiert (PCIe).
   - Aktiver Kühler installiert.
   - Noch: WLAN aktiv (wird später deaktiviert).

2. **Netzwerk-Port-Konfiguration (Dream Machine / Ubiquiti)**:
   - Port, an dem der Pi hängt:
     - **Native VLAN: 70 (DECEBU_DNS)**
     - Tagged VLANs: **keine** (Block All)
     - Port Isolation: OFF
     - STP: ON (Rapid STP aktiv)
   - STP-Problematik:
     - Zu Beginn: Port im Status **"STP Port Blocked"**.
     - Lösung: Port als **Edge Port / Port Fast** konfigurieren, damit der Pi als Endgerät behandelt wird und nicht als potenzieller Switch.

3. **DHCP & DNS**:
   - Node 1 bezieht eine IP im VLAN 70 (z. B. `192.168.70.187`).
   - DNS zunächst auf `192.168.70.6` (Unbound), später temporär auf Auto/extern umgestellt.

---

### 5.2 USB-Boot – Raspberry Pi OS Lite

1. **Preparation**:
   - Auf dem Client (Windows):
     - Raspberry Pi Imager gestartet.
     - OS: **Raspberry Pi OS Lite (64-bit)**.
     - Ziel: USB-Stick.
     - Einstellungen (Advanced):
       - Hostname: `pve-node1`
       - Benutzer: `decebu` (oder initial andere Konfiguration)
       - SSH aktiviert
       - WLAN deaktiviert (für Serverrolle)
       - Locale/Timezone: DE / Europe/Berlin

2. **Erstboot von USB**:
   - Pi bootet vom USB-Stick.
   - Node im Netzwerk sichtbar (DHCP).
   - SSH-Zugriff klappt (vom USB-basierenden System).

3. **Firmware-/Bootloader-Update**:
   - Auf dem Pi:
     ```bash
     sudo apt update && sudo apt full-upgrade -y
     sudo rpi-eeprom-update -a
     ```
   - Reboot.
   - Ausgabe (verkürzt):
     - **BOOTLOADER: up to date**
     - Datum: Wed 5 Nov 2025
   - Ergebnis: NVMe-Boot offiziell unterstützt und auf aktuellem Stand.

---

### 5.3 DNS-Probleme & temporäre Umstellung

- Problem: `apt update` via Unbound-DNS (192.168.70.6) schlägt fehl → `Could not resolve host` für Debian-Repos.
- Analyse:
  - VLAN 30 → Unbound: OK.
  - VLAN 70 → Unbound: keine externen Auflösungen.
- Temporärer Fix:
  - `/etc/resolv.conf` bzw. DHCP-/DNS für VLAN 70 so angepasst, dass:
    - UDM oder externer DNS Resolver (z. B. 8.8.8.8) genutzt wird.
  - Ergebnis: `apt update` funktioniert, Paketinstallation und Firmware-Update sind möglich.

---

### 5.4 NVMe-Setup & finaler Root-FS-Move

**Wichtiger Richtungswechsel:**  
Anstatt cloud-init + `user-data` auf NVMe zu verwenden, wurde die **robuste USB→NVMe-Klon-Strategie** gewählt.

1. **NVMe-Erkennung prüfen (vom USB-OS)**:
   ```bash
   lsblk
   ```
2. **NVMe wipen**:
   ```bash
   sudo wipefs -a /dev/nvme0n1
   ```
3. **Kopieren des kompletten funktionierenden USB-Systems auf NVMe**:
   ```bash
   sudo dd if=/dev/sdX of=/dev/nvme0n1 bs=64M status=progress
   ```
4. **Reboot-Bootorder-Realität Raspberry Pi 5**:
- Pi bootet von nvme0n1
5. **Root-Filesystem-Resize (NVMe voll ausnutzen)**:
- raspi-config --> advanced --> expand filesystem
6. **Validierung**:
   ```bash
   mount | grep " / "
   lsblk
   ```