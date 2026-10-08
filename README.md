# Homelab

This is my first homelab and it is operated headlessly via Proxmox web UI or SSH.

---

## Why I built this
Initially I was pulled into the world of homelabs due to my experience dual booting Windows and Linux and then switching to using WSL to avoid issues with having Linux and Windows on the same drive. After doing some research in the different areas of cybersecurity, participating in CTFs and doing some modules in TryHackMe I started to get really curious about the world of cybersecurity and IT, which introduced me to the world of having your own home-server/homelab.

I wanted to build a homelab because I was excited to have my own personal server, to host my own services, to have multiple VMs to test new Linux distros without the need of dual booting and to have a testing environment for other projects or things I would want to learn. On top of this, it would make me learn useful skills and knowledge that could be transferred into a field I was starting to feel really passionate about.
## What it's for
- **Future Cybersecurity labs**: pentesting, malware analysis, purple team exercises
- Isolated, disposable environments for testing tools and techniques
- A GPU-accelerated Windows VM for gaming, streamed remotely
- Self-hosted services 
- Getting hands-on experience with virtualization, networking, and Linux systems administration by running real infrastructure instead of only watching videos on YouTube about it
---

## Architecture

```
                              Internet
                                 │
                          Cloudflare DNS
                        (DNS-only, grey cloud)
                                 │
                          ┌──────┴──────┐
                          │  Tailscale  │  (private mesh, all nodes)
                          └──────┬──────┘
                                 │
        ┌────────────────────────────────────────────-┐
        │                  SRVlab (Proxmox)           │
        │  Ryzen 7 2700X · 48GB DDR4 · RTX 2060S (PT) │
        │  2TB NVMe · 2TB HDD · APC UPS + NUT         │
        │                                             │
        │ ┌──────────┐  ┌──────────┐  ┌─────────────┐ │
        │ │ Windows VM│  │  OMV VM  │  │ Traefik LXC│ │
        │ │ (gaming,  │  │ (2TB HDD │  │ (reverse   │ │
        │ │  GPU PT)  │  │  raw PT) │  │  proxy)    │ │
        │ └──────────┘  └──────────┘  └─────────────┘ │
        │  Lab VMs (Kali, Debian, targets)            │
        └────────────────────────────────────────────-┘
                                 │
                         Raspberry Pi 4
                     (Wake-on-LAN relay, DNS)
                                 │
                    ┌────────────────────────┐
                    │                        │
                    │         Main PC        │
                    │                        │
                    └────────────────────────┘
```

---

## Hardware

### SRVlab (Proxmox host)

| Component             | Spec                                                                                                    |
| --------------------- | ------------------------------------------------------------------------------------------------------- |
| CPU                   | AMD Ryzen 7 2700X                                                                                       |
| RAM                   | 48GB DDR4                                                                                               |
| Motherboard           | ASUS Prime X470-PRO                                                                                     |
| GPU (passthrough)     | NVIDIA RTX 2060 Super                                                                                   |
| GPU (spare/emergency) | AMD RX 470 (kept on hand for physical/emergency video output if the host ever needs a monitor attached) |
| Storage (OS/VMs)      | 2TB NVMe M.2                                                                                            |
| Storage (data)        | 2TB HDD (passed through raw to an OMV VM)                                                               |
| UPS                   | APC Back-UPS XS 1000M, monitored via NUT                                                                |
### Raspberry Pi 4

Hosts Home Assistant and helps with a Wake-on-LAN (WoL) relay for both the homelab and the main PC. It sits on the LAN and sends the magic packet locally when I trigger it remotely over Tailscale. Haven't found another use for it yet.

---

## Software stack

| Layer                        | Tool                                       |
| ---------------------------- | ------------------------------------------ |
| Hypervisor                   | Proxmox VE                                 |
| Storage management           | OpenMediaVault (OMV), raw disk passthrough |
| Remote access / mesh network | Tailscale                                  |
| Reverse proxy / TLS          | Traefik + Cloudflare DNS-01                |
| SSH                          | OpenSSH                                    |
| Power protection             | APC UPS + NUT (Network UPS Tools)          |
| Automation                   | Wake-on-LAN via Raspberry Pi               |

---
## Network Configuration
- **Tailscale mesh network across every node (Homelab, Pi, main PC)**
	- This is the backbone of remote access; nothing is exposed to the public internet directly.
- **Cloudflare DNS, DNS-only (grey cloud)**
	- Records point at Tailscale IPs (`100.x.x.x`). Cloudflare's proxy layer can't reach a Tailscale address anyway, so grey-cloud is mandatory here, not optional.
- **Static/reserved IPs** on the local subnet for SRVlab and the Pi.

---
## Remote Access and Security
### SSH
- OpenSSH with key-based auth
- Uses `~/.ssh/config` and Tailscale Magic DNS feature to make it easier to connect to my devices
### Tailscale Mesh VPN
- Encrypted access to every node without opening any ports on the router. Safer than port forwarding
### Reverse Proxy / TLS
- Traefik running as an LXC, fronting internal services with Let's Encrypt certs issued via Cloudflare DNS-01 validation. Proxmox's own web UI also has its own independent ACME setup (native, not through Traefik)
- Two routing methods: Docker-label auto-discovery for containers on the Traefik LXC itself, and a file-provider config for everything else (Proxmox, Home Assistant on the Pi).

### VMs GUI access
- SPICE (`virt-viewer`) for VM consoles

---
## Services
### Gaming VM with Sunshine / Moonlight
- Windows 11 VM with the passed-through RTX 2060 Super, running Sunshine as the streaming host. I connect via Moonlight from the main PC or elsewhere on the tailnet.

### NAS / Storage thanks to OpenMediaVault
- 2TB HDD passed through raw to an OMV VM, shared over the network. One real constraint worth documenting: **Proxmox can't snapshot a raw-passthrough disk**, so the OMV VM has no snapshot safety net ;(

### Cybersecurity Lab (in progress)
- Planned dedicated Proxmox network bridges for isolation; a management bridge, a lab bridge, and a malware-analysis bridge with no gateway at all

### VPS-test (in progress)
- Planned to be used as a testing environment for when I have a public VPS (like a test and prod environment)
### Power Protection with UPS + NUT
- APC Back-UPS XS 1000M monitored via NUT in standalone mode directly on the Proxmox host, configured to trigger a graceful shutdown.
- Integrated into Home Assistant for live battery/status monitoring.

### Home Assistant
- It replaced `Homepage` because I found Home Assistant more useful and flexible.

---
## Notes
- The biggest gap is that all my access to the homelab depends on Tailscale.
- UPS/NUT would sometimes fail to start after a power-loss reboot — fixed with a systemd override so it waits on the driver and retries.

---
## Current status
- [x] Proxmox host installed and hardened
- [x] GPU passthrough 
- [x] Windows gaming VM with Sunshine/Moonlight streaming
- [x] OMV VM with 2TB HDD raw passthrough
- [x] Tailscale mesh across all nodes
- [x] Traefik reverse proxy with Cloudflare DNS-01 certs
- [x] SPICE console tunneled through Traefik
- [x] UPS + NUT with graceful shutdown
- [x] Raspberry Pi Wake-on-LAN relay
- [ ] Cybersecurity lab with isolated lab network segments
- [ ] Immich (self-hosted photo management)
- [ ] Authentik (SSO for family-facing services)
- [ ] Managed switch / VLAN segmentation for lab isolation

---
## Roadmap
Finish isolating the cybersecurity lab network and eventually move to a managed switch for proper VLAN-based segmentation.

