# 🗄️ LAB : Déploiement Proxmox VE (PVE)

```txt
█████▄ ▄▄▄▄   ▄▄▄  ▄▄ ▄▄ ▄▄   ▄▄  ▄▄▄  ▄▄ ▄▄ 
██▄▄█▀ ██▄█▄ ██▀██ ▀█▄█▀ ██▀▄▀██ ██▀██ ▀█▄█▀ 
██     ██ ██ ▀███▀ ██ ██ ██   ██ ▀███▀ ██ ██
```

**Rôle :** Hyperviseur Bare-Metal (Type 1) & Plateforme d'Infrastructure

**Mission :** Déployer un environnement de virtualisation polyvalent (Homelab). L'objectif est double : fournir un véritable "bac à sable" pour concevoir des topologies réseau sur-mesure (segmentation, routage, pare-feu), tout en hébergeant efficacement des machines virtuelles et des conteneurs légers pour des services internes (supervision, multimédia, automatisation).

---

## L'intérêt technique 🎯

* **Performance Bare-Metal :** Installation directe sur le matériel (Type 1) pour une allocation brute et optimisée des ressources (CPU, RAM, I/O disques), sans la contrainte ni la latence d'un système d'exploitation hôte classique.
* **Flexibilité de Déploiement (VMs & LXC) :** Capacité à instancier à la volée une grande variété de nœuds : des appliances réseau (pfSense), des machines clientes complètes, ou des conteneurs Linux ultra-légers dédiés à un service unique (microservices).
* **Maîtrise des Flux (SDN) :** Exploitation native des commutateurs virtuels (Linux Bridges) pour dessiner des architectures L2/L3 complexes, isoler des domaines de diffusion et connecter les différentes machines virtuelles entre elles.
* **Sécurité et Maquettage :** Centralisation de l'infrastructure permettant de tester, casser et reconstruire des architectures de bout en bout en toute sécurité, notamment grâce aux sauvegardes et aux instantanés (*Snapshots*).

---

## 🛠️ Caractéristiques du Serveur (Le "Homelab")

L'infrastructure repose sur un micro-serveur optimisé et mis à niveau (*upgrade processeur i3 > i7, RAM 8go > 32go, HDD 256go > SSD 512go + HDD 2to*) pour garantir une réserve de puissance suffisante lors du maquettage de topologies complexes :

* **Châssis :** HP ProDesk 600 G4 (Format Mini-PC, silencieux et basse consommation).
* **Processeur (CPU) :** Intel Core i7-8700T (6 cœurs / 12 threads) – Parfait pour paralléliser les charges des différentes appliances réseau.
* **Mémoire (RAM) :** 32 Go (2x16 Go SK-Hynix SODIMM DDR4) – Marge confortable pour faire tourner simultanément de multiples environnements virtuels isolés.
* **Stockage Système (OS/VMs) :** SSD Toshiba 512 Go – Assure des débits I/O rapides pour le système hôte et les machines virtuelles.
* **Stockage Données (Data) :** HDD Toshiba 2 To – Espace haute capacité dédié au stockage des images ISO, aux sauvegardes (Dumps) et aux futurs montages de volumes pour les services annexes.

💰 *Budget : 300€* - Il fallait respecter un budget modeste malgré l'augmentation indécente du prix de la mémoire vive (Février 2026).

![prodesk](/images/2026-03-05-01-10-17.png)

---

> Documentation officielle : [https://pve.proxmox.com/pve-docs/](https://pve.proxmox.com/pve-docs/)

---

## Pré-requis 📋

* Une clé USB (minimum 8 Go) vierge.
* L'image ISO officielle de Proxmox VE (téléchargeable sur le site officiel).
* Un utilitaire de flashage bootable (comme BalenaEtcher ou Rufus).
* Une connexion réseau filaire (câble RJ45) reliant directement le mini-PC au commutateur/routeur du réseau domestique (indispensable pour l'interface de management).

---

## Installation (Bare-Metal)

1. **Préparation du support :** Flashage de l'image ISO de Proxmox VE sur la clé USB à l'aide de l'utilitaire choisi.
2. **Séquence de démarrage :** Insertion de la clé USB dans le mini-PC HP et modification de l'ordre de boot dans le BIOS (généralement touche F9 ou F10) pour démarrer sur le support d'installation.
3. **Lancement de l'installeur :** Sélection de l'option "Install Proxmox VE (Graphical)" sur le premier écran de démarrage.
4. **Ciblage du stockage :** Sélection rigoureuse du **SSD de 512 Go** comme disque cible pour l'installation du système (Attention à ne pas sélectionner le HDD de 2 To dédié aux données).
5. **Configuration Réseau initiale (L'étape critique) :** * Choix de l'interface réseau physique (`eno1` ou `nic0`).
    * Définition d'un nom d'hôte qualifié (FQDN), par exemple `pve-server.lan`.
    * Attribution d'une adresse IP statique de management (ex: `192.168.1.240/24`) et renseignement de la passerelle par défaut. *C'est cette étape qui crée automatiquement le premier pont virtuel `vmbr0`.*

6. **Sécurité :** Définition du mot de passe super-administrateur (`root`) et d'une adresse email de contact pour les alertes système.
7. **Finalisation :** Redémarrage automatique du serveur. Le système affiche alors l'invite de commande confirmant que l'interface d'administration web est accessible depuis n'importe quel poste du réseau LAN via l'URL : `https://[IP_DU_SERVEUR]:8006`.

---

## ⚙️ Configuration Post-Installation (Optimisation du socle)

Une fois l'hyperviseur installé, il est nécessaire de préparer l'environnement système avant de concevoir la topologie réseau.

### Fiabilisation des dépôts (Repositories)

Par défaut, Proxmox VE est configuré pour interroger les dépôts "Enterprise", nécessitant une souscription payante, ce qui génère des erreurs lors des mises à jour du système hôte. L'objectif est de basculer sur les dépôts "No-Subscription" pour les environnements de Lab.

**Automatisation via script (Proxmox VE Helper Scripts) :**
Plutôt que d'éditer manuellement les sources de paquets Linux, exécution d'un script utilitaire reconnu par la communauté pour nettoyer le système et appliquer les mises à jour de base :

1. Ouverture du "Shell" depuis l'interface web de l'hyperviseur.
2. Exécution de la commande : `bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/misc/post-pve-install.sh)"`
3. Choix des options : Désactivation du dépôt *Enterprise*, activation du dépôt *No-Subscription*, et application des mises à jour (`apt update && apt upgrade`).

    **PVE Post-install** : <https://community-scripts.github.io/ProxmoxVE/scripts?id=post-pve-install>
    * La correction des dépôts (Repositories) : Il va désactiver le dépôt officiel "Enterprise" et il active automatiquement le dépôt gratuit "No-Subscription" pour pouvoir télécharger les paquets librement.
    * La suppression de l'avertissement : Il retire le pop-up "No valid subscription" à chaque connection sur l'interface web.
    * La mise à jour globale : Il se charge de télécharger et d'installer directement les dernières mises à jour de Proxmox.

4. Il reste un dernier dépôt à corriger (stockage **Ceph**) pour ne pas avoir d'erreurs de mise à jour : pve-server > Updates > Repositories : Disable *ceph.sources* et *pve-enterprise.sources*

![disable](/images/2026-03-02-09-16-46.png)

---

## 🕸️ Conception de l'Architecture Réseau Virtuelle (SDN)

C'est le cœur de l'infrastructure. Proxmox ne gérant nativement que les ponts virtuels (Linux Bridges), l'objectif est de créer des commutateurs virtuels (Virtual Switches) pour isoler les futurs domaines de diffusion (Layer 2) avant d'y injecter des appliances de routage (Layer 3).

Configuration réalisée via le menu : **PVE-Server > System > Network**.

### Les Commutateurs Virtuels (Bridges)

* 🌉 **vmbr0 (Uplink / "WAN") :**
  * **Rôle :** Pont physique lié à la carte réseau réelle (`eno1` ou `nic0`).
  * **Adressage :** Porte l'IP de management du serveur sur le réseau domestique (ex: `192.168.1.240/24`) ainsi que sa passerelle par défaut.
  * **Fonction :** Permet la sortie vers l'internet physique.

Pour les autres voir le [Lab déploiement d'un Routeur/Pare-feu pfSense](./Homelab_pfSense.md)

### Le Trunking

L'activation du **Trunking (802.1Q)**

Avec cette case cochée le switch virtuel se comporte comme un vrai port trunk. On pourra gérer les sous-interfaces et les VLANs directement depuis la ligne de commande des futures instances Cisco, pfSense, VyOS, etc.

![vlanaware](/images/2026-03-01-23-52-19.png)

---

## Partitionnement du stockage 💾

On va pouvoir configurer un cloisonnement de stockage directement depuis l'interface web, de manière très visuelle. On veut utiliser le SSD pour monter nos VM/containers (et leurs snapshots), et le HDD pour stocker les ISOs, les Backups et de la Data.

### 1. Initialiser et formater le HDD

Dans la partie Disks on peut voir l'état de nos disques, il faut d'abord dire à Proxmox que ce nouveau disque mécanique existe et le préparer.

Directory > Create Directory : sélectionner le disque, système de fichier **ext4**, HDD-data, Create.

![disks](/images/2026-03-02-00-14-45.png)

### 2. Attribuer les rôles

Maintenant que le disque est prêt, on va définir ce que Proxmox a le droit de mettre sur chaque espace de stockage.

* Restreindre le HDD : Double-clic sur **HDD-Data**. Dans la liste déroulante Content on sélectionne uniquement : ISO Image, Backup, et Container Template. On retire tout le reste.

* Restreindre le SSD : Double-clic sur **local** (la racine du SSD). Dans Content, on laisse seulement Import.

Maintenant nos VM et Containers seront sur le SSD dans **local-lvm**, les ISOs, les Backup et les Datas seront sur le HDD.

---

## VM & Containers 🖧

Il faut penser à activer le **Start at boot** pour le redémarrage automatique dans les options de chaque machine si besoin.

* Réglages VM général :

  * Activer SSD **Emulation**
  * Activer **Discard**
  * **VirtIO** pour des VM Linux, **Intel E1000** pour des VM Windows

* Réglages container LXC :

  * Laisse Unprivileged container coché : approche sécuritaire par conception, car elle empêche les attaquants d’obtenir un accès root à l’hôte même en échappant au conteneur.
  * au démarrage pour autoriser SSH via mdp ⚠️ *Ne pas laisser en prod* :

  ```sh
  sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin yes/' /etc/ssh/sshd_config
  ```

💡 Pour supprimer une ancienne clef SSH d’une IP de VM :

```sh
ssh-keygen -R 192.168.1.x
```

---

## Correction du NTP (Network Time protocol)

Checkmk WARN sur le NTP Time du PVE-Server : "Stratum 4" avec un décalage (Offset), cela signifie que Proxmox interroge un serveur de temps lui-même assez éloigné de la source atomique (Stratum 1), ce qui génère de la latence.

![NTP error](/images/2026-03-05-00-07-48.png)

### Modifier les sources de temps

Ouvrir le fichier de configuration avec l'éditeur de texte :

```Bash
nano /etc/chrony/chrony.conf
```

### Ajouter les serveurs locaux

Repèrer les lignes qui commencent par pool (généralement `pool 2.debian.pool.ntp.org iburst`).

Les commenter et ajouter les serveurs français juste en dessous pour obtenir ceci :

```Plaintext
#pool 2.debian.pool.ntp.org iburst
pool 0.fr.pool.ntp.org iburst
pool 1.fr.pool.ntp.org iburst
pool 2.fr.pool.ntp.org iburst
```

### Relancer le service

Appliquer les modifications en redémarrant le démon NTP :

```Bash
systemctl restart chrony
```

### Vérifier la synchronisation

Pour confirmer que l'hyperviseur a bien accroché les nouveaux serveurs, lancer :

```Bash
chronyc sources -v
```

![NTP](/images/2026-03-05-00-12-49.png)

---
