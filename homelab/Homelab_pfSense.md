# 🌐 LAB : Déploiement d'un Routeur/Pare-feu pfSense sous Proxmox

```txt
        __
 _ __  / _|___  ___ _ __  ___  ___
| '_ \| |_/ __|/ _ \ '_ \/ __|/ _ \
| |_) |  _\__ \  __/ | | \__ \  __/
| .__/|_| |___/\___|_| |_|___/\___|
|_|
```

![Type](https://img.shields.io/badge/Environnement-Machine%20Virtuelle-005085?style=flat-square&logo=virtualbox&logoColor=white)
![OS](https://img.shields.io/badge/OS-pfSense%20CE%202.7.2-A81D33?style=flat-square&logo=pfsense&logoColor=white)
![Role](https://img.shields.io/badge/Rôle-Routeur%20%2F%20Pare--feu-00B32C?style=flat-square)
![VPN](https://img.shields.io/badge/Service-OpenVPN%20(UDP%201194)-EA7E20?style=flat-square&logo=openvpn&logoColor=white)
![Network](https://img.shields.io/badge/Réseau-LAN%20↔%20DMZ-purple?style=flat-square&logo=web&logoColor=white)

**Rôle :** Ingénieur Réseau / Administrateur d'Infrastructures Sécurisées

**Mission :** Concevoir et déployer une architecture réseau segmentée et virtualisée. L'objectif est d'isoler les services applicatifs (DMZ) du réseau de confiance (LAN) au travers d'une appliance de routage professionnelle (pfSense), en configurant le routage, le NAT, le filtrage de flux, les services DHCP et les tunnels VPN chiffrés (OpenVPN).

> 📚 Documentation : <https://docs.netgate.com/pfsense/en/latest/config/>

---

## L'intérêt technique 🎯

1. **Séparation des couches (L2/L3) :** Exploiter les ponts virtuels (Linux Bridges) pour créer des topologies propres — `vmbr0` pour le LAN physique, `vmbr2` pour la DMZ isolée.
2. **Maîtrise du NAT et du filtrage :** Configurer le masquage d'adresses (MASQUERADE) pour la sortie internet et la redirection de ports (DNAT) pour les services exposés (Plex, VPN).
3. **Déploiement d'Appliance Réseau :** Manipuler pfSense, un standard de l'industrie, pour implémenter des politiques de sécurité et des règles de pare-feu aux frontières de l'infrastructure.
4. **Tunnels VPN :** Implémenter une infrastructure à clés publiques (CA/Certificats) et monter un tunnel OpenVPN (interface *tun*) pour acheminer le trafic de clients distants vers le sous-réseau privé.

---

## 🛠️ Architecture du Lab (Actuelle)

- **Environnement :** Serveur Proxmox VE (Hyperviseur Bare-Metal)
- **Topologie Réseau :**

| Bridge | Rôle | Adressage |
| --- | --- | --- |
| `vmbr0` | LAN physique (Uplink WAN) | `192.168.1.0/24` |
| `vmbr2` | DMZ isolée (Switch virtuel pur) | `10.0.0.0/24` |

- **pfSense (VM) :**
  - Interface WAN : `192.168.1.251` (sur `vmbr0` — directement sur le LAN physique)
  - Interface LAN : `10.0.0.1` (sur `vmbr2` — passerelle DMZ)
  - Cartes réseau paravirtualisées (*VirtIO*)
  - Services : routage inter-VLANs, NAT, DHCP (DMZ), DNS Forwarder, OpenVPN, [CrowdSec](./Homelab_Crowdsec.md)

---

## 1️⃣ Création de la VM pfSense

Depuis l'interface Proxmox → **Créer une VM** :

```sh
ID : 1000
Nom : pfSense
ISO : pfSense 2.7.2
Type : Other (pas "Linux")
Disque : 20 Go | SSD Emulation : ✅ | Discard : ✅
RAM : 2048 Mo
CPU : 1
Pont : vmbr0 (⚠️ interface WAN)
Pare-feu Proxmox : décoché (⚠️ pfSense gère son propre filtrage)
Model : VirtIO
```

**Avant de démarrer**, ajouter une seconde interface réseau : **Matériel > Ajouter > Carte réseau** → Pont `vmbr2`, Pare-feu décoché.

### Installation

Démarrer la VM → Console. L'installation est guidée, valider par défaut sauf :

- ⚠️ Lors du choix du disque dur, appuyer sur **Espace** pour cocher avant **Entrée**

Une fois terminée, éjecter l'ISO : **Matériel > CD/DVD > N'utiliser aucun média**.

![interface](/images/2026-03-04-01-10-14.png)

---

## 2️⃣ Configuration Réseau pfSense

Au premier démarrage, pfSense demande de préciser les interfaces :

- VLANs : `no`
- WAN : `vtnet0`
- LAN : `vtnet1`

Passer le clavier en Azerty : `8) Shell` → `kbdcontrol -l /usr/share/vt/keymaps/fr.kbd` → `exit`.

### Interface WAN (`192.168.1.251`)

`2) Set interface(s) IP address` → `1` (WAN) :

```txt
DHCP? : n
IPv4 address : 192.168.1.251
Subnet : 24
Gateway : 192.168.1.254 (Box FAI)
IPv6 DHCP? : n
IPv6 address : (vide)
DHCP server on WAN : n
Revert to HTTP : n
```

Vérifier l'accès Internet : `7) Ping host` → `8.8.8.8`

### Interface LAN / DMZ (`10.0.0.1`)

`2) Set interface(s) IP address` → `2` (LAN) :

```txt
DHCP? : n
IPv4 address : 10.0.0.1
Subnet : 24
Gateway : (vide)
IPv6 : (vide)
DHCP server : y
Start : 10.0.0.50
End : 10.0.0.150
Revert to HTTP : n
```

![interfaces](/images/2026-03-04-01-26-12.png)

---

## 3️⃣ Configuration Initiale (Wizard)

Depuis une VM du réseau DMZ (`vmbr2`), ouvrir `https://10.0.0.1` dans le navigateur.

![login](/images/2026-02-26-18-25-45.png)

Connexion avec `admin` / `pfsense`, puis suivre le Wizard :

```txt
- Hostname : pfSense
- Domain : pve-server.lan
- Primary DNS : 192.168.1.250 (AdGuard Home)
- Secondary DNS : 9.9.9.9
- Override DNS : décocher
- Timezone : Europe/Paris
- ⚠️ Décocher "Block RFC1918 Private Networks" (sinon le trafic LAN est bloqué)
- Admin password : mot de passe fort
```

`Reload` pour appliquer.

> 💡 Si problème de DNS après le wizard : **Services > DNS Resolver** → relancer le service (🔄).
>
> 💡 Message d'alerte ISC DHCP → **System > Advanced > Networking** → activer `Kea DHCP`.

---

## 4️⃣ Redirection de Ports (Box FAI → pfSense)

La Box FAI redirige les ports directement vers pfSense (`192.168.1.251`) :

| Port | Protocole | Service |
| --- | --- | --- |
| 1194 | UDP | OpenVPN |
| 32400 | TCP | Plex (relayé vers `10.0.0.10` par pfSense) |

Configuration dans l'interface d'administration de la Box FAI → **NAT / Port Forwarding**.

---

## 5️⃣ Serveur OpenVPN

### Création du serveur VPN

Dans `VPN > OpenVPN` → onglet `Wizards` → `Local User Access` :

```txt
CA :
  - Descriptive name : vpn
  - Common Name : vpn

Certificat serveur :
  - Descriptive name : vpn-cert-server
  - Common Name : vpn-cert-server

Serveur OpenVPN :
  - Description : vpn-remote-access
  - IPv4 Tunnel Network : 10.42.0.0/24
  - IPv4 Local Network : 10.0.0.0/24

Firewall :
  - Firewall Rule : ✅
  - OpenVPN rule : ✅
```

### Utilisateur et certificat client

Dans `System > User Manager` → `Add` :

```txt
Username : user
Password : mot de passe fort
Certificate : ✅
  - Descriptive name : user-vpn-cert
```

Vérifier dans `System > Certificates` → onglet `Certificates` : 2 certificats (serveur + utilisateur).

### Export de la configuration client

Dans `System > Package Manager > Available Packages` → installer `openvpn-client-export`.

Puis `VPN > OpenVPN` → onglet `Client Export` :

- **Host Name Resolution :** remplacer `Interface IP Address` par `Other`
- **Host Name :** saisir l'adresse IPv4 publique du serveur (`<IP_PUBLIQUE>`)

Télécharger le fichier **Inline Configurations** → `Most Clients` ou `OpenVPN Connect (iOS/Android)`.

### Connexion au VPN

Installer [OpenVPN Connect](https://openvpn.net/client/) → `Upload File` → charger le fichier de configuration → `Connect` avec les identifiants créés.

![vpn](/images/2026-03-04-02-33-53.png)

Accéder à `https://10.0.0.1/` depuis le navigateur → si la page pfSense s'affiche, le VPN fonctionne 🎉

---

## 6️⃣ Keep Alive (Maintien des connexions)

### OpenVPN

Dans **VPN > OpenVPN > Servers** → éditer le serveur → **Ping settings** :

| Paramètre | Valeur |
| --- | --- |
| Inactivity Timeout | 600 |
| Ping interval | 10 |
| Ping timeout | 60 |

### Firewall States

Dans **System > Advanced > Firewall & NAT** → **Firewall Optimization Options** → sélectionner **Conservative**.

Évite que pfSense ne coupe les connexions SSH silencieuses (vers Proxmox par exemple).

---

## 🎯 Validation de l'Architecture

- **Isolation L2 et Distribution L3 :** Le domaine de diffusion DMZ (`vmbr2`) est étanche. pfSense agit comme passerelle exclusive (`10.0.0.1/24`) avec DHCP.
- **Résolution DNS :** Intégration d'[AdGuard Home](./Homelab_AdGuard.md) (`192.168.1.250`) + [Unbound](./Homelab_Unbound.md) (résolveur récursif). pfSense agit comme relais DNS Forwarder.
- **Accès Distant (VPN) :** Tunnel chiffré L3 OpenVPN (interface *tun*). Flux bidirectionnel entre le réseau tunnel (`10.42.0.0/24`) et la DMZ (`10.0.0.0/24`).

## 🔄 Analyse de Flux : Paquet VPN entrant

```txt
1. Client externe → <IP_PUBLIQUE>:1194 (UDP)
2. Box FAI → Port Forward vers 192.168.1.251:1194 (pfSense WAN)
3. pfSense → Validation PKI, décapsulation OpenVPN
4. pfSense → Routage vers 10.0.0.0/24 (interface LAN/DMZ sur vmbr2)
```

---

## 📝 Note historique : Évolution de l'architecture

> L'architecture initiale utilisait un pont de transit (`vmbr1`, réseau `192.168.10.0/24`) entre Proxmox et pfSense. L'hyperviseur agissait comme routeur intermédiaire avec des règles NAT/iptables, créant un **double NAT** (Proxmox + pfSense).
>
> Cette topologie a été **simplifiée** en rattachant directement l'interface WAN de pfSense au réseau LAN physique (`vmbr0`, IP `192.168.1.251`). Le pont `vmbr1` a été supprimé, les règles iptables de transit purgées, et les redirections de ports déplacées vers la Box FAI.
>
> **Bénéfices :** élimination du double NAT, réduction de la charge CPU sur l'hyperviseur, simplification du troubleshooting, et chaîne de routage plus lisible.
>
> Cette évolution illustre le principe d'**Efficience** : moins il y a de nœuds de translation entre Internet et le pare-feu, plus le réseau gagne en robustesse.

---

## 📚 Références

- pfSense Documentation : <https://docs.netgate.com/pfsense/en/latest/config/>
- OpenVPN Connect : <https://openvpn.net/client/>
