# 📺 LAB : Streaming Multimédia avec Plex en Conteneur LXC

```txt
 _______  _        _______          
(  ____ )( \      (  ____ \|\     /|
| (    )|| (      | (    \/( \   / )
| (____)|| |      | (__     \ (_) / 
|  _____)| |      |  __)     ) _ (  
| (      | |      | (       / ( ) \ 
| )      | (____/\| (____/\( /   \ )
|/       (_______/(_______/|/     \|
```

![Type](https://img.shields.io/badge/Environnement-Conteneur%20LXC-FFA500?style=flat-square&logo=linux&logoColor=white)
![OS](https://img.shields.io/badge/OS-Debian%2013%20Trixie-A81D33?style=flat-square&logo=debian&logoColor=white)
![Service](https://img.shields.io/badge/Service-Plex%20Media%20Server-E5A00D?style=flat-square&logo=plex&logoColor=white)
![Role](https://img.shields.io/badge/Rôle-Streaming%20Multimédia-005085?style=flat-square)
![Network](https://img.shields.io/badge/Réseau-DMZ-purple?style=flat-square&logo=web&logoColor=white)

**Rôle :** Administrateur d'Infrastructures Sécurisées

**Mission :** Concevoir et déployer un service de streaming multimédia performant et isolé en DMZ, en exploitant le stockage de masse de l'hyperviseur via un montage direct (Bind Mount) et en sécurisant l'accès par une micro-segmentation réseau via pfSense. Le déploiement inclut l'accélération matérielle (Intel QSV) pour le transcodage vidéo et un serveur de fichiers Samba dédié pour l'alimentation en contenu.

---

## L'intérêt technique 🎯

1. **Performance (Bind Mount) :** Le conteneur accède directement au système de fichiers de l'hôte, sans disque virtuel intermédiaire (`.qcow2`). Les performances d'écriture/lecture sont natives et l'espace disque n'est pas partitionné rigidement.
2. **Sécurité (Isolation et Moindre Privilège) :** Conteneur LXC **non privilégié** (Unprivileged) — en cas de compromission de Plex, l'attaquant ne peut pas obtenir les droits `root` sur l'hyperviseur. Le Bind Mount est configuré en **lecture seule** (`ro=1`).
3. **Ingénierie Réseau :** Le service est placé en DMZ (`10.0.0.0/24`). L'accès aux flux vidéo nécessite la traversée du routeur pfSense, imposant une maîtrise du routage inter-VLANs et du NAT (port TCP 32400).
4. **Accélération Matérielle :** Passage du GPU intégré (Intel QSV) au conteneur via device mapping, en respectant le moindre privilège (mapping GID granulaire, pas de mode privilégié).

---

## 🛠️ Architecture du Lab

- **Hôte :** Proxmox VE (`192.168.1.240`)
- **Stockage :** HDD 2 To monté sur l'hôte (`/mnt/pve/HDD-data`)
- **Nœud Applicatif :** LXC Debian 13 — Unprivileged
  - IP : `10.0.0.10` (Zone DMZ, sur `vmbr2`)
  - CPU : 2 vCPU | RAM : 2 Go | Disque : 8 Go
- **Port Applicatif :** TCP 32400
- **Passerelle :** pfSense `10.0.0.1`

---

> 📚 Documentation : <https://support.plex.tv/articles/235974187-enable-repository-updating-for-supported-linux-server-distributions/>

---

## 1️⃣ Préparation de l'Hôte (Proxmox)

Avant de déployer le conteneur, préparer le stockage et les permissions sur l'hyperviseur.

```bash
# Mise à jour de sécurité
apt update && apt dist-upgrade -y

# Création de l'arborescence de stockage (scraping TMDB/TheTVDB)
mkdir -p /mnt/pve/HDD-data/Medias/Films
mkdir -p /mnt/pve/HDD-data/Medias/Series

# Permissions pour le conteneur non privilégié
# UID 0 du LXC = UID 100000 sur l'hôte
chown -R 100000:100000 /mnt/pve/HDD-data/Medias
```

---

## 2️⃣ Création du Conteneur et Bind Mount

### Déploiement du LXC

Depuis l'interface Proxmox :

```sh
Nom : plex-server
Template : debian-13-standard
Unprivileged : ✅ (impératif)
Disque : 8 Go (SSD — OS + cache métadonnées)
CPU : 2 vCPU | RAM : 2 Go
Réseau : vmbr2 | IP : 10.0.0.10/24 | Gateway : 10.0.0.1
```

**Ne pas démarrer le conteneur.** Noter son ID (ex: `105`).

### Configuration du Bind Mount

Sur le shell de l'hôte Proxmox :

```bash
nano /etc/pve/lxc/105.conf  # Remplacer par l'ID du conteneur
```

Ajouter à la fin :

```text
mp0: /mnt/pve/HDD-data/Medias,mp=/mnt/medias_plex,ro=1
```

> ⚠️ **Moindre Privilège :** le conteneur multimédia n'a aucun droit de modification sur le stockage de l'hyperviseur (`ro=1` = lecture seule).

Démarrer le conteneur.

---

## 3️⃣ Installation de Plex (Debian 13)

L'installation respecte les normes de sécurité Debian interdisant `apt-key` au profit du répertoire `/etc/apt/keyrings`.

```bash
# Pré-requis
apt update && apt install -y curl gnupg2

# Clé GPG Plex V2 isolée dans /etc/apt/keyrings
mkdir -p /etc/apt/keyrings
curl -L https://downloads.plex.tv/plex-keys/PlexSign.v2.key \
  | gpg --yes --dearmor -o /etc/apt/keyrings/plexmediaserver.v2.gpg

# Dépôt avec directive signed-by
echo "deb [signed-by=/etc/apt/keyrings/plexmediaserver.v2.gpg] https://repo.plex.tv/deb/ public main" \
  > /etc/apt/sources.list.d/plex.list

# Installation
apt update && apt install -y plexmediaserver

# Vérification
systemctl status plexmediaserver
```

![system](/images/2026-03-08-01-29-50.png)

---

## 4️⃣ Ingénierie Réseau — Le Double Relais NAT

Plex est isolé en DMZ (`10.0.0.0/24`). Les appareils du LAN (`192.168.1.X`) ne savent pas le joindre directement. Deux niveaux de NAT sont nécessaires.

### Port Forwarding sur pfSense

Firewall > NAT > Port Forward :

| Paramètre | Valeur |
| --- | --- |
| Interface | WAN |
| Protocol | TCP |
| Destination | WAN address |
| Destination Port | 32400 |
| Redirect target IP | 10.0.0.10 |
| Redirect target port | 32400 |

pfSense génère automatiquement la règle de pare-feu associée.

### Relais NAT sur Proxmox

Sur le shell `root` de Proxmox :

```bash
# Installer iptables-persistent
apt install -y iptables-persistent

# DNAT : redirige le port 32400 vers l'IP WAN de pfSense
iptables -t nat -A PREROUTING -p tcp --dport 32400 -j DNAT --to-destination 192.168.1.251:32400

# SNAT : masque la source pour forcer le retour par Proxmox
iptables -t nat -A POSTROUTING -p tcp -d 192.168.1.251 --dport 32400 -j MASQUERADE

# Sauvegarde permanente
netfilter-persistent save
iptables-save > /etc/iptables/rules.v4
```

---

## 5️⃣ Initialisation et Sécurité Plex

### Rattachement au compte (Claim)

Plex bloque l'accès si l'administrateur ne provient pas du même sous-réseau (protection anti-usurpation).

1. Démarrer une VM située dans le réseau DMZ (`10.0.0.0/24`)
2. Ouvrir `http://10.0.0.10:32400/web` depuis cette VM
3. Se connecter avec son compte Plex et cliquer **"Réclamer le serveur" (Claim Server)**
4. Configurer les bibliothèques : `/mnt/medias_plex/Films` et `/mnt/medias_plex/Series`

L'accès définitif depuis le LAN : `http://192.168.1.240:32400/web`.

![plex](/images/2026-03-08-02-08-01.png)

### URL personnalisée (Accès distant)

Dans Paramètres > Réseau > **URL personnalisée du serveur** :

```sh
http://192.168.1.240:32400
```

Cette configuration permet à Plex de publier la bonne adresse pour le streaming hors du réseau local (relais Plex Cloud).

---

## 6️⃣ Accélération Matérielle — Intel QSV (GPU Passthrough)

### Identification du périphérique (Proxmox)

```bash
ls -l /dev/dri
# renderD128 doit apparaître — c'est l'interface de rendu matériel
# Noter le GID du groupe render (ex: 993)
```

### Montage dans le conteneur LXC

```bash
nano /etc/pve/lxc/105.conf  # ID du conteneur Plex
```

Ajouter :

```text
# Accès au GPU de rendu (moindre privilège — pas de mode privilégié)
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
```

### Mapping GID (Conteneur non privilégié)

Le GID du groupe `render` sur l'hôte doit être mappé dans le conteneur :

```bash
# Sur Proxmox : autoriser le mappage du GID render
echo "root:993:1" >> /etc/subgid

# Dans la config LXC : mapper le GID
nano /etc/pve/lxc/105.conf
```

```text
# Mappage UID standard
lxc.idmap: u 0 100000 65536
# Mappage GID avec "trou" pour le groupe render (993)
lxc.idmap: g 0 100000 992
lxc.idmap: g 992 993 1
lxc.idmap: g 993 100993 64543
```

### Activation dans Plex

```bash
# Dans le conteneur : ajouter plex aux groupes GPU
usermod -aG video,render plex
systemctl restart plexmediaserver
```

Dans l'interface web Plex : **Paramètres > Transcodeur > "Utiliser l'accélération matérielle quand disponible"** ✅

---

## 7️⃣ Serveur de Fichiers Samba (Alimentation en contenu)

Pour alimenter les bibliothèques Plex depuis un poste Windows, un conteneur Samba dédié monte le même stockage en lecture/écriture.

### Création du LXC Samba

```sh
Nom : samba-server | Unprivileged : ✅
Template : debian-13-standard
CPU : 1 vCPU | RAM : 512 Mo | Disque : 8 Go
Réseau : vmbr0 | IP : 192.168.1.242/24 | Gateway : 192.168.1.254
```

### Bind Mount en lecture/écriture

```bash
nano /etc/pve/lxc/106.conf  # ID du LXC Samba
```

```text
mp0: /mnt/pve/HDD-data/Medias,mp=/mnt/medias_share
```

> ⚠️ **Pas de `ro=1`** ici — ce conteneur doit pouvoir écrire pour ajouter du contenu. L'accès en écriture est limité au partage SMB authentifié.

### Installation et Configuration

```bash
apt update && apt install -y samba

# Créer un utilisateur dédié au partage
useradd admin
smbpasswd -a admin  # Définir un mot de passe pour l'accès SMB
```

Configuration Samba :

```bash
nano /etc/samba/smb.conf
```

```ini
[Medias]
   path = /mnt/medias_share
   writable = yes
   valid users = admin
   force user = root
```

> ⚠️ **Note sécurité :** `force user = root` permet à Samba d'écrire les fichiers avec les mêmes droits que Plex (UID 0 mappé à 100000 sur l'hôte). En contexte de production, un utilisateur dédié avec des ACL POSIX serait préférable.

```bash
systemctl restart smbd
```

### Connexion depuis Windows

1. Explorateur de fichiers → barre d'adresse : `\\192.168.1.242`
2. Saisir `admin` et le mot de passe SMB
3. Clic droit sur **Medias** → **Connecter un lecteur réseau** (ex: `Z:`)

![lecteur](/images/2026-03-08-15-55-51.png)

---

## 📦 Maintenance : Mise à jour du dépôt Plex

Suite à la refonte de l'infrastructure de distribution Plex, migration vers le nouveau dépôt :

```bash
# Script officiel de migration
curl -LsSf https://repo.plex.tv/scripts/setupRepo.sh | sudo bash

# Vérifier la communication avec le nouveau dépôt
apt update
```

---

## 📋 Résumé

- **Service :** Plex Media Server sur LXC Debian 13 (Unprivileged)
- **Réseau :** DMZ `10.0.0.10:32400`, double NAT via pfSense et Proxmox
- **Stockage :** Bind Mount lecture seule vers HDD 2 To de l'hôte
- **GPU :** Intel QSV via device mapping granulaire (pas de mode privilégié)
- **Accès contenu :** Samba sur LXC dédié `192.168.1.242` (vmbr0)
- **Sécurité :** conteneur non privilégié, Bind Mount ro, isolation DMZ, NAT filtrant

---

## 📚 Références

- Plex Support : <https://support.plex.tv/>
- Dépôt Debian : <https://support.plex.tv/articles/235974187-enable-repository-updating-for-supported-linux-server-distributions/>
- Proxmox LXC Bind Mounts : <https://pve.proxmox.com/wiki/Linux_Container#pct_mount_points>

---

![Dashboardplex](/images/2026-06-08-00-42-41.png)
