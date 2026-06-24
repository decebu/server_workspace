# WireGuard VPN Bridge Dokumentation

## 📌 Netzwerkarchitektur

Die WireGuard-VPN-Bridge verbindet mehrere Heimnetze und mobile Clients. Alle Heimnetze und Peers nutzen den zentralen VPS als Hub.

### Subnetze der Heimnetze

| Netz           | Funktion                              | Subnetz         |
| -------------- | ------------------------------------- | --------------- |
| Heimnetz 1     | Admin / Core                          | 192.168.10.0/24 |
|                | Productive                            | 192.168.30.0/24 |
|                | Guests                                | 192.168.40.0/24 |
|                | IoT                                   | 192.168.50.0/24 |
|                | Work                                  | 192.168.60.0/24 |
| Norden         | Hauptnetz (Norden)                | 192.168.20.0/24 |
| Callitectum    | Remote-Netz (externe FRITZ!Box)       | 192.168.25.0/24 |
| VPN-Clients    | Mobile Devices (Firezone)             | 10.10.50.0/24   |

### WireGuard Overlay-IPs (Tunnel)

| Peer                          | Tunnel-IP          | Interface VPS  | Beschreibung                               |
| ----------------------------- | ------------------ | -------------- | ------------------------------------------ |
| VPS (HN1-Hub, Server)         | 10.10.10.1         | `unifi`        | Hub / Bridge Heimnetz 1                    |
| UDM Pro (HN1)                 | 10.10.10.2         | –              | Gateway Heimnetz 1 (WG-Client)             |
| VPS (Norden, Server)          | 10.10.20.1         | `hn2`          | Hub für Norden / Norden                |
| FRITZ!Box Norden              | 10.10.20.2         | –              | Gateway Norden (WG-Client)                 |
| VPS (Callitectum, **Client**) | 192.168.25.203/24  | `callitectum`  | VPS verbindet sich zur FRITZ!Box Callitectum |
| FRITZ!Box Callitectum         | 192.168.25.1       | –              | WG-Server (externe Gegenstelle)            |

---

## 1. VPS Setup

### Installation & Absicherung

```bash
sudo apt update && sudo apt upgrade -y && sudo apt install -y software-properties-common wireguard wireguard-tools
```

**Ip-Forwarding aktivieren:**

```bash
sudo nano /etc/sysctl.conf
```

**Folgende Zeilen am Ende hinzufügen:**

```ini
# Enable IPv4 packet forwarding
net.ipv4.ip_forward=1

# Enable Proxy ARP
net.ipv4.conf.all.proxy_arp=1

# Disable IPv6 packet forwarding
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

**Änderungen aktivieren:**

```bash
sudo sysctl -p && sudo sysctl --system
```

---

## 2. Schlüsselgenerierung

### UDM Pro (Heimnetz 1)

**Private und Public Key:**

```bash
(umask 077 && printf "PrivateKey= " | sudo tee /etc/wireguard/privatekey_unifi > /dev/null) \
&& wg genkey | sudo tee -a /etc/wireguard/privatekey_unifi | wg pubkey | sudo tee /etc/wireguard/publickey_unifi \
&& sudo cat /etc/wireguard/privatekey_unifi && sudo touch /etc/wireguard/unifi.conf
```

**Preshared Key:**

```bash
wg genpsk | sudo tee /etc/wireguard/psk_unifi
```

**Keys anzeigen:**

```bash
sudo cat /etc/wireguard/privatekey_unifi
sudo cat /etc/wireguard/publickey_unifi
sudo cat /etc/wireguard/psk_unifi
```

### Norden (FRITZ!Box)

**Private und Public Key:**

```bash
(umask 077 && printf "PrivateKey= " | sudo tee /etc/wireguard/privatekey_hn2 > /dev/null) \
&& wg genkey | sudo tee -a /etc/wireguard/privatekey_hn2 | wg pubkey | sudo tee /etc/wireguard/publickey_hn2 \
&& sudo cat /etc/wireguard/privatekey_hn2 && sudo touch /etc/wireguard/hn2.conf
```

**Preshared Key:**

```bash
wg genpsk | sudo tee /etc/wireguard/psk_hn2
```

**Keys anzeigen:**

```bash
sudo cat /etc/wireguard/privatekey_hn2
sudo cat /etc/wireguard/publickey_hn2
sudo cat /etc/wireguard/psk_hn2
```

### Callitectum (VPS als WG-Client zur externen FRITZ!Box)

> Die FRITZ!Box ist hier der WireGuard-Server. Der VPS verbindet sich als Client. Der Public Key des VPS muss beim FRITZ!Box-Betreiber als Peer eingetragen werden.

**Private und Public Key generieren:**

```bash
cd /etc/wireguard
wg genkey | tee privatekey_callitectum | wg pubkey > publickey_callitectum
chmod 600 privatekey_callitectum
```

**Public Key ausgeben (an FRITZ!Box-Betreiber übergeben):**

```bash
cat /etc/wireguard/publickey_callitectum
```

---

## 3. VPS Konfigurationen

### UDM Peer (`/etc/wireguard/unifi.conf`)

```ini
[Interface]
Address    = 10.10.10.1/32
ListenPort = 55120
PrivateKey = <VPS_PRIVATE_UNIFI>

# Routing / NAT (Multi-Interface):
# - unifi  = WG-Interface für HN1
# - eth0   = WAN-Interface (Ionos) Die Netzwerkkarte eth0 muss angepasst werden (z.B. via `ip addr` prüfen)
PostUp     = iptables -t nat -A POSTROUTING -o unifi -j MASQUERADE
PostDown   = iptables -t nat -D POSTROUTING -o unifi -j MASQUERADE
PostUp     = iptables -t nat -A POSTROUTING -o eth0  -j MASQUERADE
PostDown   = iptables -t nat -D POSTROUTING -o eth0  -j MASQUERADE

[Peer] # UDM Pro
PublicKey    = <UDM_PUBLIC>
PresharedKey = <PSK_UNIFI>
AllowedIPs   = 10.10.10.2/32, 192.168.10.0/24, 192.168.30.0/24, 192.168.50.0/24
# bei AllowedIPs gegebenenfalls weitere Heimnetzsubnetze eintragen. Die Freigabe einzelner IPs erfolgt dann in der UDMPro Firewall
```

### Norden FRITZ!Box Peer (`/etc/wireguard/hn2.conf`)

```ini
[Interface]
Address    = 10.10.20.1/32
ListenPort = 51822
PrivateKey = <VPS_PRIVATE_HN2>

# Routing / NAT (Multi-Interface):
# - hn2   = WG-Interface für HN2
# - eth0  = WAN-Interface (Ionos)
PostUp     = iptables -t nat -A POSTROUTING -o hn2  -j MASQUERADE
PostDown   = iptables -t nat -D POSTROUTING -o hn2  -j MASQUERADE
PostUp     = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown   = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer] # FRITZ!Box HN2
PublicKey    = <FRITZ_PUBLIC>
PresharedKey = <PSK_HN2>
AllowedIPs   = 10.10.20.2/32, 192.168.20.0/24
```

### Callitectum (`/etc/wireguard/callitectum.conf`)

> VPS agiert als WG-Client. Kein `ListenPort`. Die FRITZ!Box ist der Server mit öffentlichem Endpoint via MyFRITZ!.
>
> **Achtung:** Die FRITZ!Box liefert in ihrer Client-Config `AllowedIPs = 0.0.0.0/0` (Full Tunnel). Das darf auf dem VPS **nicht** übernommen werden – es würde den gesamten VPS-Traffic durch die FRITZ!Box leiten. Stattdessen nur das Callitectum-Subnetz eintragen.

```ini
[Interface]
Address    = 192.168.25.203/24
PrivateKey = <VPS_PRIVATE_CALLITECTUM>

# MASQUERADE: Heimnetz-Traffic erscheint bei FRITZ!Box mit VPS-IP (192.168.25.203)
PostUp   = iptables -t nat -A POSTROUTING -o callitectum -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -o callitectum -j MASQUERADE

[Peer]  # FRITZ!Box Callitectum
PublicKey           = <FRITZ_CALLITECTUM_PUBLIC>
PresharedKey        = <PSK_CALLITECTUM>
Endpoint            = <MYFRITZ_HOSTNAME>:52721
AllowedIPs          = 192.168.25.0/24
PersistentKeepalive = 25
```

### Dienste starten & aktivieren

```bash
sudo wg-quick up unifi
sudo wg-quick up hn2
sudo wg-quick up callitectum
sudo systemctl enable wg-quick@unifi wg-quick@hn2 wg-quick@callitectum
```

---

## 4. Heimnetz 1 (UDM Pro)

### 4.1 UDM als VPN Client

> Hinweis: Die UDM Pro unterstützt aktuell **kein WireGuard Site-to-Site VPN**. Deshalb wird die Verbindung als *VPN Client* eingerichtet. Dadurch verhält sich HN1 wie ein Client zum VPS-Hub. Zukünftig sollte geprüft werden, ob Ubiquiti Site-to-Site mit WireGuard nachrüstet, um statische Routen und Workarounds zu vermeiden.

- Modus: **VPN Client (WireGuard)**
- Tunnel-IP: `10.10.10.2/32`
- VPS Endpoint: `<VPS_HOST>:55120`
- Keys: `<UDM_PRIVATE>` / `<VPS_PUBLIC_UNIFI>`
- Preshared Key: `<PSK_UNIFI>`

### 4.2 Statische Routen (Kompensation)

Da kein echtes Site-to-Site vorhanden ist, müssen die Netze von HN2 und Firezone explizit auf die Tunnel-IP des VPS geroutet werden. Das entfällt, wenn AllowedIPs verwendet werden kann. Da werden die Routen automatisch gesetzt. 

- `192.168.20.0/24 → 10.10.10.2` (Norden / HN2)
  - Distance 1
  - Next Hop
- `10.10.50.0/24 → 10.10.10.2` (Firezone)
  - Distance 1
  - Next Hop
- `10.10.10.0/24 → 10.10.10.2` (VPN Overlay HN1)
  - Distance 1
  - Next Hop
- `10.10.20.0/24 → 10.10.10.2` (VPN Overlay Norden)
  - Distance 1
  - Next Hop
- `192.168.25.0/24 → 10.10.10.1` (Callitectum, via VPS hn-callitectum)
  - Distance 1
  - Next Hop
---

## 5. Heimnetz 2 (FRITZ!Box 5530)

Die Fritzbox kann eine fertige Config importieren. Private Key und PresharedKey einfach auf einem anderen Linux Rechner erzeugen und die Conf Datei eintragen. 

### Import-Konfiguration `fritz-hn2.conf`

```ini
[Interface]
Address    = 10.10.20.2/32
PrivateKey = <FRITZ_PRIVATE>
# DNS = 192.168.10.1 #Optional

[Peer]
PublicKey    = <VPS_PUBLIC_HN2>
PresharedKey = <PSK_HN2>
Endpoint     = <VPS_HOST>:51822
PersistentKeepalive = 25

AllowedIPs = 10.10.20.1/32, 10.10.10.1/32, 10.10.10.2/32, \
             192.168.10.0/24, 192.168.30.0/24, 192.168.40.0/24, \
             192.168.50.0/24, 192.168.60.0/24
# Netzwerke der Gegenstelle (Split-Tunnel, streng). 
# Hier müssen alle Netze aufgenommen werden, die später aus dem Netz der Fritzbox remote erreichbar sein sollen. 
```

---

## 6. Testen der WireGuard Verbindungen

Nach dem Start der Tunnel sollten die Grundfunktionen geprüft werden.

### 6.1 Handshake prüfen

Auf dem VPS oder einem Gateway:

```bash
sudo wg show
```

- `latest handshake` sollte wenige Sekunden alt sein.
- `transfer` zeigt, ob Daten in beide Richtungen fließen.

### 6.2 Ping-Tests

Vom VPS:

```bash
# Ping UDM im Tunnel
ping 10.10.10.2

# Ping Norden FRITZ!Box im Tunnel
ping 10.10.20.2

# Ping Callitectum Gateway (FRITZ!Box)
ping 192.168.25.1

# Ping eigene VPS-Tunnel-IP Callitectum
ping 192.168.25.203
```

Vom Client in HN2:

```bash
# Ping VPS im Tunnel
ping 10.10.20.1

# Ping VPS zentrales Interface
ping 10.10.10.1
```

### 6.3 Basis-Zugriffe

HTTP-Check (sofern ein Webdienst wie UDM-UI oder Home Assistant freigegeben ist):

```bash
curl -k https://192.168.10.1
```

### Hinweis

Diese Tests funktionieren auch ohne konfigurierte Firewall-Regeln, da sie nur die Erreichbarkeit der Tunnel-Endpunkte und den Aufbau des Overlays prüfen. Für Zugriffe auf interne Dienste sind anschließend die Firewall-Regeln auf UDM/VPS erforderlich.

