# Challenge C501 16/03/2026

## 🧑‍🏫 Pitch de l’exercice

Challenge : <.>

[Cours C501.](/RESUME.md#-c501)

> 📚 **Ressources** :
>
> - Kali Docs : <https://www.kali.org/docs/>

---

## Déploiement VM Kali Linux (Pentest Ready)

**Objectif :** Disposer d'une machine d'audit de sécurité isolée et performante dans le réseau `10.0.0.0/24`.

### 1. Configuration Proxmox (Hardware)

- **Options :** Activer "**QEMU Guest Agent**"
- **Processeur :** 2-4 vCPUs (Type : **host**)
- **Mémoire :** 4 Go (4096 MiB)
- **Réseau :** Bridge `vmbr2` (LAN Isolé derrière pfSense)
- **Disque :** 50 Go (Bus : VirtIO / **Discard** : On / **SSD Emulation** : On)

### 2. Post-Installation

Une fois l'installation terminée, on doit préparer le système pour qu'il soit opérationnel et sécurisé.

```bash
# MISE À JOUR DU SYSTÈME
sudo apt update && sudo apt upgrade -y

# OPTIMISATION PROXMOX
sudo apt install qemu-guest-agent -y
sudo systemctl enable --now qemu-guest-agent

# CONFIGURATION RÉSEAU & ACCÈS SSH
# Configuration de l'interface Ethernet en statique
auto eth0
iface eth0 inet static
    address 10.0.0.30/24
    gateway 10.0.0.1
    dns-nameservers 10.0.0.1 9.9.9.9

sudo systemctl restart networking
ip a

# Activer SSH pour l'administration distante
sudo apt install openssh-server -y
sudo systemctl enable --now ssh

# NETTOYAGE
sudo apt autoremove -y && sudo apt clean

```

---

## Déploiement DVWA via Docker

**Objectif :** Fournir un environnement d'entraînement légal et cloisonné pour pratiquer l'identification, l'exploitation et la remédiation des vulnérabilités web critiques (OWASP Top 10). Il permet de monter en compétences sur des techniques d'attaque réelles sans risquer d'endommager des systèmes de production.

**Pré-requis :** Avoir installé Docker et le plugin Compose sur la machine cible (`sudo apt install docker.io docker-compose-v2 -y`).

### 1. Récupération et Configuration

On récupère les sources et on prépare le fichier de déploiement.

```bash
# Récupération du dépôt
git clone https://github.com/digininja/DVWA.git
cd DVWA/

# Création/Édition du fichier compose.yml
nano compose.yml

```

### 2. Contenu du fichier `compose.yml`

Voici une configuration propre pour un réseau `10.0.0.0/24`. On mappe le port **4280** pour l'accès web.

```yaml
services:
  dvwa:
    image: vulnerables/web-dvwa
    ports:
      - "4280:80"
    restart: always

```

### 3. Lancement du service

```bash
# Lancement en arrière-plan
docker compose up -d

# Vérification que le conteneur tourne
docker ps
```

![docker](/images/2026-03-16-12-41-43.png)

### 3. Premier accès et Configuration

- **URL d'accès :** `http://10.0.0.30:4280/login.php`
- **Identifiants par défaut :** `admin` / `admin`.
- **Initialisation :** Une fois connecté, il faut impérativement cliquer sur le bouton **"Create / Reset Database"** en bas de page pour générer les tables MySQL.
- **Identifiants :** `admin` / `password`.
- **Sécurité :** Pour débuter tes tests d'intrusion (Pentest), aller dans l'onglet **"DVWA Security"** et règler le niveau sur **"Low"**.

![DVWA](/images/2026-03-16-12-08-55.png)

---

## Burpsuite PortSwigger

<https://portswigger.net/>
