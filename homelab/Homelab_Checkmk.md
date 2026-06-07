# 📊 LAB : Supervision de l'Infrastructure avec Checkmk

```txt
_________ .__                   __            __    
\_   ___ \|  |__   ____   ____ |  | __ _____ |  | __
/    \  \/|  |  \_/ __ \_/ ___\|  |/ //     \|  |/ /
\     \___|   Y  \  ___/\  \___|    <|  Y Y  \    < 
 \______  /___|  /\___  >\___  >__|_ \__|_|  /__|_ \
        \/     \/     \/     \/     \/     \/     \/
```

![Type](https://img.shields.io/badge/Environnement-Conteneur%20LXC-FFA500?style=flat-square&logo=linux&logoColor=white)
![OS](https://img.shields.io/badge/OS-Debian%2013%20Trixie-A81D33?style=flat-square&logo=debian&logoColor=white)
![Service](https://img.shields.io/badge/Service-Checkmk%20Raw%202.4-00B32C?style=flat-square)
![Role](https://img.shields.io/badge/Rôle-Supervision-005085?style=flat-square)
![Network](https://img.shields.io/badge/Réseau-LAN-purple?style=flat-square&logo=web&logoColor=white)

**Rôle :** Administrateur Réseau / DevOps

**Mission :** Déployer Checkmk (Raw Edition), une solution de supervision d'infrastructure hautes performances. Checkmk interroge activement les équipements du réseau via des agents ultra-légers ou des protocoles standards (SNMP) pour remonter l'état de santé, la bande passante et les erreurs de flux en temps réel. Son moteur d'auto-découverte cartographie instantanément l'architecture réseau (routeurs, switches, hyperviseurs, serveurs) pour offrir une vue proactive et détecter les anomalies avant qu'elles n'impactent les utilisateurs.

> 📚 Documentation : <https://docs.checkmk.com/latest/fr/>

---

## L'intérêt technique 🎯

1. **Visibilité Réseau et Flux :** Analyser en temps réel la charge des interfaces (bande passante), la topologie, et détecter les anomalies de transmission (paquets perdus, erreurs CRC, saturation d'interfaces).
2. **Efficacité (Auto-découverte) :** Checkmk scanne l'hôte cible et identifie automatiquement les dizaines de services et interfaces à monitorer — pas de configuration manuelle fastidieuse.
3. **Intégration API :** Interrogation directe de l'API Proxmox pour superviser les machines virtuelles via le mécanisme de Piggyback, sans agent sur chaque VM.
4. **Sécurité (Flux TLS & SNMPv3) :** Canal chiffré de bout en bout entre le serveur central et ses agents (port TCP 6556), et interrogation SNMPv3 chiffrée (AES) pour les équipements réseau.

---

## 🛠️ Architecture du Lab

- **Environnement :** Serveur Proxmox VE
- **Serveur Central :** Conteneur LXC sous Debian 13 (Trixie)
  - IP : `192.168.1.241` (Zone LAN, sur `vmbr0`)
  - CPU : 1 vCPU | RAM : 1 Go | Disque : 15 Go
- **Site de supervision :** `mkmonitor`
- **Cibles supervisées :**
  - Proxmox VE (`192.168.1.240`) — via API + Agent
  - pfSense (`192.168.1.251`) — via SNMPv3
  - Machines virtuelles et LXC — via Piggyback et agents
  - Raspberry Pi (`192.168.1.250`) — via agent Linux

---

## 1️⃣ Pré-requis et Création du LXC

Création du conteneur LXC sur l'hyperviseur Proxmox :

```sh
Hostname : checkmk
Template : debian-13-standard
Disque : 15 Go
CPU : 1 Cœur
RAM : 1024 Mo (1 Go)
Réseau : vmbr0
IPv4/CIDR : 192.168.1.241/24
Passerelle : 192.168.1.254
DNS : 192.168.1.250
```

Une fois le conteneur démarré, installer les pré-requis :

```bash
apt update && apt upgrade -y
apt install -y wget openssh-server
```

> ⚠️ **Sécurité :** L'accès SSH en `root` direct (`PermitRootLogin yes`) est acceptable en lab pour simplifier l'administration, mais doit être durci en production (clé SSH, utilisateur dédié, `PermitRootLogin no`).

---

## 2️⃣ Installation de Checkmk

Téléchargement et installation du paquet avec vérification GPG :

```bash
# Importer la clé GPG officielle
wget https://download.checkmk.com/checkmk/Check_MK-pubkey.gpg
apt install -y gnupg
gpg --import Check_MK-pubkey.gpg

# Télécharger le paquet pour Debian 13
wget https://download.checkmk.com/checkmk/2.4.0p22/check-mk-raw-2.4.0p22_0.trixie_amd64.deb

# Vérifier la signature
gpg --verify ./check-mk-raw-*.deb
```

![gpg](/images/2026-02-27-12-49-12.png)

```bash
# Installer avec les dépendances
apt install -y ./check-mk-raw-2.4.0p22_0.trixie_amd64.deb
```

Vérification de l'installation : `omd version`

![omd](/images/2026-02-27-12-51-24.png)

### Création du site de supervision

```bash
# Créer l'instance
omd create mkmonitor

# ⚠️ NOTER le mot de passe généré pour l'utilisateur cmkadmin

# Démarrer le moteur
omd start mkmonitor
```

L'interface web est accessible sur `http://192.168.1.241/mkmonitor/` 🎉

![login](/images/2026-02-27-13-06-14.png)

Vérifier l'état des services : `omd status mkmonitor`

![status](/images/2026-02-27-13-10-59.png)

---

## 3️⃣ Déploiement des Agents

Le serveur Checkmk héberge lui-même les agents — pas besoin de les télécharger sur Internet. Disponibles dans : **Setup > Agents > Windows, Linux, Solaris, AIX**.

![agents](/images/2026-02-27-13-16-39.png)

![agents](/images/2026-02-27-13-23-15.png)

### Agent Linux

![agents](/images/2026-02-27-13-51-44.png)

```bash
# Télécharger depuis le serveur Checkmk
wget http://192.168.1.241/mkmonitor/check_mk/agents/check-mk-agent_2.4.0p22-1_all.deb

# Installer
apt install -y ./check-mk-agent_2.4.0p22-1_all.deb

# Enregistrer l'agent auprès du serveur (canal TLS chiffré)
cmk-agent-ctl register --hostname <NOM_HÔTE> --server 192.168.1.241 --site mkmonitor --user cmkadmin --password <MOT_DE_PASSE>
```

### Agent Windows

Télécharger le `.msi` depuis l'interface web Checkmk, installer, puis en invite de commande administrateur :

```powershell
cmk-agent-ctl register --hostname <NOM_HÔTE> --server 192.168.1.241 --site mkmonitor --user cmkadmin --password <MOT_DE_PASSE>
```

> 💡 Si la commande n'est pas reconnue, utiliser le chemin complet : `"C:\Program Files (x86)\checkmk\service\cmk-agent-ctl.exe" register ...`

### Découverte automatique des services

Une fois l'agent enregistré :

1. **Setup > Hosts > Run Service Discovery** (icône jaune 🟨)
2. Checkmk scanne et liste tous les services détectés (interfaces réseau, CPU, disques, services)
3. Valider avec **Accept all**
4. **Appliquer les changements** (bouton jaune en haut à droite → Activate on selected sites)

![discover](/images/2026-02-27-14-30-56.png)

![datasources](/images/2026-02-27-14-33-04.png)

![hosts](/images/2026-02-27-14-42-05.png)

---

## 4️⃣ Intégration API Proxmox VE

L'API Proxmox permet de superviser l'hyperviseur et ses VMs sans agent lourd sur chaque système.

### Côté Proxmox

1. **Créer un compte de service :** utilisateur `checkmk@pve` avec mot de passe classique
   - ⚠️ La sonde Checkmk utilise `/api2/json/access/ticket` qui attend un couple identifiant/mot de passe. Les API Tokens modernes provoquent une erreur.
2. **Droits d'accès :** rôle `PVEAuditor` (lecture seule) sur le chemin racine `/`
3. **Fuseau horaire :** dans `PVE-Server > System > Time`, resélectionner et valider le fuseau horaire
   - ⚠️ Sans cette validation, l'API renvoie un champ vide → crash du script Python Checkmk avec l'erreur `StopIteration`

![user](/images/2026-03-06-01-52-54.png)

### Côté Checkmk

1. **Nom d'hôte :** doit correspondre **exactement** au nom de l'hyperviseur (ex: `pve-server`, sensible à la casse)
2. **Règle Proxmox VE :** Setup > Agents > VM, cloud, container
   - Username : `checkmk@pve`
   - Password : *Explicit* + le mot de passe
   - SSL : cocher `Disable SSL certificate validation` **ET** la sous-option (certificat auto-signé Proxmox)
3. **Data sources :** forcer **Configured API integrations AND Checkmk agent** pour cumuler les métriques système et hyperviseur

![host](/images/2026-03-06-01-54-06.png)

![agents](/images/2026-03-06-01-59-43.png)

![activation](/images/2026-03-06-02-20-32.png)

### Ajout d'une VM via Piggyback

Pour les VMs supervisées indirectement via l'API Proxmox :

- **Host name :** nom exact de la VM dans Proxmox
- **IP address family :** `No IP`
- **Checkmk agent :** `No API integrations, no Checkmk agent`
- Laisser **Piggyback** sur la valeur par défaut (récupération silencieuse via l'API)

![add](/images/2026-03-06-12-32-10.png)

![VM](/images/2026-03-06-12-34-41.png)

---

## 5️⃣ Intégration pfSense via SNMPv3

### Configuration côté pfSense

1. Installer le paquet `net-snmp` : **System > Package Manager > Available Packages**
2. Activer le service : **Services > SNMP (NET-SNMP) > General > Enable**
3. Créer un utilisateur SNMPv3 : **Users > Add**
   - Username : `srv_checkmk`
   - Entry type : `User entry (USM)`
   - Access : `Read Only (GET, GETNEXT)` *(moindre privilège)*
   - Authentication : `SHA` + mot de passe
   - Privacy : `AES` + passphrase

4. **Règle de pare-feu :** Firewall > Rules > LAN
   - Pass | UDP | Source : IP Checkmk | Destination : This Firewall | Port : 161 (SNMP)

### Configuration côté Checkmk

1. **Add host :** `pfsense-fw01` avec l'IP de pfSense
2. **Data Sources :** No Checkmk agent + SNMP v2c or v3
3. **SNMP credentials :** SNMPv3 (authPriv) avec les identifiants configurés sur pfSense
4. **Run Service Discovery → Accept all → Activate**

![agent](/images/2026-03-08-00-27-00.png)

![OK](/images/2026-03-08-00-26-41.png)

> ⚠️ Vérifier le `timedatectl status` — un décalage horaire fera échouer les vérifications NTP.

---

## 📊 Dashboard personnalisé

Le tableau de bord est configuré avec des widgets pour visualiser l'état de l'infrastructure en un coup d'œil : état des hôtes, services critiques, bande passante des interfaces réseau, et alertes actives.

![dashboardcheckmk](/images/2026-06-08-00-48-01.png)

---

## 📚 Références

- Checkmk Raw Edition : <https://checkmk.com/product/raw-edition>
- Documentation complète : <https://docs.checkmk.com/latest/fr/>
- GitHub : <https://github.com/Checkmk/checkmk>
- Guide agents : <https://docs.checkmk.com/latest/fr/wato_monitoringagents.html>
