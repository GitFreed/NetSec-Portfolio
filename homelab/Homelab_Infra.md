# 🏗️ Cartographie de l'Infrastructure Homelab

```txt
$$\   $$\                                   $$\                $$\       
$$ |  $$ |                                  $$ |               $$ |      
$$ |  $$ | $$$$$$\  $$$$$$\$$$$\   $$$$$$\  $$ |      $$$$$$\  $$$$$$$\  
$$$$$$$$ |$$  __$$\ $$  _$$  _$$\ $$  __$$\ $$ |      \____$$\ $$  __$$\ 
$$  __$$ |$$ /  $$ |$$ / $$ / $$ |$$$$$$$$ |$$ |      $$$$$$$ |$$ |  $$ |
$$ |  $$ |$$ |  $$ |$$ | $$ | $$ |$$   ____|$$ |     $$  __$$ |$$ |  $$ |
$$ |  $$ |\$$$$$$  |$$ | $$ | $$ |\$$$$$$$\ $$$$$$$$\\$$$$$$$ |$$$$$$$  |
\__|  \__| \______/ \__| \__| \__| \_______|\________|\_______|\_______/ 
```

![Hardware](https://img.shields.io/badge/Matériel-HP%20ProDesk%20600%20G4-0096D6?style=flat-square&logo=hp&logoColor=white)
![OS](https://img.shields.io/badge/OS-Proxmox%20VE-E57000?style=flat-square&logo=proxmox&logoColor=white)
![Hardware](https://img.shields.io/badge/Matériel-Raspberry%20Pi-C51A4A?style=flat-square&logo=raspberrypi&logoColor=white)
![OS](https://img.shields.io/badge/OS-Raspbian-005085?style=flat-square&logo=linux&logoColor=white)
![Network](https://img.shields.io/badge/Réseaux-WAN%20LAN%20DMZ-purple?style=flat-square)

---

## Pourquoi un Homelab ?

Ce lab est né d'une conviction : la meilleure façon de comprendre les infrastructures réseau, c'est de les construire, les casser, et les reconstruire soi-même. Chaque service déployé ici répond à un double objectif — **apprendre** en pratiquant sur du matériel réel, et **reprendre le contrôle** sur ses propres données.

### Auto-hébergement et souveraineté numérique

L'auto-hébergement (*self-hosting*) consiste à faire tourner ses propres services plutôt que de dépendre de fournisseurs tiers. Dans ce lab, cela se traduit concrètement par :

- **DNS souverain** : la chaîne [AdGuard Home](./Homelab_AdGuard.md) + [Unbound](./Homelab_Unbound.md) résout les noms de domaine directement depuis les serveurs racine, sans passer par Google, Cloudflare ou un quelconque intermédiaire soumis à des injonctions de blocage.
- **VPN personnel** : un tunnel [OpenVPN](./Homelab_pfSense.md) chiffré permet d'accéder à l'ensemble de l'infrastructure depuis n'importe où, sans confier son trafic à un VPN commercial.
- **Streaming local** : [Plex](./Homelab_Plex.md) offre un service multimédia complet hébergé sur du stockage physique maîtrisé, sans abonnement ni collecte de données de visionnage.
- **Supervision autonome** : [Checkmk](./Homelab_Checkmk.md) et [CrowdSec](./Homelab_Crowdsec.md) assurent la visibilité et la sécurité de l'ensemble sans dépendance à un SOC externe.

L'intérêt va au-delà de la vie privée : c'est un terrain d'entraînement permanent pour un Administrateur d'Infrastructures Sécurisées, couvrant le réseau, le système, le DevOps et la cybersécurité.

---

## 1. Topologie Physique et Cœur de Réseau (LAN)

Ce segment représente le réseau de confiance domestique et l'accès à Internet.

- **Réseau logique :** `192.168.1.0/24`
- **Box FAI (Routeur de Bordure) :** `192.168.1.254`
  - *Rôle :* Accès WAN (Internet), redirection de ports (Port Forwarding UDP 1194 & TCP 32400) vers pfSense.
  - *Sécurité :* Serveur DHCP natif désactivé pour éviter les conflits (Rogue DHCP).

- **Raspberry Pi :** `192.168.1.250`
  - [AdGuard Home](./Homelab_AdGuard.md) — Serveur DNS (Sinkhole publicitaire/malware) et unique serveur DHCP de la zone LAN.
  - [Unbound](./Homelab_Unbound.md) — Résolveur DNS récursif (port 5335), interroge directement les serveurs racine. DNSSEC + QNAME minimisation.

- **PC Fixe (Poste d'Administration) :** `192.168.1.5`
  - *Rôle :* Accès aux interfaces de gestion (Proxmox, pfSense, Checkmk) et au réseau isolé via le routage local.

## 2. L'Hyperviseur et la Couche de Gestion (Management)

Le socle *Bare-Metal* hébergeant l'infrastructure virtualisée.

- **Matériel :** HP ProDesk Mini 600 G4 (BIOS durci : C-States et ASPM désactivés pour garantir la **Disponibilité**).
- **[Proxmox VE](./Homelab_Proxmox.md) (Hôte physique) :** `192.168.1.240` (sur le pont `vmbr0`).
- **[Checkmk](./Homelab_Checkmk.md) (LXC Trixie) :** `192.168.1.241` (sur le pont `vmbr0`).
  - *Rôle :* Supervision active via SNMP et agents. Interroge les équipements du LAN et de la DMZ au travers du pare-feu pfSense.
- **[Samba](./Homelab_Plex.md) (LXC Trixie) :** `192.168.1.242` (sur le pont `vmbr0`).
  - *Rôle :* Serveur de fichiers SMB pour l'accès aux médias depuis les postes Windows.
- **[Scanopy](./Homelab_Scanopy.md) (Sonde LAN — LXC Trixie) :** `192.168.1.243` (sur le pont `vmbr0`).
  - *Rôle :* Cartographie ARP du réseau physique, communique avec le serveur central en DMZ.
- **Nginx (Reverse Proxy — LXC) :** `192.168.1.244` *(prévu)*
  - *Rôle :* Proxy inverse pour les services internes. Permettra d'accéder aux interfaces web via des URL propres (`plex.home`, `checkmk.home`) sans numéros de port, avec terminaison TLS.

## 3. Le Périmètre de Sécurité et la Zone Isolée (DMZ)

C'est ici que s'applique le principe du **Moindre Privilège** (Zero Trust). Les services applicatifs sont confinés et n'ont pas d'accès direct au réseau physique.

- **[pfSense](./Homelab_pfSense.md) (Routeur Virtuel en VM) :**
  - *Interface WAN :* `192.168.1.251` (Connectée à `vmbr0` — reçoit le trafic Internet via la Box).
  - *Interface LAN :* `10.0.0.1` (Passerelle par défaut de la zone isolée — connectée à `vmbr2`).
  - *Rôle :* Pare-feu filtrant, serveur OpenVPN (accès distant), routage inter-VLANs, [CrowdSec](./Homelab_Crowdsec.md) (IDS/IPS collaboratif).
  - **Réseau logique isolé :** `10.0.0.0/24` (porté par le commutateur virtuel `vmbr2`, décorrélé de toute carte réseau physique).

- **[Plex](./Homelab_Plex.md) (Serveur Multimédia en LXC Trixie) :** `10.0.0.10`
  - *Rôle :* Streaming multimédia avec *Bind Mount* vers le HDD 2 To de l'hôte. Accélération matérielle Intel QSV (iGPU).

- **Docker (Moteur de Conteneurs en VM Trixie) :** `10.0.0.20`
  - *Rôle :* Hébergement des services containerisés. L'utilisation d'une VM garantit une isolation stricte des privilèges liés au démon Docker.
  - [Scanopy Server](./Homelab_Scanopy.md) — Serveur central de cartographie réseau (port 60072).
  - **Homarr** — Dashboard centralisé avec thème Catppuccin Macchiato et DNS rewrites AdGuard pour les URLs `.home`.

- **Kali Linux (VM) :** `10.0.0.30`
  - *Rôle :* Poste d'audit et de pentest interne. Isolé en DMZ pour limiter l'impact en cas de manipulation d'exploits.

---

## 🗺️ Représentation Visuelle

Ce schéma représente les flux logiques et la segmentation par ponts virtuels (Linux Bridges).

```txt

[INTERNET / WAN]
    │
    ▼
[Box FAI - 192.168.1.254] (Routage WAN / Port Forwarding UDP 1194 & TCP 32400)
    │
    ├─> [AdGuard Home + Unbound (Pi) - 192.168.1.250] (DNS/DHCP + Résolveur récursif)
    ├─> [PC Fixe - 192.168.1.5] (Poste d'Administration)
    │
    ▼
================== Pont Virtuel vmbr0 (LAN Physique 192.168.1.0/24) ==================
    │
    ├─> [Proxmox VE - 192.168.1.240] (Hyperviseur Bare-Metal)
    ├─> [Checkmk LXC - 192.168.1.241] (Supervision SNMP + Agents)
    ├─> [Samba LXC - 192.168.1.242] (Serveur de fichiers SMB)
    ├─> [Scanopy LXC - 192.168.1.243] (Sonde cartographie LAN)
    ├─> [Nginx LXC - 192.168.1.244] (Reverse Proxy — prévu)
    └─> [pfSense VM - 192.168.1.251] (Routeur / Pare-feu / VPN / CrowdSec)
        │   (Patte WAN : 192.168.1.251 → vmbr0)
        │   (Patte LAN : 10.0.0.1 → vmbr2)
        │
        ▼
================== Pont Virtuel vmbr2 (Zone Isolée / DMZ 10.0.0.0/24) ==================
        │
        ├─> [Plex LXC - 10.0.0.10:32400] (Streaming + Intel QSV)
        ├─> [Kali VM - 10.0.0.30] (Audit / Pentest)
        └─> [Docker VM - 10.0.0.20] (Conteneurs isolés)
            ├─> Scanopy Server (:60072) — Cartographie & IPAM
            └─> Homarr (:7575) — Dashboard


```

---

## 📋 Index des Fiches Techniques

| Service | Fiche | Rôle | Zone |
| --- | --- | --- | --- |
| Proxmox VE | [Homelab Proxmox](./Homelab_Proxmox.md) | Hyperviseur Bare-Metal | Hôte |
| pfSense | [Homelab pfSense](./Homelab_pfSense.md) | Routeur / Pare-feu / VPN | LAN ↔ DMZ |
| AdGuard Home | [Homelab AdGuard](./Homelab_AdGuard.md) | DNS Sinkhole + DHCP | LAN |
| Unbound | [Homelab Unbound](./Homelab_Unbound.md) | Résolveur DNS récursif | LAN |
| CrowdSec | [Homelab CrowdSec](./Homelab_Crowdsec.md) | IDS/IPS Collaboratif | LAN (pfSense) |
| Checkmk | [Homelab Checkmk](./Homelab_Checkmk.md) | Supervision infrastructure | LAN |
| Scanopy | [Homelab Scanopy](./Homelab_Scanopy.md) | Cartographie réseau & IPAM | LAN + DMZ |
| Plex | [Homelab Plex](./Homelab_Plex.md) | Streaming multimédia | DMZ |
| Homarr | [Homelab Homarr](./Homelab_Homarr.md) | Dashboard centralisé | DMZ |
| Nginx | *(à venir)* | Reverse Proxy | LAN |
