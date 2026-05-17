# 🔒 wireguard-vpn-lab

A lightweight, self-hosted VPN built with WireGuard on Oracle Cloud Free Tier (Frankfurt). Zero monthly cost. Full traffic encryption, DNS leak protection, and IP masking — running on a permanent free VM.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        WITHOUT VPN                              │
│                                                                 │
│   Your Laptop ──────────► Finnish ISP ──────────► Internet     │
│   (real IP visible)       (sees all traffic)                    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                         WITH VPN                                │
│                                                                 │
│   Your Laptop                                                   │
│   10.8.0.2/32                                                   │
│       │                                                         │
│       │  ChaCha20 + Poly1305 Encryption                        │
│       │  UDP Port 51820                                         │
│       ▼                                                         │
│   ╔═══════════════════════════════╗                             │
│   ║   Encrypted Tunnel            ║                             │
│   ║   (ISP sees only gibberish)   ║                             │
│   ╚═══════════════════════════════╝                             │
│       │                                                         │
│       ▼                                                         │
│   Oracle Cloud VM — Frankfurt 🇩🇪                               │
│   10.8.0.1/24  │  130.61.115.156                                │
│       │        │                                                │
│       │   iptables NAT + MASQUERADE                            │
│       │   IP Forwarding                                         │
│       ▼                                                         │
│   Internet                                                      │
│   (websites see Frankfurt IP, not your real IP)                 │
└─────────────────────────────────────────────────────────────────┘
```

## What This VPN Does

| Feature | Details |
|---|---|
| **IP Masking** | Websites see Frankfurt IP, not your real IP |
| **ISP Privacy** | Your ISP only sees encrypted traffic to one server |
| **DNS Protection** | DNS queries go through Cloudflare 1.1.1.1 via tunnel |
| **Encryption** | ChaCha20-Poly1305 — military grade |
| **Protocol** | WireGuard over UDP — fastest VPN protocol available |
| **Cost** | €0/month forever |

---

## Stack

| Component | Choice | Why |
|---|---|---|
| VPN Protocol | WireGuard | Fastest, modern, ~4000 lines of code |
| Server OS | Ubuntu 22.04 | Stable, WireGuard kernel native support |
| Cloud Provider | Oracle Cloud Free Tier | Only provider with truly free permanent VM in EU |
| Region | Frankfurt, Germany | Closest free EU region to Finland |
| VM Shape | VM.Standard.E2.1.Micro | Always Free, enough for single-user VPN |
| Encryption | ChaCha20-Poly1305 | WireGuard default, faster than AES on mobile |
| DNS | Cloudflare 1.1.1.1 | Fast, privacy-focused |

---

## Prerequisites

- Oracle Cloud account (free — requires card for verification only)
- A Revolut virtual card (optional, for safe Oracle signup)
- SSH client (WSL on Windows, Terminal on Linux/macOS)
- WireGuard client app

---

## Implementation Guide

### Step 1 — Oracle Cloud VM Setup

1. Sign up at [cloud.oracle.com](https://cloud.oracle.com)
2. **Set home region to Frankfurt** (cannot be changed later)
3. Go to **Compute → Instances → Create Instance**
4. Configure:
   - **Name:** `wireguard-vpn-lab`
   - **Image:** Canonical Ubuntu 22.04
   - **Shape:** VM.Standard.E2.1.Micro (Always Free)
   - **Networking:** Create new VCN + public subnet
   - **Public IPv4:** Enable
   - **SSH keys:** Generate and download both keys

> ⚠️ If you see "Out of capacity" for A1.Flex shape, use VM.Standard.E2.1.Micro instead — it's sufficient for a personal VPN.

---

### Step 2 — Open Firewall Ports

**Oracle Security List** (in the console):
- Go to **Networking → Virtual Cloud Networks → your VCN → Security Lists**
- Add Ingress Rule:
  - Source CIDR: `0.0.0.0/0`
  - Protocol: UDP
  - Destination Port: `51820`

**OS-level firewall:**
```bash
sudo iptables -I INPUT -p udp --dport 51820 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 22 -j ACCEPT
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

---

### Step 3 — Connect via SSH

```bash
# Fix key permissions
cp ssh-key-2026-05-17.key ~/.ssh/
chmod 400 ~/.ssh/ssh-key-2026-05-17.key

# Connect
ssh -i ~/.ssh/ssh-key-2026-05-17.key ubuntu@<YOUR_SERVER_IP>
```

---

### Step 4 — Install WireGuard on Server

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install WireGuard
sudo apt install wireguard -y
```

---

### Step 5 — Generate Keys

```bash
# Server keys
wg genkey | sudo tee /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key

# Client keys
wg genkey | sudo tee /etc/wireguard/client_private.key | wg pubkey | sudo tee /etc/wireguard/client_public.key

# View all keys
sudo cat /etc/wireguard/server_private.key
sudo cat /etc/wireguard/server_public.key
sudo cat /etc/wireguard/client_private.key
sudo cat /etc/wireguard/client_public.key
```

> 🔑 Copy all 4 keys somewhere safe before proceeding.

---

### Step 6 — Check Network Interface

```bash
ip link show
```

Look for your main interface — on Oracle Cloud it's typically `enp0s6`. Use this in the config below.

---

### Step 7 — Create Server Config

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
PrivateKey = <server_private_key>
Address = 10.8.0.1/24
ListenPort = 51820
PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o enp0s6 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o enp0s6 -j MASQUERADE

[Peer]
PublicKey = <client_public_key>
AllowedIPs = 10.8.0.2/32
```

> Replace `enp0s6` with your actual interface name if different.

---

### Step 8 — Enable IP Forwarding

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

### Step 9 — Start WireGuard

```bash
# Enable and start
sudo systemctl enable --now wg-quick@wg0

# Verify it's running
sudo wg show
```

Expected output:
```
interface: wg0
  public key: <server_public_key>
  private key: (hidden)
  listening port: 51820

peer: <client_public_key>
  allowed ips: 10.8.0.2/32
```

---

### Step 10 — Fix Traffic Forwarding

```bash
sudo iptables -I FORWARD -i wg0 -o enp0s6 -j ACCEPT
sudo iptables -I FORWARD -i enp0s6 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo netfilter-persistent save
```

---

### Step 11 — Generate Client Config

```bash
sudo cat << EOF | sudo tee /etc/wireguard/client.conf
[Interface]
PrivateKey = $(sudo cat /etc/wireguard/client_private.key)
Address = 10.8.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = $(sudo cat /etc/wireguard/server_public.key)
Endpoint = <YOUR_SERVER_IP>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF

sudo cat /etc/wireguard/client.conf
```

---

### Step 12 — Connect Client

**Windows (recommended):**
1. Download WireGuard from [wireguard.com/install](https://wireguard.com/install)
2. Open WireGuard → **Add Tunnel → Add empty tunnel**
3. Paste the client config
4. Name it `wireguard-vpn-lab`
5. Click **Save → Activate**

**Mobile (iOS/Android):**
```bash
# On server — generate QR code
sudo apt install qrencode -y
sudo qrencode -t ansiutf8 < /etc/wireguard/client.conf
```
Scan with WireGuard mobile app.

**Linux:**
```bash
sudo apt install wireguard -y
sudo nano /etc/wireguard/wg0.conf  # paste client config
sudo wg-quick up wg0
```

---

## Verification Tests

After connecting, run all four tests:

| Test | URL | Expected Result |
|---|---|---|
| IP check | [ifconfig.me](https://ifconfig.me) | Shows Frankfurt server IP |
| Location check | [iplocation.net](https://iplocation.net) | Shows Germany, Frankfurt |
| DNS leak test | [dnsleaktest.com](https://dnsleaktest.com) | All servers in Germany |
| Speed test | [fast.com](https://fast.com) | ~40-50 Mbps |

---

## Network Diagram (Detailed)

```
┌──────────────────────────────────────────────────────────────────────┐
│  CLIENT (Windows/Linux/Mobile)                                       │
│                                                                      │
│  WireGuard Interface wg0                                             │
│  ├── IP Address:     10.8.0.2/32                                     │
│  ├── DNS:            1.1.1.1 (Cloudflare)                            │
│  └── AllowedIPs:     0.0.0.0/0 (all traffic)                        │
└──────────────────────┬───────────────────────────────────────────────┘
                       │
                       │  UDP/51820
                       │  ChaCha20-Poly1305 encrypted
                       │
┌──────────────────────▼───────────────────────────────────────────────┐
│  ORACLE CLOUD VM — Frankfurt                                         │
│  Ubuntu 22.04 LTS                                                    │
│                                                                      │
│  WireGuard Interface wg0                                             │
│  ├── IP Address:     10.8.0.1/24                                     │
│  ├── Listen Port:    51820                                           │
│  └── Peer:           10.8.0.2/32                                     │
│                                                                      │
│  iptables Rules                                                      │
│  ├── FORWARD:        wg0 → enp0s6 ACCEPT                            │
│  ├── FORWARD:        enp0s6 → wg0 ESTABLISHED,RELATED ACCEPT        │
│  └── NAT POSTROUTING: enp0s6 MASQUERADE                             │
│                                                                      │
│  IP Forwarding:      net.ipv4.ip_forward = 1                        │
└──────────────────────┬───────────────────────────────────────────────┘
                       │
                       │  enp0s6 (public interface)
                       │  Source IP: 130.61.115.156
                       ▼
                    Internet
```

---

## Key Management

> ⚠️ Never commit private keys to Git.

All key files are stored server-side in `/etc/wireguard/` with strict permissions:

```
/etc/wireguard/
├── wg0.conf              # server config (chmod 600)
├── client.conf           # client config (chmod 600)
├── server_private.key    # (chmod 600)
├── server_public.key     # (chmod 644)
├── client_private.key    # (chmod 600)
└── client_public.key     # (chmod 644)
```

---

## Adding More Devices

For each new device, generate a new key pair and add a new `[Peer]` block to the server config:

```bash
# Generate new client keys
wg genkey | sudo tee /etc/wireguard/client2_private.key | wg pubkey | sudo tee /etc/wireguard/client2_public.key

# Add to server config
sudo nano /etc/wireguard/wg0.conf
```

Add under existing `[Peer]`:
```ini
[Peer]
PublicKey = <client2_public_key>
AllowedIPs = 10.8.0.3/32
```

Reload without dropping connections:
```bash
sudo wg syncconf wg0 <(sudo wg-quick strip wg0)
```

Each device gets its own IP: `10.8.0.2`, `10.8.0.3`, `10.8.0.4` etc.

---

## Useful Commands

```bash
# Check VPN status
sudo wg show

# Start/stop VPN
sudo systemctl start wg-quick@wg0
sudo systemctl stop wg-quick@wg0

# Restart VPN
sudo systemctl restart wg-quick@wg0

# View live traffic
sudo wg show wg0 transfer

# Check iptables rules
sudo iptables -L FORWARD -v
sudo iptables -t nat -L POSTROUTING -v

# View logs
sudo journalctl -u wg-quick@wg0 -f
```

---

## Security Notes

- Private keys never leave the server
- Each client has a unique key pair
- `AllowedIPs = 0.0.0.0/0` routes ALL traffic through VPN (full tunnel)
- WireGuard uses perfect forward secrecy — session keys rotate automatically
- No logs are kept on the server

---

## Related Projects

- [ubuntu-cloud-lab](https://github.com/Ahmedaltu/ubuntu-cloud-lab)
- [metal3-kubernetes-lab](https://github.com/Ahmedaltu/metal3-kubernetes-lab)
- [istio-service-mesh-lab](https://github.com/Ahmedaltu/istio-service-mesh-lab)

---

## License

MIT
