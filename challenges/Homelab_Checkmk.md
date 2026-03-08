# 📊 LAB : Supervision de l'Infrastructure et Visibilité Réseau

```txt
_________ .__                   __            __    
\_   ___ \|  |__   ____   ____ |  | __ _____ |  | __
/    \  \/|  |  \_/ __ \_/ ___\|  |/ //     \|  |/ /
\     \___|   Y  \  ___/\  \___|    <|  Y Y  \    < 
 \______  /___|  /\___  >\___  >__|_ \__|_|  /__|_ \
        \/     \/     \/     \/     \/     \/     \/
```

**Rôle :** Administrateur Réseau

**Mission :** Déployer Checkmk (Raw Edition), une solution de supervision d'infrastructure hautes performances. Il fonctionne en interrogeant activement les équipements du réseau via des agents ultra-légers ou des protocoles standards (SNMP) pour remonter l'état de santé, la bande passante, et les erreurs de flux en temps réel. Cette méthode permet de cartographier instantanément l'architecture réseau (routeurs, switchs, hyperviseurs, serveurs) grâce à un puissant moteur d'auto-découverte, offrant ainsi une vue "hélicoptère" proactive pour détecter les goulots d'étranglement ou les pannes avant qu'elles n'impactent les utilisateurs.

[Github Checkmk](https://github.com/Checkmk/checkmk)

![checkmk](/images/2026-02-27-10-50-09.png)

---

## L'intérêt technique 🎯

1. **Visibilité Réseau et Flux :** Analyser en temps réel la charge des interfaces (bande passante), la topologie, et surtout détecter les anomalies de transmission (comme les paquets perdus ou en erreur sur une interface `ens18`).
2. **Efficacité (Auto-découverte) :** Oublier la configuration manuelle fastidieuse. Checkmk scanne l'hôte cible et identifie de lui-même les dizaines de services et interfaces réseaux à monitorer.
3. **Sécurité (Flux TLS) :** Garantir la confidentialité des données de supervision en instaurant un canal chiffré de bout en bout (TLS) et une authentification forte entre le serveur central et ses agents sur le port TCP 6556.

---

## 🛠️ Architecture du Lab

* **Environnement :** Serveur Proxmox
* **Serveur Central :** Conteneur LXC sous Debian 13 (Trixie) alloué avec 1 vCPU et 1 Go de RAM.
* **Cibles (Agents) :** Machines virtuelles Ubuntu et PC physiques sous Windows 10/11.
* **Réseau Lab :** `10.0.0.0/16`
* **Cible Checkmk (LXC) :** L'IP statique de notre serveur sera `10.0.0.80`
* **Site de supervision :** `mkmonitor`

---

> Documentation : <https://docs.checkmk.com/latest/fr/>
>
> Installation Guide : <https://checkmk.com/download?platform=cmk&distribution=debian&release=trixie&edition=cre&version=2.4.0p22>

---

## Pré-requis Proxmox et LXC

Installation d'un container LXC

```sh
Hostname : Checkmk
Template : debian-13-standard (ou ubuntu-24.04)
Disque : Taille 15 Go
CPU Cœurs : 1
RAM : 1024 Mo (1 Go)
IPv4 : Static
IPv4/CIDR : 10.0.0.80/16
Passerelle : 10.0.0.1
DNS : 10.0.0.1
```

Une fois lancé on doit vérifier si `wget` et `ssh` sont bien installés

```sh
apt update && apt upgrade -y
apt install wget && apt install openssh-server
```

Pour pouvoir se connecter en SSH on doit modifier le fichier `sshd_config`

```sh
nano /etc/ssh/sshd_config
   PermitRootLogin yes
```

et restart `systemctl restart sshd`

ou en ligne de commande directement :

```sh
sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin yes/' /etc/ssh/sshd_config
systemctl restart ssh
```

On va aussi configurer l'IP en statique : `nano /etc/network/interfaces`

```sh
auto ens18
iface ens18 inet static
    address : 10.0.0.50
    netmask : 255.255.0.0
    gateway : 10.0.0.1
    dns-nameservers : 10.0.0.1
```

et restart `systemctl restart networking`

---

## Installation et Configuration Checkmk

Téléchargement de la clé GPG officielle depuis leur site web :

`wget https://download.checkmk.com/checkmk/Check_MK-pubkey.gpg`

Installer l'outil GPG

`apt update && apt install gnupg -y`

Importer la clé dans la liste des signatures fiables du système :

`gpg --import Check_MK-pubkey.gpg`

Téléchargement du packet spécifique pour Debian 13 :

`wget https://download.checkmk.com/checkmk/2.4.0p22/check-mk-raw-2.4.0p22_0.trixie_amd64.deb`

Vérifier la validité du paquet avec gpg :

`gpg --verify ./check-mk-raw-*.deb`

![gpg](/images/2026-02-27-12-49-12.png)

Installation du paquet avec les dépendances :

`sudo apt install ./check-mk-raw-2.4.0p22_0.trixie_amd64.deb`

Test de l'installation :

`omd version`

> omd : Open Monitory Distribution

![omd](/images/2026-02-27-12-51-24.png)

On va maintenant pouvoir créer notre instance de supervision :

`omd create mkmonitor`

> ⚠️ Bien noter le mot de passe aléatoire généré pour l'utilisateur `cmkadmin`, et l'URL pour se connecter à l'interface web

![password](/images/2026-02-27-13-01-03.png)

Démarrer le moteur de supervision :

`omd start mkmonitor`

![OK](/images/2026-02-27-13-04-30.png)

L'installation est terminée. On peut se connecter à l'interface web en tapant <http://10.0.0.80/mkmonitor/> dans le navigateur 🎉

![login](/images/2026-02-27-13-06-14.png)

![dashboard](/images/2026-02-27-13-06-28.png)

Pour vérifier les services de l'instance : `omd status`

![status](/images/2026-02-27-13-10-59.png)

Schema des différentes façons dont Checkmk peut accéder aux systèmes à superviser

![agents](/images/2026-02-27-13-16-39.png)

---

## Installation des Agents

> Documentation Agents de supervision checkmk : <https://docs.checkmk.com/latest/fr/wato_monitoringagents.html>

Le grand avantage de Checkmk, c'est que l'installation des agents ne demande aucune configuration système fastidieuse : on installe le paquet, ça ouvre un port d'écoute, et tout le reste de l'intelligence se gère depuis l'interface web centrale.

Pour déployer tes agents : Le serveur est le dépôt 💡

![agents](/images/2026-02-27-13-23-15.png)

Pas besoin de chercher les agents sur Internet. Dès que le site "mkmonitor" est créé, Checkmk génère et héberge lui-même les agents.

Ils sont toujours disponibles directement sur le serveur via l'interface web dans :
Setup > Agents > Windows, Linux, Solaris, AIX, etc

On peut copier directement le lien de l'agent dont on a besoin.

### Linux

![agents](/images/2026-02-27-13-51-44.png)

Pour télécharger l'agent depuis notre propre serveur Checkmk avec `wget` :

`wget http://10.0.0.80/mkmonitor/check_mk/agents/check-mk-agent_2.4.0p22-1_all.deb`

Installation de l'agent :

`sudo apt install ./check-mk-agent_*.deb`

### Windows

C'est encore plus simple, on ouvre un navigateur directement sur le PC Windows cible et on se connecte à l'interface Checkmk. Setup > Agents > Windows.

On télécharge le package Windows Installer (.msi).

![windows](/images/2026-02-27-13-56-43.png)

Et on lance le fichier téléchargé, et on termine l'install avec les option de base.

![exe](/images/2026-02-27-14-07-34.png)

Côté réseau : L'installateur crée automatiquement un service Windows en arrière-plan et ajoute la règle au pare-feu Windows pour ouvrir le fameux port TCP 6556. Il n'y a rien d'autre à faire.

---

## Découverte des agents

### Ajout des agents

Une fois que le port 6556 des machines cibles est ouvert et prêt à répondre, on retourne sur l'interface web Checkmk pour lancer la découverte :

On va dans Setup > Hosts > Add host

![add](/images/2026-02-27-14-12-49.png)

On renseigne le nom de la machine (ex: PC-Fixe ou Proxmox-Host) et son adresse IP, puis **Save & view folder**

![add](/images/2026-02-27-14-14-21.png)

On peut ajouter nos autres hôtes

![hosts](/images/2026-02-27-14-17-30.png)

### Sécurisation des flux

Pour sécuriser les flux et les chiffrer on va utiliser la commande :

`sudo cmk-agent-ctl register --hostname ubuntu-9005 --server 10.0.0.80 --site mkmonitor --user cmkadmin --password Zy8KpTRvhZi6`

> 🔐 L'agent va s'authentifier auprès du serveur Checkmk, échanger des certificats, et tout le trafic de supervision sur le port 6556 sera chiffré. De plus, l'agent n'acceptera de parler qu'au serveur.

![secure](/images/2026-02-27-14-20-12.png)

Pour Windows on fait de même en invite de commande ou powershell en tant qu'administrateur

`sudo cmk-agent-ctl register --hostname windows10-9002 --server 10.0.0.80 --site mkmonitor --user cmkadmin --password Zy8KpTRvhZi6`

Si ça ne fonctionne pas (windows ne reconnaît pas la variable d'environnement) on peut utiliser

`"C:\Program Files (x86)\checkmk\service\cmk-agent-ctl.exe" register --hostname windows10-9002 --server 10.0.0.80 --site mkmonitor --user cmkadmin --password Zy8KpTRvhZi6`

### Découverte

Maintenant que le canal de communication est sécurisé, on va pouvoir aspirer les métriques : la découverte automatique.

Voici comment faire remonter toutes les informations dans le tableau de bord : Setup > Hosts > cliquer sur le carré jaune `Run Service Discovery` 🟨

![discover](/images/2026-02-27-14-30-56.png)

Là, Checkmk va scanner les services sur la machine et te lister tout ce qu'il a trouvé (interfaces réseaux, CPU, disques, services).

On valide avec `Accept all` pour commencer à monitorer toutes ces métriques.

![datasources](/images/2026-02-27-14-33-04.png)

On fait de même pour les autres agents/machines

Les changements sont "en attente" (pending). Il faut les appliquer pour que le moteur de supervision les prenne en compte. Tout en haut à droite il y a un bouton jaune avec un point d'exclamation (indiquant par exemple 2 changes) : `Activate on selected sites`

On peut retrouver les résultats du check dans Monitor > Overview > All Hosts

![hosts](/images/2026-02-27-14-42-05.png)

![dashboard](/images/2026-02-27-14-45-34.png)

---

## Intégration API Proxmox VE

**Objectif :** Permettre au serveur de supervision (Checkmk) d'interroger directement l'hyperviseur (Proxmox) via son API pour remonter l'état des interfaces réseaux virtuelles (`vmbr`) et des nœuds sans installer d'agent lourd sur chaque système.

### 1. Préparation côté Proxmox (Sécurité & Pré-requis)

* **Création du compte de service :** Créer un utilisateur `checkmk@pve` avec un mot de passe classique.
* *⚠️ Piège de l'authentification :* La sonde native Checkmk utilise le chemin `/api2/json/access/ticket` qui attend un couple identifiant/mot de passe explicite. Les "API Tokens" modernes génèrent une erreur d'authentification.

* **Droits d'accès :** Assigner le rôle `PVEAuditor` (lecture seule) à cet utilisateur sur le chemin racine `/`.
* **Le correctif du fuseau horaire :** Dans `PVE-Server > System > Time`, resélectionner et valider le fuseau horaire (ex: *Europe/Paris*).
* *⚠️ Piège du `StopIteration` :* Sans cette validation manuelle dans l'interface web, Proxmox ne génère pas le fichier `/etc/timezone`. L'API renvoie un champ vide, ce qui fait planter le script Python de Checkmk avec l'erreur critique `StopIteration`.

![user](/images/2026-03-06-01-52-54.png)

### 2. Configuration côté Checkmk (La Sonde API)

* **Alignement d'identité :** Le nom de l'hôte dans Checkmk doit correspondre **exactement** au nom de l'hyperviseur (ex: `pve-server` en minuscules).
* *⚠️ Piège de la casse :* Si le nom diffère (ex: majuscules), Checkmk ne trouvera pas les données correspondantes dans le fichier `.json` renvoyé par Proxmox et plantera.

* **Création de la règle "Proxmox VE" (*Setup > Agents > VM, cloud, container*) :**
* **Username :** `checkmk@pve`
* **Password :** *Explicit* + le mot de passe créé sur Proxmox.
* **Validation SSL :** Cocher la case principale `Disable SSL certificate validation` **ET** cocher obligatoirement la sous-option `SSL certificate validation is disabled`.
* *⚠️ Piège du certificat local :* Oublier la sous-option déclenche l'erreur `[SSL: CERTIFICATE_VERIFY_FAILED]`, car Proxmox utilise un certificat auto-signé.
* **Conditions :** Assigner la règle explicitement à l'hôte `pve-server`.

![host](/images/2026-03-06-01-54-06.png)

### 3. Finalisation (Le cumul des flux)

* **Propriétés de l'hôte Checkmk :** Dans la section *Data sources*, forcer l'option **Configured API integrations AND Checkmk agent**.
* *⚠️ Piège de l'écrasement :* Par défaut, l'ajout d'une API désactive l'écoute de l'agent TCP classique. Cumuler les deux permet d'avoir à la fois les métriques du système hôte et celles de l'hyperviseur.

![agents](/images/2026-03-06-01-59-43.png)

* **Activation :** Appliquer les changements (point d'exclamation jaune) et lancer une découverte de services (*Rescan*).

![activation](/images/2026-03-06-02-20-32.png)

### 4. Ajout d'une VM

* Voici comment ajuster les options pour activer la magie du **Piggyback** :

  * **Host name** : le nom exact de la VM dans Proxmox
  * **Network address** (Pour éviter que Checkmk ne cherche à pinguer la machine) :
  * Cocher la petite case grise juste à gauche de **IP address family**.
  * Le menu déroulant va s'activer : remplacer "IPv4 only" par **No IP**.

* **Monitoring agents** (Pour désactiver l'interrogation directe) :

  * Cocher la petite case grise à gauche de **Checkmk agent / API integrations**.
  * Dans le menu déroulant, sélectionner **No API integrations, no Checkmk agent**.
  * (Laisser la ligne "**Piggyback**" juste en dessous sur sa valeur par défaut, c'est elle qui autorise la récupération silencieuse des données !)

Cliquer sur le bouton **Save & run service discovery** en haut de page.

![add](/images/2026-03-06-12-32-10.png)

![VM](/images/2026-03-06-12-34-41.png)

---

## Intégration de pfSense avec le protocole SNMP

### 1. Configuration de pfSense (Activation SNMP)

#### Installer le paquet sécurisé

1. Naviguer dans **System > Package Manager**, puis dans l'onglet **Available Packages**.
2. Rechercher `net-snmp`.
3. Cliquer sur **Install** puis **Confirm** pour lancer le déploiement du paquet.

#### Configurer SNMPv3 via NET-SNMP

Un nouveau menu de configuration est désormais disponible.

1. Naviguer dans **Services > SNMP (NET-SNMP)**.
2. Dans l'onglet **General** : Cocher *Enable the SNMP service* et sauvegarder.
3. Se rendre dans l'onglet **Users** et cliquer sur le bouton **Add**.
4. Renseigner les champs de sécurité :

* **Username :** `srv_checkmk`
* **Entry type :** `User entry (USM)`
* **Read/Write Access :** `Read Only (GET, GETNEXT)` (*Application stricte du principe de moindre privilège*).
* **Authentication Type :** `SHA`
* **Password :** (Saisir le mot de passe d'authentification).
* **Privacy Protocol :** `AES`
* **Passphrase :** (Saisir la clé de chiffrement).

1. Cliquer sur **Save**.

### 2. Filtrage Réseau (Pare-feu pfSense)

L'activation du service ne suffit pas ; il faut autoriser le flux réseau entre Checkmk et pfSense.

1. Naviguer vers **Firewall > Rules**.
2. Sélectionner l'interface sur laquelle Checkmk va interroger pfSense (généralement l'interface de Management ou le LAN, par exemple `vmbr1` ou `LAN`).
3. Ajouter une règle avec les paramètres suivants :

* **Action :** `Pass`
* **Interface :** `LAN` (ou l'interface concernée)
* **Address Family :** `IPv4`
* **Protocol :** `UDP`
* **Source :** `Single host or alias` -> Saisir l'adresse IP exacte de la machine Checkmk (ex: `192.168.1.X`).
* **Destination :** `This Firewall (self)`
* **Destination Port Range :** `SNMP (161)`

1. Ajouter une description claire (ex: `Allow Checkmk to poll pfSense SNMP`).

2. Sauvegarder et appliquer les changements.

### 3. Intégration dans Checkmk (Administration Système)

Une fois la couche réseau et le service préparés, il faut déclarer l'hôte dans l'outil de supervision.

1. Se connecter à l'interface web de Checkmk.
2. Aller dans **Setup > Hosts** et cliquer sur **Add host**.
3. Remplir les informations de base :

* **Hostname :** `pfsense-fw01`
* **IPv4 address :** Saisir l'adresse IP de pfSense.

Dans la section **Data Sources**, configurer comme suit :

* **Checkmk agent :** Sélectionner `No API integrations, no Checkmk agent` (puisqu'aucun agent lourd n'est installé sur pfSense).
* **SNMP :** Cocher la case et sélectionner `SNMP v2c or v3`.

Dans les **SNMP credentials**, sélectionner **SNMPv3 (authPriv)** et reporter exactement les informations configurées sur pfSense
:

* Security level: `authPriv`
* User name: `srv_checkmk`
* Authentication protocol: `SHA-1`
* Authentication password: (Le mot de passe défini)
* Privacy protocol: `AES-128`
* Privacy passphrase: (La clé définie)

![agent](/images/2026-03-08-00-27-00.png)

Cliquer sur **Save & run service discovery**

Checkmk va envoyer des requêtes UDP 161. Si le pare-feu et SNMPv3 sont bien configurés, les services (Interfaces réseau, CPU, RAM, Uptime) vont apparaître.

Accepter les services voulus (`Accept all`) et activer les changements (Bouton jaune **Activate on selected sites**).

![OK](/images/2026-03-08-00-26-41.png)

⚠️ Toujours vérifier le `timedatectl status` s'il y a un décalage les vérifications avec le `NTP` échoueront.

---

## Custom Dashboard

![dash](/images/2026-02-27-15-29-01.png)

to be continued...
