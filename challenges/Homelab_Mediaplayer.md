# 📺 LAB : Déploiement d'un Serveur Multimédia (Plex) en Conteneur LXC avec Bind Mount

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

**Rôle :** Administrateur d'Infrastructures Sécurisées / Ingénieur DevOps

**Mission :** Concevoir et déployer un service de streaming multimédia performant et isolé, en s'affranchissant de la lourdeur d'un système NAS dédié, tout en exploitant le stockage de masse de l'hyperviseur de manière granulaire et sécurisé par une micro-segmentation réseau (DMZ) via pfSense.

---

## Intérêt Technique 🎯

* **Administration Système (Modularité et Performance) :** L'utilisation de la technologie **Bind Mount** permet au conteneur d'accéder directement au système de fichiers de l'hôte. Il n'y a aucune création de disque virtuel intermédiaire (`.raw` ou `.qcow2`), ce qui garantit des performances d'écriture/lecture natives. L'espace n'est pas partitionné rigidement : Plex utilise l'espace disponible sur les 2 To de manière dynamique, sans bloquer les autres services (comme les sauvegardes).
* **Sécurité by Design (Isolation et Moindre Privilège) :** Le déploiement s'effectue via un **conteneur LXC non privilégié (Unprivileged)**. En cas de compromission de l'application Plex (via une faille Zero-Day par exemple), l'attaquant ne pourra pas obtenir les droits `root` sur l'hyperviseur Proxmox.
* **Ingénierie Réseau :** Le service est placé dans une zone réseau isolée (`DMZ`). L'accès aux flux vidéo nécessite la traversée du routeur (pfSense), imposant une maîtrise du routage inter-VLANs et de la gestion des ports applicatifs (TCP 32400).
* **Sécurité des Paquets (Debian 13)** : Implémentation des standards modernes de gestion de clés GPG (`/etc/apt/keyrings`) pour garantir l'intégrité des paquets logiciels tiers.

---

## Architecture Lab

* **Hôte :** Hyperviseur Proxmox VE (IP : `192.168.1.240`).
* **Stockage Hôte :** Disque dur de 2 To monté localement (ex: `/mnt/pve/HDD_2To`).
* **Nœud Applicatif :** Conteneur LXC Debian 13 (Unprivileged).
* **Zone Réseau (LAN pfSense):** Segment isolé DMZ (`10.0.0.0/24`). IP statique cible : `10.0.0.10`.
* **Passerelle (WAN pfSense)** : Interface connectée au réseau de l'hyperviseur (192.168.10.254)
* **Flux Applicatif :** Port d'écoute Plex Media Server (`32400` TCP).

---

> 📚 Documentation :
>
> Ajout d'un repo Plex : <https://support.plex.tv/articles/235974187-enable-repository-updating-for-supported-linux-server-distributions/>

---

## ✅ Pré-requis et Préparation (Hyperviseur)

Avant de déployer le conteneur, il faut préparer l'hôte physique. Se connecter en `root` sur le shell de Proxmox.

**1. Mise à jour de sécurité de l'hyperviseur :**
On utilise `dist-upgrade` pour gérer intelligemment les changements de dépendances du noyau.

```bash
apt update && apt dist-upgrade -y

```

**2. Création de l'arborescence de stockage :**
Plex nécessite des répertoires séparés pour appliquer les bons agents d'indexation (Scrapers : TMDB pour les films, TheTVDB pour les séries).

```bash
mkdir -p /mnt/pve/HDD-data/Medias/Films
mkdir -p /mnt/pve/HDD-data/Medias/Series

```

**3. Gestion des permissions (Le Mappage UID) :**
Le conteneur sera "Unprivileged". L'utilisateur `root` (UID 0) à l'intérieur du conteneur correspond à l'utilisateur `100000` sur l'hôte. Il faut lui donner la propriété du dossier.

```bash
chown -R 100000:100000 /mnt/pve/HDD-data/Medias

```

---

## 🗂️ Phase 1 : Création du Conteneur et Bind Mount

Le déploiement du nœud applicatif s'effectue depuis l'interface graphique de Proxmox.

**1. Déploiement du LXC :**

* **General :** Nommer le conteneur (ex: `plex-server`) et **laisser impérativement la case "Unprivileged container" cochée**.
* **Template :** `debian-13-standard`.
* **Disque :** 8 Go sur le stockage local (SSD) pour l'OS et le cache des métadonnées.
* **CPU / RAM :** 2 vCPUs et 2 Go de RAM (Configuration optimale pour de la lecture directe sans transcodage matériel 4K).
* **Network :** Attacher au pont isolé `vmbr2`. Assigner l'IP `10.0.0.10/24` avec la passerelle `10.0.0.1` (pfSense).
* **Ne pas démarrer le conteneur.** Noter son ID (ex: `105`).

**2. Configuration du Bind Mount :**
Retourner sur le shell de l'hôte Proxmox pour lier le dossier physique au conteneur.

```bash
nano /etc/pve/lxc/105.conf # Remplacer 10X par l'ID de votre conteneur

```

Ajouter cette ligne à la fin du fichier :

```text
mp0: /mnt/pve/HDD-data/Medias,mp=/mnt/medias_plex,ro=1

```

*(Principe de Moindre Privilège : Le conteneur multimédia ne doit avoir aucun droit de modification ou de suppression sur le stockage de masse de l'hyperviseur).*

**Démarrer maintenant le conteneur.**

---

## 💾 Phase 2 : Installation Applicative (Debian 13 Stable)

Ouvrir la console du conteneur LXC (en `root`). L'installation respecte les nouvelles normes de sécurité Debian interdisant l'usage de `apt-key`.

```bash
# 1. Mise à jour et installation des pré-requis
apt update && apt install -y curl gnupg2

# 2. Création du répertoire sécurisé pour les clés GPG isolées
mkdir -p /etc/apt/keyrings

# 3. Téléchargement et conversion binaire de la clé Plex V2 officielle
curl -L https://downloads.plex.tv/plex-keys/PlexSign.v2.key | gpg --yes --dearmor -o /etc/apt/keyrings/plexmediaserver.v2.gpg

# 4. Ajout du dépôt avec la directive "signed-by" pour lier la clé au dépôt
echo "deb [signed-by=/etc/apt/keyrings/plexmediaserver.v2.gpg] https://repo.plex.tv/deb/ public main" > /etc/apt/sources.list.d/plex.list

# 5. Installation du serveur multimédia
apt update && apt install -y plexmediaserver

# 6. Vérification du démarrage du service
systemctl status plexmediaserver

```

![system](/images/2026-03-08-01-29-50.png)

---

## 🌐 Phase 3 : Ingénierie Réseau et Routage (Le double relais)

Plex est isolé dans le réseau `10.0.0.0/24`. Les appareils du domicile (`192.168.1.X`) ne savent pas comment le joindre, et la Box Internet ne gère pas les routes statiques. Nous allons faire de Proxmox la porte d'entrée universelle.

### Étape 1 : Le Port Forwarding sur pfSense (L'accès DMZ)

1. Sur l'interface web de pfSense, aller dans **Firewall > NAT > Port Forward**.
2. Cliquer sur Add :
    * **Interface :** `WAN` / **Protocol :** `TCP`
    * **Destination :** `WAN address`
    * **Destination Port Range :** `32400`
    * **Redirect target IP :** `10.0.0.10` (IP de Plex)
    * **Redirect target port :** `32400`
    * **Description :** `Redirection Plex`

3. Sauvegarder et appliquer. *(pfSense génère automatiquement la règle de pare-feu associée).*

### Étape 2 : Le Relais NAT sur Proxmox (La persistance)

Sur le shell `root` de Proxmox, nous redirigeons le trafic arrivant sur l'hyperviseur vers pfSense.

```bash
# Installation de l'outil de persistance iptables
apt update && apt install iptables-persistent -y

# Règle 1 (DNAT) : Redirige le port 32400 vers l'IP WAN de pfSense
iptables -t nat -A PREROUTING -p tcp --dport 32400 -j DNAT --to-destination 192.168.10.254:32400

# Règle 2 (SNAT) : Masque la source pour forcer le retour par Proxmox
iptables -t nat -A POSTROUTING -p tcp -d 192.168.10.254 --dport 32400 -j MASQUERADE

# Sauvegarde permanente des règles
netfilter-persistent save

# Pour être certain de sauvegarder l'iptable
iptables-save > /etc/iptables/rules.v4

```

---

## 🔗 Phase 4 : Initialisation, Sécurité et optimisation Plex

### Étape 1 : Rattachement au compte (Le "Claim")

Lors de la première connexion, Plex bloque l'accès si l'administrateur ne provient pas exactement du même sous-réseau (Protection contre l'usurpation).

*La solution élégante (Bypass de sécurité local) :*

1. Démarrer une **Machine Virtuelle située dans le réseau isolé** (10.0.0.0/24)
2. Ouvrir le navigateur de cette VM et se rendre sur : `http://10.0.0.10:32400/web`.
3. Plex reconnaîtra une connexion locale légitime. Se connecter avec son compte en ligne.
4. Cliquer sur **"Réclamer le serveur" (Claim Server)**.
5. *(Optionnel)* Configurer les bibliothèques en pointant vers `/mnt/medias_plex/Films` et `/mnt/medias_plex/Series`.

Le serveur est désormais lié à votre compte. L'accès définitif depuis n'importe quel appareil du réseau domestique sera : `http://192.168.1.240:32400/web`.

![plex](/images/2026-03-08-02-08-01.png)

### Étape 2 : Optimisation du Routage (URL Personnalisée)

Pour informer Plex de l'architecture spécifique (DMZ derrière pfSense) et éviter qu'il ne considère le réseau local domestique comme externe :

1. Aller dans **Réglages > [Nom du Serveur] > Réseau**.
2. S'assurer que les paramètres avancés sont affichés (Bouton orange).
3. Dans le champ **URL personnalisées pour accéder au serveur**, renseigner l'adresse de rebond (Proxmox) :
👉 `http://192.168.1.240:32400`
4. *Résultat : Plex indique cette route précise aux applications clientes (TV, PC) pour un accès direct à pleine vitesse.*

![claimed](/images/2026-03-08-13-56-46.png)

### Étape 3 : Désactivation du Relais (Bypass du Bridage)

Pour empêcher Plex d'utiliser ses propres serveurs (Plex Relay) en cas de doute sur le routage (ce qui bride la qualité à 2 Mbps) :

1. Toujours dans **Réglages > Réseau**.
2. Chercher la case **Activer le relais** (Enable Relay).
3. **Décocher** impérativement cette case.
4. Enregistrer les modifications en bas de page.

### Étape 4 : Qualité Originale sur les Clients

Sur chaque application cliente (Smart TV, Apple TV, etc.) :

1. Aller dans les **Paramètres** de l'application.
2. Section **Qualité Vidéo**.
3. Régler **Qualité Locale** et **Qualité Distante** sur **Maximale** (ou Originale).

---

## 📡 Phase 5 : Ingénierie Réseau (Relais Multicast / SSDP)

Par défaut, pfSense bloque les trames de diffusion (Broadcast/Multicast) entre ses interfaces WAN (réseau domestique `192.168.1.X`) et LAN (zone isolée `10.0.0.X`). Pour qu'une Smart TV découvre Plex, il faut implémenter un relais UDP pour le protocole **SSDP (Simple Service Discovery Protocol)**.

**Déploiement du relais sur pfSense :**

1. **Installation :** Naviguer dans le menu **System > Package Manager > Available Packages**. Rechercher et installer le paquet `udpbroadcastrelay`.

2. **Configuration :** Se rendre dans **Services > UDP Broadcast Relay**.

3. Cliquer sur **Add** pour créer une nouvelle instance avec les paramètres stricts suivants :

   * **Description :** `Relais SSDP Plex`
   * **Port :** `1900` *(Le port standard utilisé par le protocole SSDP)*
   * **Interfaces :** Sélectionner simultanément les interfaces `WAN` et `LAN` (utiliser la touche `Ctrl` pour une sélection multiple).
   * **Multicast Address :** `239.255.255.250` *(L'adresse de diffusion standard de SSDP)*
   * **Source Address :** Laisser vide (ou spécifier l'IP du serveur Plex `10.0.0.10` pour plus de sécurité).

4. Cliquer sur **Save** et s'assurer que le service est démarré.

*[Note de sécurité : Ce paquet agit au niveau du noyau (Kernel) pour dupliquer les trames UDP d'une interface à l'autre, contournant la limitation de la couche 3]*

---

## 🛡️ Phase 6 : Sécurisation des Flux de Découverte (Pare-feu)

Le relais duplique les trames, mais le moteur du pare-feu doit être explicitement autorisé à les laisser passer pour respecter la politique de blocage par défaut (*Default Deny*).

Il est nécessaire de créer deux règles d'autorisation chirurgicales :

1. **Sur l'interface WAN (Pour les requêtes de la Smart TV) :**

   * **Action :** `Pass`
   * **Protocol :** `UDP`
   * **Source :** `Réseau domestique`  (ou l'IP fixe de la TV si connue).
   * **Destination :** `Network` -> `239.255.255.250`
   * **Destination Port :** `1900`

2. **Sur l'interface LAN (Pour les réponses du conteneur Plex) :**

   * **Action :** `Pass`
   * **Protocol :** `UDP`
   * **Source :** `10.0.0.10` (IP du LXC Plex)
   * **Destination :** `Network` -> `239.255.255.250`
   * **Destination Port :** `1900`

Une fois ces règles appliquées, l'application Plex apparaîtra automatiquement sur les écrans connectés du réseau principal, tout en restant confinée en toute sécurité derrière le routeur virtuel.

---

## 📁 Phase 7 : Mise en place du partage réseau (Alimentation de la bibliothèque)

Pour envoyer les fichiers multimédias depuis le poste de travail Windows vers le stockage de l'hyperviseur, deux méthodes sont envisageables selon le besoin de transparence pour l'utilisateur.

### Option A : La méthode "Sysadmin" (WinSCP / SFTP)

Idéale pour des transferts ponctuels, sans aucune configuration supplémentaire requise sur le serveur. Elle exploite le protocole SSH natif de l'hyperviseur.

1. Télécharger et installer le client **WinSCP** sur le poste Windows.
2. Créer une nouvelle session avec les paramètres suivants :
      * **Protocole :** `SFTP`
      * **Hôte :** `192.168.1.240` *(IP de Proxmox)*
      * **Port :** `22`
      * **Identifiants :** `root` et le mot de passe de l'hyperviseur.

3. Dans l'interface, naviguer à droite vers le dossier cible : `/mnt/pve/HDD-data/Medias/`.
4. Glisser-déposer les fichiers depuis le panneau gauche (Windows) vers le panneau droit (Serveur).

---

### Option B : La méthode "Transparente" (Serveur Samba / SMB)

Idéale pour un usage quotidien. Elle permet de monter le dossier du serveur comme un véritable disque dur (Disque Z:) directement dans l'Explorateur Windows.

*Dans le respect des bonnes pratiques, nous déployons ce service dans un conteneur LXC dédié (Micro-service) pour ne pas polluer l'hôte Proxmox.*

#### 1. Création du LXC "Serveur de Fichiers"

Depuis l'interface Proxmox, créer un nouveau conteneur :

* **Général :** Nom `samba-server`, cocher **Unprivileged container**.
* **Template :** `debian-13-standard`.
* **Ressources :** Très léger (1 vCPU, 512 Mo de RAM, 8 Go de disque).
* **Réseau :** Pour simplifier la découverte par Windows, l'attacher au pont classique **`vmbr0`** (le réseau de la box) avec une IP statique comme `192.168.1.242/24` et la passerelle de la box.
* *Ne pas démarrer le conteneur. Noter son ID (ex: 106).*

#### 2. Le Bind Mount (en Lecture/Écriture)

Contrairement à Plex, ce conteneur doit avoir le droit de modifier les fichiers.
Sur le Shell `root` de Proxmox :

```bash
nano /etc/pve/lxc/10X.conf # Remplacer 10X par l'ID du LXC Samba

```

Ajouter la ligne de montage (sans le `ro=1`) :

```text
mp0: /mnt/pve/HDD-data/Medias,mp=/mnt/medias_share

```

*Démarrer le conteneur.*

#### 3. Installation et Configuration de Samba

Ouvrir la console du conteneur Samba en `root` :

```bash
# Installation du paquet
apt update && apt install -y samba

# Création d'un utilisateur sécurisé (ex: "admin")
useradd admin
smbpasswd -a admin # Définir un mot de passe pour le partage Windows

```

Éditer le fichier de configuration de Samba :

```bash
nano /etc/samba/smb.conf

```

Effacer tout le contenu (ou aller tout à la fin) et coller ce bloc de configuration strict :

```ini
[Medias]
   path = /mnt/medias_share
   writable = yes
   valid users = admin
   force user = root

```

*(L'astuce `force user = root` permet à Samba d'écrire les fichiers avec les mêmes droits que Plex, évitant tout conflit de permissions entre les deux conteneurs).*

Redémarrer le service pour appliquer :

```bash
systemctl restart smbd

```

#### 4. Connexion depuis Windows

Sur le PC Windows :

1. Ouvrir l'Explorateur de fichiers.
2. Dans la barre d'adresse en haut, taper : `\\192.168.1.242` *(L'IP du LXC Samba)* et appuyer sur Entrée.
3. Windows demande des identifiants : saisir `admin` et le mot de passe configuré via `smbpasswd`.
4. Le dossier **Medias** apparaît. Faire un clic droit dessus > **Connecter un lecteur réseau** pour lui attribuer une lettre (ex: `Z:`).

![lecteur](/images/2026-03-08-15-55-51.png)

---

## Phase 8 : Device Mapping GPU (Intel QSV) pour LXC Plex

### 1. Identification du périphérique sur l'Hyperviseur (Host)

Il faut d'abord s'assurer que la carte graphique intégrée (iGPU) est reconnue par l'hôte Proxmox et identifier ses identifiants système.

* Ouvrir un shell sur le nœud Proxmox et exécuter :

```bash
ls -l /dev/dri

```

* Le résultat doit afficher un périphérique nommé `renderD128`. C'est l'interface de rendu d'accélération matérielle.
* Il est crucial de noter le numéro de groupe (GID) propriétaire de `renderD128` (généralement le groupe `render`, avec un GID autour de 104 ou 108).

### 2. Configuration du montage LXC (Démarche DevOps)

Pour respecter le principe de sécurité du moindre privilège, on ne donne accès qu'au composant de rendu vidéo strict, sans exposer l'intégralité des bus matériels.

* Éditer le fichier de configuration du conteneur (remplacer `XXX` par l'ID du conteneur Plex) :

```bash
nano /etc/pve/lxc/XXX.conf

```

* Ajouter les lignes suivantes à la fin du fichier pour mapper le périphérique et lui accorder les droits de lecture/écriture/création (rwm) :

```text
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file

```

### 3. Gestion des Permissions (Sécurité Sysadmin)

Si le conteneur LXC a été créé en mode "Non-Privilégié" (ce qui est la norme de sécurité standard pour isoler les flux), l'utilisateur interne exécutant le service Plex n'aura pas les droits pour écrire sur le périphérique `renderD128` de l'hôte.

* Il faut se connecter à l'intérieur du conteneur Plex.
* Ajouter l'utilisateur `plex` au groupe vidéo/render local. Si une erreur de droit persiste, il faudra mettre en place un *ID Mapping* (correspondance des UID/GID) entre le conteneur et l'hyperviseur.

### 4. Activation applicative

* Redémarrer le conteneur LXC.
* Se connecter à l'interface d'administration web de Plex.
* Naviguer dans **Paramètres > Transcodeur**.
* Cocher l'option **"Utiliser l'accélération matérielle quand disponible"**.

---
