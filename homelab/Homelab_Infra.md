# 📄 Cartographie de mon Infrastructure Homelab

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

## 1. Topologie Physique et Cœur de Réseau (LAN Physique)

Ce segment représente le réseau de confiance "domestique" et l'accès à Internet.

* **IP fullstack :** `176.xxx.xxx.xxx`
* **Réseau logique :** `192.168.1.0/24`
* **Box FAI (Routeur de Bordure) :** `192.168.1.254`
* *Rôle :* Accès WAN (Internet), redirection de ports (Port Forwarding UDP 1194 & 32400) vers pfSense.
* *Sécurité :* Serveur DHCP natif obligatoirement désactivé pour éviter les conflits (Rogue DHCP).

* **Raspberry Pi (AdGuard Home) :** `192.168.1.250`
* *Rôle Système/Réseau :* Serveur DNS (Sinkhole pour blocage publicitaire/malware) et unique serveur DHCP de la zone LAN.

* **PC Fixe (Poste d'Administration) :** `192.168.1.5`
* *Rôle :* Accès aux interfaces de gestion (Proxmox, pfSense, Checkmk) et au réseau isolé via le routage local.

## 2. L'Hyperviseur et la Couche de Gestion (Management)

Le socle *Bare-Metal* hébergeant l'infrastructure virtualisée.

* **Matériel :** HP ProDesk Mini 600 G4 (BIOS durci : C-States et ASPM désactivés pour garantir la **Disponibilité**).
* **Proxmox VE (Hôte physique) :** `192.168.1.240` (sur le pont `vmbr0`).
* **Checkmk (LXC Trixie) :** `192.168.1.241` (sur le pont `vmbr0`).
* *Rôle DevOps :* Supervision active via SNMP et agents. Interroge les équipements du LAN et du sous-réseau isolé au travers du pare-feu pfSense.

## 3. Le Périmètre de Sécurité et la Zone Isolée (DMZ / LAN Virtuel)

C'est ici que s'applique le principe du **Moindre Privilège** (Zero Trust). Les services applicatifs sont confinés et n'ont pas d'accès direct au réseau physique.

* **pfSense (Routeur Virtuel en VM) :**
  * *Interface WAN :* `192.168.1.251` (Connectée à `vmbr0` - Reçoit le trafic Internet via la Box).
  * *Interface LAN :* `10.0.0.1` (Passerelle par défaut de la zone isolée - Connectée à `vmbr2`).
  * *Rôle :* Pare-feu filtrant, serveur OpenVPN (accès distant), et routage Inter-VLAN.
  * **Réseau logique isolé :** `10.0.0.0/24` (Porté par le commutateur virtuel `vmbr2`, décorrélé de toute carte réseau physique).

* **Plex (Serveur Multimédia en LXC Trixie) :** `10.0.0.10`
  * *Rôle Système :* Exploitation d'un *Bind Mount* pour l'accès direct et performant aux médias stockés sur l'hôte, sans surcouche d'émulation de disque.

* **Docker (Moteur de Conteneurs en VM Trixie) :** `10.0.0.20`
  * *Rôle DevOps :* Hébergement des futurs services (Homarr, OliveTin, Terraria). L'utilisation d'une VM garantit une isolation stricte des privilèges liés au démon Docker.

---

## 🗺️ Représentation Visuelle

Ce schéma représente les flux logiques et la segmentation par ponts virtuels (Linux Bridges).

```txt

[INTERNET / WAN - 176.xxx.xxx.xxx]
    │
    ▼
[Box FAI - 192.168.1.254] (Routage WAN / Port Forwarding UDP 1194 & 32400)
    │
    ├─> [AdGuard Home (Pi) - 192.168.1.250] (Serveur DNS / DHCP)
    ├─> [PC Fixe - 192.168.1.5] (Poste d'Administration)
    │
    ▼
================== Pont Virtuel vmbr0 (LAN Physique 192.168.1.0/24) ==================
    │
    ├─> [Proxmox VE - 192.168.1.240] (Hyperviseur)
    ├─> [Checkmk LXC - 192.168.1.241] (Supervision)
    ├─> [Samba LXC - 192.168.1.242] (Protocole SMB)
    ├─> [Scanopy LXC - 192.168.1.243] (Daemon cartographie réseau)
    └─> [pfSense VM - 192.168.1.251] (Routeur Virtuel)
        │   (Rôle Pare-feu / Serveur VPN OpenVPN / Inspection des flux)
        │   (Patte WAN : 192.168.1.251 rattachée à vmbr0)
        │   (Patte LAN : 10.0.0.1 rattachée à vmbr2)
        │
        ▼
================== Pont Virtuel vmbr2 (Zone Isolée / DMZ 10.0.0.0/24) ==================
        │
        ├─> [Plex LXC - 10.0.0.10:32400] (Serveur Multimédia avec accès Bind Mount)
        ├─> [Kali VM - 10.0.0.30]
        └─> [Docker VM - 10.0.0.20] (Moteur de conteneurs isolés)
            └─> [Scanopy - 10.0.0.20:60072] (Serveur & Daemon de cartographie réseau)
            └─> [Homarr - 10.0.0.20:7575] (…)


```

[Adguard Home](./Homelab_AdGuard.md) (DHCP & filtrage DNS), [Proxmox VE](./Homelab_Proxmox.md) (Hyperviseur socle du Homelab) , [pfSense](./Homelab_pfSense.md) (Routeur & Pare-feu), [Checkmk Raw](./Homelab_Checkmk.md) (Supervision), [Crowdsec](./Homelab_Crowdsec.md) (Sécurité Collaborative), [Scanopy](./Homelab_Scanopy.md) (Scan Réseau), [Unbound](./Homelab_Unbound.md) (Résolution DNS récursive), [Plex](./Homelab_Plex.md) (Streaming Multimédia) tbc...
