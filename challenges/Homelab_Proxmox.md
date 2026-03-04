# 📊 LAB : PVE

**Rôle :** Virtualisation

---

## L'intérêt technique 🎯

---

## 🛠️ Caractéristiques du Serveur

HP ProDesk 600 G4
Intel Core i7-8700T
2x16Go SK-Hynix SODIMM RAM DDR4
SSD Toshiba 512 Go
HDD Toshiba 2 To

---

> Documentation : <https://pve.proxmox.com/pve-docs/>

---

## Pré-requis

---

## Installation

---

## Correction des dépôts

Utilisation du script communautaire **PVE Post-install** : <https://community-scripts.github.io/ProxmoxVE/scripts?id=post-pve-install>

- La correction des dépôts (Repositories) : Il va désactiver le dépôt officiel "Enterprise" et il active automatiquement le dépôt gratuit "No-Subscription" pour pouvoir télécharger les paquets librement.

- La suppression de l'avertissement : Il retire le pop-up "No valid subscription" à chaque connection sur l'interface web.

- La mise à jour globale : Il se charge de télécharger et d'installer directement les dernières mises à jour de Proxmox.

Il reste un dernier dépôt à corriger (stockage **Ceph**) pour ne pas avoir d'erreurs de mise à jour : pve-server > Updates > Repositories : Disable *ceph.sources* et *pve-enterprise.sources*

![disable](/images/2026-03-02-09-16-46.png)

---

## Network

L'activation du **Trunking (802.1Q)**

Avec cette case cochée le switch virtuel se comporte comme un vrai port trunk. On pourra gérer les sous-interfaces et les VLANs directement depuis la ligne de commande des futures instances Cisco, pfSense, VyOS, etc.

![vlanaware](/images/2026-03-01-23-52-19.png)

---

## Partitionnement du stockage

On va pouvoir configurer un cloisonnement de stockage directement depuis l'interface web, de manière très visuelle. On veut utiliser le SSD pour monter nos VM/containers (et leurs snapshots), et le HDD pour stocker les ISOs, les Backups et de la Data.

### 1. Initialiser et formater le HDD

Dans la partie Disks on peut voir l'état de nos disques, il faut d'abord dire à Proxmox que ce nouveau disque mécanique existe et le préparer.

Directory > Create Directory : sélectionner le disque, système de fichier **ext4**, HDD-data, Create.

![disks](/images/2026-03-02-00-14-45.png)

### 2. Attribuer les rôles

Maintenant que le disque est prêt, on va définir ce que Proxmox a le droit de mettre sur chaque espace de stockage.

- Restreindre le HDD : Double-clic sur **HDD-Data**. Dans la liste déroulante Content on sélectionne uniquement : ISO Image, Backup, et Container Template. On retire tout le reste.

- Restreindre le SSD : Double-clic sur **local** (la racine du SSD). Dans Content, on laisse seulement Import.

Maintenant nos VM et Containers seront sur le SSD dans **local-lvm**, les ISOs, les Backup et les Datas seront sur le HDD.

---

## VM & Containers

Il faut penser à activer le **Start at boot** pour le redémarrage automatique dans les Options de chaque si besoin
