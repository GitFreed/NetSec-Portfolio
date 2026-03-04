# LAB : Déploiement d'un Routeur/Pare-feu pfSense sous Proxmox

```txt
        __
 _ __  / _|___  ___ _ __  ___  ___
| '_ \| |_/ __|/ _ \ '_ \/ __|/ _ \
| |_) |  _\__ \  __/ | | \__ \  __/
| .__/|_| |___/\___|_| |_|___/\___|
|_|
```

**Rôle :** Architecte / Ingénieur Réseau

**Mission :** Concevoir et déployer une architecture réseau segmentée et virtualisée. L'objectif est d'isoler un domaine de diffusion local (LAN) d'un réseau externe (WAN) au travers d'une appliance de routage professionnelle (pfSense), en configurant le routage, le NAT, les services DHCP et les tunnels cryptographiques (OpenVPN), avec une approche strictement orientée flux et topologies.

---

## L'intérêt technique 🎯

* **Séparation des couches (L2/L3) :** Exploiter les ponts virtuels (Linux Bridges) pour créer des topologies propres, en les utilisant soit comme des commutateurs isolés (Layer 2), soit comme des passerelles de routage (Layer 3).
* **Maîtrise du NAT et du filtrage :** Configurer le masquage d'adresses (MASQUERADE) pour la sortie internet et la redirection de ports (DNAT) via `iptables` pour gérer finement les flux entrants et sortants.
* **Déploiement d'Appliance Réseau :** Manipuler pfSense, un véritable standard de l'industrie, pour implémenter des politiques de sécurité et des règles de pare-feu aux frontières de l'infrastructure.
* **Tunnels VPN :** Implémenter une infrastructure à clés publiques (CA/Certificats) et monter un tunnel OpenVPN (interface *tun*) pour acheminer le trafic de clients distants directement vers le sous-réseau privé.

---

## 🛠️ Architecture du Lab

* **Environnement :** Serveur de virtualisation Proxmox (Hyperviseur de type 1).
* **Topologie Réseau (Interfaces virtuelles) :**

  * `vmbr0` **(Sortie WAN) :** Pont lié à l'interface physique (`nic0`), doté d'une IP statique sur le réseau domestique externe (`192.168.1.240/24`). Porte les règles `iptables` de post-routage.
  * `vmbr1` **(Segment de Transit) :** Réseau d'interconnexion interne (`192.168.10.0/24`) assurant le lien point-à-point entre l'hyperviseur (`192.168.10.1`) et la patte WAN du routeur pfSense (`192.168.10.254`).
  * `vmbr2` **(Switch LAN) :** Commutateur virtuel pur (aucune IP côté hyperviseur) isolant le domaine de diffusion privé des machines clientes (`10.0.0.0/24`).

* **Équipements et Nœuds :**
  * **Routeur (pfSense 2.7.2) :** Passerelle par défaut (`10.0.0.1`), serveur DHCP et concentrateur VPN. Utilisation de cartes réseau paravirtualisées (*VirtIO*) pour maximiser le débit de commutation interne.
  * **Sonde de test (Lubuntu/Windows 10) :** Machine cliente allégée connectée sur le `vmbr2`, utilisée pour valider l'attribution DHCP, la résolution DNS locale et le routage de bout en bout.

---

> Documentation : <https://docs.netgate.com/pfsense/en/latest/config/>

---

## Configuration Proxmox

Proxmox, un Hyperviseur de type 1, permet de superviser le matériel serveur. On doit configurer l'interface réseau en ajoutant des Bridges qui vont permettre de connecter nos interfaces réseau virtuelles à l'interface réseau physique.

Mon serveur a une seule adresse IP sur mon LAN, déjà configurée.

On va devoir mettre en place du NAT : un premier directement au niveau de Promox, un deuxième dans une VM pfSense

### Interfaces bridge

Sur GNU/Linux, on peut créer des interfaces réseaux appelées `bridges` (pont, en français). Ces interfaces virtuelles vont nous permettre d'inter-connecter des interfaces réseau physiques et également les interfaces réseau de nos machines virtuelles

On peut voir ça comme une sorte de **switch virtuel**

Par défaut, on a qu'une seule interface bridge : `vmbr0`, l'interface réseau physique `nic0` du serveur y est connectée. ⚠️ *Ne pas modifier la configuration de cette interface !*

**Cette interface `vmbr0` sera l'interface de sortie de notre NAT.**

Pour connecter le "WAN" de la VM pfSense qu'on va créer par la suite, il va nous falloir une nouvelle interface bridge.

➡️ Dans la section `Système > Réseau` du serveur Proxmox : `Créer` > `Linux Bridge` pour créer cette nouvelle interface.

➡️ Dans le champ `IPv4/CIDR`, on met l'adresse IP statique au format CIDR que nous voulons attribuer à notre serveur Proxmox sur cette interface : `192.168.10.1/24`. `Démarrage automatique` doit être coché, on laisse le reste des champs vides et `OK`.

➡️ On refait la même opération pour créer l'interface `vmbr2`, sans adresse `IPv4/CIDR` ce coup-ci ⚠️

➡️ `Appliquer la configuration` en haut !

On peut maintenant mettre en place notre premier NAT, au niveau du serveur Proxmox ! Proxmox est basé sur Debian, donc on va utiliser le pare-feu `iptables` pour faire ça

➡️ On ouvre le `Shell`, on vérifie nos interfaces bridges avec `ip a` pour voir si elles sont bien détectées et correctement configurées.

On doit avoir la configuration suivante :

* **vmbr0 :** l'adresse IP du serveur (`192.168.1.240/24`)
* **vmbr1 :** `192.168.10.1/24`
* **vmbr2 :** pas d'adresse IPv4

### NAT Proxmox

➡️ Créer un fichier dédié au routage dans le dossier prévu à cet effet (`sysctl.d`) avec : `sudo nano /etc/sysctl.d/99-routing.conf`

On ajoute `net.ipv4.ip_forward=1` puis sauvegarder et quitter.

➡️ On lance la commande `sudo sysctl --system` pour appliquer la modification que nous venons d'effectuer, on doit voir défiler quelques lignes, dont le `net.ipv4.ip_forward = 1` vers la fin..

> 💡 Cette modification permet d'**activer l'IP forward**, sorte de "mode routeur" du noyau Linux.

➡️ Pour finir, `sudo iptables -t nat -A POSTROUTING -s 192.168.10.0/24 -o vmbr0 -j MASQUERADE` afin d'activer le NAT.

> `-s 192.168.10.0/24` permet de n'autoriser que les paquets en provenance de ce sous-réseau à traverser le NAT.
>
> `-o vmbr0` permet d'indiquer l'interface réseau de sortie.

On peut lancer la commande `sudo iptables -L -t nat` pour vérifier la configuration du pare-feu `iptables` et vérifier la configuration des interfaces avec `sudo nano /etc/network/interfaces`

![interfaces](/images/2026-03-04-00-39-21.png)

Ça y est, la configuration réseau sur Proxmox est terminée 🎉

## pfSense

💡 **pfSense est un système d'exploitation** permettant de transformer n'importe quel ordinateur en un **routeur** professionnel. Il n'est pas rare d'en rencontrer en entreprise !

Sur cette étape, on va créer une VM pfSense qui servira de passerelle/de routeur pour toutes nos VMs pour le reste de la formation.

### Création VM

➡️ Couton `Créer une VM`, tout en haut dans l'interface.

Voici les réglages à utiliser :

```sh
ID : 1000
nom : pfSense
iso : pfSense 2.7.2
type : Other (à la place de "Linux")
disque : 20 Gio
SSd Emulation : ✅
Discard : ✅
RAM : 2048 MiB
CPU : 1
Pont (bridge) : vmbr1 (⚠️ très important)
Pare-feu : décoché (⚠️ très important)
Model : VirtIO
```

**Laisser tous les autres réglages par défaut.**

➡️ **Avant de démarrer la VM**, on ajoute une seconde interface réseau depuis la section `Matériel`. On sélectionne le Pont `vmbr2` cette fois-ci, et on décoche aussi le pare-feu.

### Installation

➡️ On démarre la VM, et on va dans la section `Console`

Rien de particulier pendant l'installation, il faut appuyer quasiment tout le temps sur `Entrée` pour valider le choix par défaut !
⚠️ **Seule exception**, lors du choix du disque dur, il faudra appuyer sur la touche `Espace` pour cocher la case **avant** d'appuyer sur `Entrée`.

💡 Une fois l'installation terminée, on éjecte l'image ISO du lecteur de DVD virtuel via la section `Matériel` (`Éditer > N'utiliser aucun média`), afin d'éviter que l'installation se relance en boucle après le redémarrage.

![interface](/images/2026-03-04-01-10-14.png)

### Configuration et test

Une fois la VM pfSense démarrée, elle nous demande de préciser les interfaces WAN et LAN, on met *no* pour les vlans, WAN : `vtnet0` et LAN : `vtnet1`

Maintenant on va pouvoir configurer les adresses IPv4 sur ses interfaces et vérifier qu'elle a bien accès à Internet.

💡 Notre pfSense ne va pas récupérer automatiquement d'adresse sur son interface WAN : il n'y a pas de serveur DHCP actif sur notre pont `vmbr1` !

➡️ Sur pfSense : `8) Shell` avec le clavier (attention il est en Qwerty de base !), puis `kbdcontrol -l /usr/share/vt/keymaps/fr.kbd` pour passer le clavier en Azerty. `exit` pour retourner au menu.

➡️ `2) Set interface(s) IP address`, puis `1` pour configurer l'interface WAN (192.168.10.254).

```txt
Configure IPv4 address WAN interface via DHCP? : n (Non, on veut l'IP statique)
Enter the new IPv4 WAN address: 192.168.10.254
Enter the new IPv4 subnet bit count: 24
For a WAN, enter the new IPv4 upstream gateway address: 192.168.10.1 (⚠️ Très important : c'est l'adresse du Proxmox sur le pont vmbr1, c'est lui qui assure le routage vers la box internet)
Use this gateway as default : y
Configure IPv6 address WAN interface via DHCP6? : n.
Enter the new IPv6 WAN address: Vide et Entrée.
Enable the DHCP server on WAN : n
Do you want to revert to HTTP as the webConfigurator protocol? : n
```

➡️ `7) Ping host` > `8.8.8.8` pour vérifier que la VM a bien accès à Internet.

➡️ `2) Set interface(s) IP address`, puis `2` pour configurer l'interface LAN (10.0.0.1).

```txt
Configure IPv4 address WAN interface via DHCP? : n (Non, on veut l'IP statique)
Enter the new IPv4 WAN address: 192.168.10.254
Enter the new IPv4 subnet bit count: 24
For a WAN, enter the new IPv4 upstream gateway address: Vide et Entrée
Configure IPv6 address WAN interface via DHCP6? : n.
Enter the new IPv6 WAN address: Vide et Entrée.
Enable the DHCP server on WAN : y
Enter the start address of the IPv4 client address range: 10.0.0.50
Enter the end address of the IPv4 client address range: 10.0.0.150
Do you want to revert to HTTP as the webConfigurator protocol? : n
```

![interfaces](/images/2026-03-04-01-26-12.png)

### VM Test et interface web

➡️ On va créer une nouvelle VM Lubuntu

💡 L'interface réseau doit être connectée sur le pont `vmbr2`, notre "switch virtuel" pour toutes nos VMs ! (décocher également le pare-feu dans la partie `Réseau`)

À la fin de l'installation, la machine devrait obtenir une adresse IP grace au serveur DHCP de la pfSense installée précédemment.

➡️ Depuis le navigateur web de la VM , on va sur l'interface de la pfSense à l'adresse [10.0.0.1](10.0.0.1).

💡 Le navigateur nous avertira d'un risque de sécurité : `Avancé` et poursuivre !

![login](/images/2026-02-26-18-25-45.png)

On se connecte avez le nom d'utilisateur `admin` et le mot de passe `pfsense`.

➡️ On suit les étapes du `Wizard pfSense Setup`, pour effectuer la configuration initiale de notre pfSense.

Voici les réglages à utiliser :

```txt
- Première page :
  - Hostname : pfSense
  - Domain : pve-server.lan
  - Primary DNS Server : 192.168.1.250
  - Secondary DNS Server : 9.9.9.9
  - Override DNS : décocher
- Deuxième page :
  - Time server hostname : laisser le serveur renseigné par défaut
  - Timezone : changer pour `Europe/Paris`
- Troisième page :
  - rien à changer, sauf tout en bas ! Décocher la case `Block RFC1918 Private Networks` (⚠️ très important)
- Quatrième page :
  - rien à changer, on laisse l'IP configurée précédemment
- Cinquième page :
  - admin password : On met un vrai mot de passe fort
  ```

`Reload` pour appliquer les changements.

💡 Il y aura probablement un problème de DNS. Le service DNS interne de pfSense qui ne s'est pas bien initialisé avec les nouvelles interfaces. Pour le résoudre, il faut se rendre dans `Services > DNS Resolver` sur la pfSense, puis relancer le service en utilisant le bouton 🔄 en haut à droite.au service DNS interne de pfSense qui ne s'est pas bien initialisé avec les nouvelles interfaces.

Une fois le service relancé, on a accès à Internet depuis la VM Windows 🎉

On voit également le message d'alerte en jaune : *ISC HDCP has reached end-of-life...* on peut clicker **System / Advanced / Networking** et activer `Kea DHCP`

![pfsense](/images/2026-02-26-18-33-35.png)

## VPN

Pour pouvoir plus facilement bosser sur nos VMs par la suite, on va créer un VPN permettant de directement accéder à notre pfSense depuis le navigateur web de notre PC, et pouvoir prendre la main à distance sur nos VMs en utilisant le protocole RDP ou SSH.

### Serveur OpenVPN

Dans `VPN > OpenVPN`, onglet `Wizards`.

On laisse `Local User Access`, puis `Next`.

Remplir les différents champs en suivant :

* Première page :
  * Descriptive name : `vpn`
  * Common Name : `vpn`
  * les autres champs par défaut, et `Add new CA`.
* Deuxième page :
  * `Add new Certificate`
  * Descriptive name :`vpn-cert-server`
  * Common Name :`vpn-cert-server`
  * les autres champs par défaut, et `Create new Certificate`.
* Troisième page :
  * Description :`vpn-remote-access`
  * IPv4 Tunnel Network :`10.42.0.0/24`
  * IPv4 Local Network :`10.0.0.0/24`
  * les autres champs par défaut, et `Next`.
* Quatrième page :
  * Firewall Rule : ✅
  * OpenVPN rule : ✅
  * puis `Next`.
* et enfin `Finish` !

Le VPN est presque prêt ! Il faut encore que l'on créé les utilisateurs !

### Utilisateurs & certificats

Dans `System > User Manager`, puis `Add` :

* Username : `user`
* Password/Confirm Password : un mot de passe solide.
* Certificate : ✅ (⚠️ sinon ça ne fonctionnera pas)
* Section `Create Certificate for User` :
  * Descriptive name : `user-vpn-cert`
* les autres champs par défaut et `Save`.

On peut vérifier dans `System > Certificates` puis dans l'onglet `Certificates`, on doit avoir 2 nouveaux certificats (un pour le serveur, et un l'utilisateur, en plus du certificat `GUI default` qui est présent de base)

### Export de la configuration client

Pour pouvoir exporter facilement les configurations pour nos utilisateurs du serveur VPN, on va devoir installer un paquet logiciel sur la pfSense.

Dans `System > Package Manager` > `Available Packages`.

On cherche "openvpn", et on installe le premier paquet de la liste : `openvpn-client-export` (en cliquant sur le bouton `+ Install` correspondant) !

Ensuite dans `VPN > OpenVPN`, puis sur l'onglet `Client Export`.

On va changer le champ `Host Name Resolution` : on remplace `Interface IP Address` par `Other`, **puis dans le champ `Host Name` l'adresse IPv4 publique de notre serveur Proxmox.**

Une fois que c'est fait, on va télécharger le fichier Inline Configurations : `Most Clients` ou `OpenVPN Connect (iOS/Android)` pour notre utilisateur un peu plus bas.

### Redirection de port sur Proxmox

Sur le shell Proxmox, se connecter et lancer la commande suivante :

```sh
sudo iptables -t nat -A PREROUTING -i vmbr0 -p udp --dport 1194 -j DNAT --to-destination 192.168.10.254
```

💡 Cette commande permet de créer une redirection du port 1194 sur le protocole UDP (utilisé par OpenVPN) vers notre pfSense.

### Connexion au VPN

Il faut installer le logiciel [OpenVPN Connect](https://openvpn.net/client/).

Une fois sur OpenVPN Connect, il faudra cliquer sur le bouton `Upload File` en bas pour charger votre fichier de configuration VPN récupéré à l'étape précédente.

Une fois le fichier importé, on peut se connecter en cliquant sur le bouton `Connect`. Renseignez le nom d'utilisateur (`user`) et le mot de passe.

![vpn](/images/2026-03-04-02-33-53.png)

Depuis notre navigateur web (sur notre PC) accéder à [https://10.0.0.1/](https://10.0.0.1/). Si la page de connexion de notre pfSense s'affiche ça veut dire que le VPN fonctionne 🎉

## Sauvegarde iptables

Si tout fonctionne bien (Lubuntu a accès à Internet et le VPN fonctionne), on peut sauvegarder notre configuration `iptables`.

En effet, notre config' `iptables` pour le NAT sur Proxmox ne sera pas conservée après un redémarrage. Comme pour les équipements Cisco, il va falloir qu’on « sauvegarde » cette config.

➡️ **Sur le Shell de notre serveur Proxmox**, exporter la config' dans le fichier `/etc/iptables-rules.save` avec la commande `sudo iptables-save | sudo tee /etc/iptables-rules.save`.

On veut que ce fichier soit « appliqué » au démarrage, pour cela on doit rajouter la ligne `post-up iptables-restore < /etc/iptables-rules.save` dans le fichier `/etc/network/interfaces`, sous notre interface `vmbr0` (après une tabulation).

On utilise la commande `sudo nano /etc/network/interfaces` pour modifier la config, et on ajoute la ligne indiquée ci-dessus à la suite du vmbr0

![interfaces](/images/2026-03-04-00-44-33.png)

---

Super, félicitations pour avoir mené cette architecture jusqu'au bout ! 🎉

Voici la suite et fin pour ta fiche de lab, toujours rédigée sous l'angle de l'ingénierie des flux et du routage, prête à être copiée-collée à la suite de la première partie :

---

## 🎯 Validation de l'Architecture

* **Isolation L2 et Distribution L3 :** Le domaine de diffusion privé (`vmbr2`) est parfaitement étanche. Le pare-feu agit comme passerelle exclusive (`10.0.0.1/24`) et gère l'attribution dynamique des baux DHCP pour les clients du réseau local.
* **Résolution DNS :** Intégration d'un résolveur externe (AdGuard Home en `192.168.1.250`). Le pfSense agit comme relais (DNS Forwarder), validant la chaîne de requêtes depuis le LAN jusqu'au réseau physique en passant par le pont de transit.
* **Accès Distant (VPN) :** Création réussie d'un tunnel chiffré L3 (OpenVPN - interface *tun*). La table de routage est correctement renseignée pour assurer un flux bidirectionnel entre le réseau d'adressage du tunnel (`10.42.0.0/24`) et le sous-réseau LAN cible (`10.0.0.0/24`).

## 🔄 Analyse de Flux : Le chemin d'un paquet VPN (Inbound)

1. **Initiation :** Le client externe cible l'IP externe de l'hyperviseur (`192.168.1.240` sur le port UDP `1194`).
2. **Translation (Hyperviseur) :** Proxmox intercepte le paquet sur `vmbr0` et applique une règle de destination NAT (DNAT) pour réécrire l'IP cible vers la patte WAN du pfSense (`192.168.10.254`).
3. **Décapsulation (Routeur) :** pfSense reçoit le paquet encapsulé sur son interface WAN, valide l'échange de clés asymétriques (PKI), et décapsule la charge utile pour lire l'IP de destination réelle.
4. **Routage Interne :** pfSense consulte sa table de routage et transfère le paquet décapsulé vers l'interface LAN (`vmbr2`) pour atteindre la machine cible dans le réseau `10.0.0.0/24`.

## ⚙️ Commandes et Configurations Clés

**Règles de routage Proxmox (`iptables`) :**
Pour assurer la translation d'adresse (Sortie internet) et la redirection de port (Entrée VPN) en bordure de l'hyperviseur, sans toucher à la couche OS des VMs :

```bash
# Activation du transfert de paquets L3 (IP Forwarding) dans le noyau
sysctl -w net.ipv4.ip_forward=1

# Règle de Post-routage (Source NAT / Masquerade) pour l'accès internet du réseau de transit
iptables -t nat -A POSTROUTING -s 192.168.10.0/24 -o vmbr0 -j MASQUERADE

# Règle de Pré-routage (Destination NAT / Port Forwarding) pour le trafic OpenVPN entrant
iptables -t nat -A PREROUTING -i vmbr0 -p udp --dport 1194 -j DNAT --to 192.168.10.254:1194

```

---
