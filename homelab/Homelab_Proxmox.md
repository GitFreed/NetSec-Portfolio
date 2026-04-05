# 🗄️ LAB : Déploiement Proxmox VE (PVE)

```txt
█████▄ ▄▄▄▄   ▄▄▄  ▄▄ ▄▄ ▄▄   ▄▄  ▄▄▄  ▄▄ ▄▄ 
██▄▄█▀ ██▄█▄ ██▀██ ▀█▄█▀ ██▀▄▀██ ██▀██ ▀█▄█▀ 
██     ██ ██ ▀███▀ ██ ██ ██   ██ ▀███▀ ██ ██
```

![Hardware](https://img.shields.io/badge/Matériel-HP%20ProDesk%20600%20G4-0096D6?style=flat-square&logo=hp&logoColor=white)
![OS](https://img.shields.io/badge/OS-Proxmox%20VE-E57000?style=flat-square&logo=proxmox&logoColor=white)
![Role](https://img.shields.io/badge/Rôle-Hyperviseur-68bc71?style=flat-square)
![Network](https://img.shields.io/badge/Réseau-LAN-purple?style=flat-square)

**Rôle :** Hyperviseur Bare-Metal (Type 1) & Plateforme d'Infrastructure

**Mission :** Déployer un environnement de virtualisation polyvalent (Homelab). L'objectif est double : fournir un véritable "bac à sable" pour concevoir des topologies réseau sur-mesure (segmentation, routage, pare-feu), tout en hébergeant efficacement des machines virtuelles et des conteneurs légers pour des services internes (supervision, multimédia, automatisation).

> 📚 Documentation : <https://pve.proxmox.com/pve-docs/>

---

## L'intérêt technique 🎯

1. **Performance Bare-Metal :** Installation directe sur le matériel (Type 1) pour une allocation brute et optimisée des ressources (CPU, RAM, I/O disques), sans la contrainte d'un système d'exploitation hôte.
2. **Flexibilité de Déploiement (VMs & LXC) :** Capacité à instancier à la volée des appliances réseau (pfSense), des machines clientes complètes, ou des conteneurs Linux ultra-légers dédiés à un service unique (microservices).
3. **Maîtrise des Flux (SDN) :** Exploitation native des commutateurs virtuels (Linux Bridges) pour dessiner des architectures L2/L3 complexes et isoler les domaines de diffusion.
4. **Sécurité et Maquettage :** Tester, casser et reconstruire des architectures en toute sécurité grâce aux sauvegardes et aux instantanés (*Snapshots*).

---

## 🛠️ Caractéristiques du Serveur

L'infrastructure repose sur un micro-serveur optimisé et mis à niveau (*upgrade processeur i3 → i7, RAM 8 Go → 32 Go, HDD 256 Go → SSD 512 Go + HDD 2 To*) :

- **Châssis :** HP ProDesk 600 G4 (Format Mini-PC, silencieux et basse consommation)
- **Processeur :** Intel Core i7-8700T (6 cœurs / 12 threads)
- **Mémoire :** 32 Go (2x16 Go SK-Hynix SODIMM DDR4)
- **Stockage Système :** SSD Toshiba 512 Go (OS + VMs)
- **Stockage Data :** HDD Toshiba 2 To (ISOs, sauvegardes, médias)

💰 *Budget : 300€*

![prodesk](/images/2026-03-05-01-10-17.png)

---

## Pré-requis 📋

- Une clé USB (minimum 8 Go) vierge
- L'image ISO officielle de Proxmox VE
- Un utilitaire de flashage bootable (BalenaEtcher ou Rufus)
- Une connexion réseau filaire (câble RJ45)

---

## 1️⃣ Installation (Bare-Metal)

1. **Flashage** de l'ISO sur la clé USB
2. **Boot** sur la clé (touche F9 ou F10 dans le BIOS)
3. Sélection **"Install Proxmox VE (Graphical)"**
4. **Disque cible :** sélectionner le SSD 512 Go (⚠️ pas le HDD 2 To)
5. **Configuration Réseau initiale :**
   - Interface réseau physique : `eno1` / `nic0`
   - FQDN : `pve-server.lan`
   - IP statique : `192.168.1.240/24`
   - Passerelle : `192.168.1.254`
   - *Cette étape crée automatiquement le premier pont virtuel `vmbr0`*
6. **Mot de passe root** et email de contact
7. **Redémarrage** → interface web accessible sur `https://192.168.1.240:8006`

---

## 2️⃣ Configuration Post-Installation

### Fiabilisation des dépôts (Repositories)

Par défaut, Proxmox interroge les dépôts "Enterprise" (payants). Bascule vers les dépôts "No-Subscription" pour le Lab :

```bash
# Script communautaire de post-installation
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/misc/post-pve-install.sh)"
```

Ce script désactive le dépôt Enterprise, active le dépôt No-Subscription, supprime le pop-up de souscription, et lance les mises à jour.

**PVE Post-install** : <https://community-scripts.github.io/ProxmoxVE/scripts?id=post-pve-install>

Corriger le dernier dépôt : **pve-server > Updates > Repositories** → Disable `ceph.sources` et `pve-enterprise.sources`.

![disable](/images/2026-03-02-09-16-46.png)

---

## 3️⃣ Architecture Réseau Virtuelle (SDN)

Configuration via **PVE-Server > System > Network**.

### Les Commutateurs Virtuels (Bridges)

| Bridge | Rôle | Adressage | Lien physique |
| --- | --- | --- | --- |
| `vmbr0` | LAN physique (Uplink WAN) | `192.168.1.240/24` | `eno1` (carte réseau) |
| `vmbr2` | DMZ isolée (Switch virtuel pur) | Aucune IP côté hyperviseur | Aucun |

- **`vmbr0` :** pont physique lié à `eno1`, porte l'IP de management et la passerelle par défaut. L'interface WAN de pfSense (`192.168.1.251`) y est directement rattachée.
- **`vmbr2` :** commutateur virtuel isolé, aucune connectivité physique. Porte le domaine de diffusion DMZ (`10.0.0.0/24`). L'interface LAN de pfSense (`10.0.0.1`) y est rattachée.

### Le Trunking (802.1Q)

L'activation de `VLAN aware` sur un bridge permet de gérer les sous-interfaces et les VLANs directement depuis les instances réseau (Cisco, pfSense, VyOS, etc.).

![vlanaware](/images/2026-03-01-23-52-19.png)

### Activation du routage IP (IP Forward)

Pour que l'hyperviseur puisse relayer des paquets entre ses interfaces (nécessaire pour certaines redirections de ports) :

```bash
# Fichier de routage prioritaire
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-routing.conf

# Appliquer immédiatement
sysctl --system
```

### Configuration réseau actuelle (`/etc/network/interfaces`)

```sh
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
        address 192.168.1.240/24
        gateway 192.168.1.254
        bridge-ports eno1
        bridge-stp off
        bridge-fd 0

auto vmbr2
iface vmbr2 inet manual
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        bridge-vlan-aware yes
```

> 💡 Depuis la simplification de l'architecture, Proxmox ne gère plus de règles NAT/iptables. Toutes les redirections de ports (VPN 1194, Plex 32400) sont assurées par la Box FAI directement vers pfSense (`192.168.1.251`). Voir la [fiche pfSense](./Homelab_pfSense.md).

---

## 4️⃣ Partitionnement du Stockage 💾

Cloisonnement SSD (performances) / HDD (capacité) :

### Initialiser le HDD

**Disks > Directory > Create Directory :**

- Disque : HDD 2 To
- Système de fichier : **ext4**
- Nom : `HDD-data`

![disks](/images/2026-03-02-00-14-45.png)

### Attribuer les rôles

| Stockage | Contenu autorisé |
| --- | --- |
| **local-lvm** (SSD) | Disques VM, Containers |
| **local** (SSD racine) | Import uniquement |
| **HDD-Data** (HDD) | ISO Images, Backups, Container Templates |

---

## 5️⃣ VMs & Containers 🖧

Activer **Start at boot** dans les options de chaque machine pour le redémarrage automatique.

**Réglages VM :**

- SSD **Emulation** ✅
- **Discard** ✅
- **VirtIO** pour Linux, **Intel E1000** pour Windows

**Réglages LXC :**

- Laisser **Unprivileged container** coché (sécurité par conception — impossible d'obtenir root sur l'hôte même en s'échappant du conteneur)
- Autorisation SSH temporaire en lab (⚠️ pas en production) :

```bash
sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin yes/' /etc/ssh/sshd_config
systemctl restart ssh
```

💡 Pour supprimer une ancienne clef SSH d'une IP de VM :

```bash
ssh-keygen -R 192.168.1.x
```

---

## 6️⃣ Correction du NTP (Network Time Protocol)

Checkmk remontait un WARN sur le NTP du PVE-Server : "Stratum 4" avec un décalage (Offset).

![NTP error](/images/2026-03-05-00-07-48.png)

### Modifier les sources de temps

```bash
nano /etc/chrony/chrony.conf
```

Commenter le pool par défaut et ajouter les serveurs français :

```text
#pool 2.debian.pool.ntp.org iburst
pool 0.fr.pool.ntp.org iburst
pool 1.fr.pool.ntp.org iburst
pool 2.fr.pool.ntp.org iburst
```

```bash
# Relancer le service
systemctl restart chrony

# Vérifier la synchronisation
chronyc sources -v
```

![NTP](/images/2026-03-05-00-12-49.png)

---

## 7️⃣ Incident : Mise à jour BIOS et reboots intempestifs 💥

### Description de l'incident

- **Symptômes :** Déconnexions aléatoires et redémarrages intempestifs des VMs
- **Déclencheur :** Mise à jour du firmware BIOS (HP ProDesk 600 G4)
- **Impact (DIC) :** Perte critique du pilier **Disponibilité**

### Investigation et Analyse des Logs

```bash
# État de l'hôte
uptime

# Journaux depuis le dernier boot
journalctl --since today

# Erreurs critiques du boot précédent
journalctl -p 3 -xb -1

# Erreurs critiques aujourd'hui
journalctl --since today -p err

# Recherche OOM Killer (RAM)
journalctl --since today | grep -i "oom-killer"

# Recherche erreurs disques
journalctl --since today | grep -iE "I/O error|ext4|xfs"
```

**Résultat :** `uptime` affichait `up 37 min` → c'est le serveur physique complet qui redémarrait. Multiples marqueurs de boot dans les logs. Aucune erreur logicielle (OOM, I/O) avant les crashs → **Hard Lock** matériel.

### Cause Racine

Le flash BIOS a réinitialisé les paramètres d'usine, réactivant les profils d'économie d'énergie bureautiques incompatibles avec un serveur Bare-Metal :

- **C-States (CPU) :** veille profonde du processeur → désynchronisation KVM → panique système
- **ASPM (PCIe) :** mise en veille de la carte réseau → perte de paquets → chute des Linux Bridges

### Remédiation (BIOS — Advanced Power Management)

| Paramètre | Action | Raison |
| --- | --- | --- |
| C-States (CPU) | **Désactivé** | Processeur toujours éveillé |
| ASPM (PCIe) | **Désactivé** | Carte réseau toujours alimentée |
| SATA Power Management | **Désactivé** | Disques toujours actifs (anti-corruption) |
| Runtime Power Management | **Désactivé** | Puissance CPU maximale |

Sauvegarder via `F10` et redémarrer.

> **Leçon :** La stabilité électrique et matérielle est la condition *sine qua non* pour maintenir l'intégrité de la segmentation réseau et du filtrage pfSense. Sans une couche physique robuste, aucune politique de cybersécurité logicielle ne peut tenir.

---

## 📚 Références

- Proxmox VE Documentation : <https://pve.proxmox.com/pve-docs/>
- PVE Post-install Scripts : <https://community-scripts.github.io/ProxmoxVE/>
- Linux Bridges : <https://pve.proxmox.com/wiki/Network_Configuration>
