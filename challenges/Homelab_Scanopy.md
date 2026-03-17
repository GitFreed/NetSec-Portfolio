# 📊 LAB : Cartographie Réseau et IPAM (Gestion des Adresses IP)

```txt
 _____                                   
/  ___|                                  
\ `--.  ___ __ _ _ __   ___  _ __  _   _ 
 `--. \/ __/ _` | '_ \ / _ \| '_ \| | | |
/\__/ / (_| (_| | | | | (_) | |_) | |_| |
\____/ \___\__,_|_| |_|\___/| .__/ \__, |
                            | |     __/ |
                            |_|    |___/ 
```

![Type](https://img.shields.io/badge/Environnement-Docker%20%26%20LXC-2496ED?style=flat-square&logo=docker&logoColor=white)
![OS](https://img.shields.io/badge/OS-Debian%2013%20Trixie-A81D33?style=flat-square&logo=debian&logoColor=white)
![Service](https://img.shields.io/badge/Service-Scanopy-1ABC9C?style=flat-square)
![Role](https://img.shields.io/badge/Rôle-Cartographie%20%26%20IPAM-005085?style=flat-square)
![Network](https://img.shields.io/badge/Réseau-DMZ%20%26%20LAN-purple?style=flat-square&logo=web&logoColor=white)

**Rôle :** Administrateur Réseau / Ingénieur DevOps

**Mission :** Déployer Scanopy, une solution moderne de cartographie réseau distribuée. Contrairement à un simple scanner, Scanopy agit comme un annuaire IPAM dynamique. Il s'appuie sur une architecture décentralisée (un serveur central + des démons déportés) pour scanner simultanément plusieurs sous-réseaux isolés (VLANs, DMZ, LAN physique). Il utilise des requêtes de Niveau 2 (ARP) et Niveau 3/4 (ICMP, TCP, UDP, SNMP) pour découvrir les équipements, identifier les ports ouverts, et construire une topologie visuelle interactive de l'ensemble du système d'information.

> - [Github Scanopy](https://github.com/scanopy/scanopy)
> - [Documentation Officielle](https://scanopy.net/docs/)

---

## L'intérêt technique 🎯

1. **Visibilité Couche 2 (ARP) :** Déployer des démons locaux sur chaque sous-réseau permet de récupérer les adresses MAC et de détecter les hôtes furtifs qui bloquent le ping (ICMP), chose impossible au travers d'un routeur.
2. **Architecture Zero Trust :** Le serveur central est isolé en DMZ. Les sondes communiquent avec lui via un modèle *Polling* (ce sont les sondes qui initient la connexion, le serveur n'ouvre aucun flux vers l'extérieur) en utilisant des jetons d'authentification uniques (API Tokens).
3. **Sécurité par Design (Moindre Privilège) :** Abandon du mode Docker privilégié (`privileged: true`) au profit d'une élévation de privilèges granulaire via les `Linux Capabilities` (`NET_RAW`, `NET_ADMIN`).

---

## 🛠️ Architecture du Lab

- **Environnement :** Serveur Proxmox VE
- **Serveur Central (UI + API + DB) :** Machine Virtuelle Docker (Debian 13)
  - IP : `10.0.0.20` (Zone DMZ, sur `vmbr2`)
  - Port d'écoute API/Web : `60072` TCP
- **Sonde Locale (DMZ) :** Conteneur Docker tournant sur la même VM (`10.0.0.20`), chargé de scanner la zone 10.0.x.x.
- **Sonde Déportée (LAN) :** Conteneur LXC (Debian 13) léger dédié au scan du réseau physique.
  - IP : `192.168.1.243` (Zone LAN, sur `vmbr0`)
- **Routeur / Pare-feu :** pfSense (`192.168.1.251` côté WAN)

---

## 1️⃣ Installation du Serveur Core (Docker)

Le serveur central et sa base de données sont déployés via Docker Compose sur la VM `10.0.0.20`.
Le fichier intègre également une sonde locale pour scanner la DMZ.

### Configuration du fichier d'orchestration

`nano docker-compose.yml`

> ⚠️ **Alerte Cybersécurité :** Le bloc `daemon` officiel utilise `privileged: true`. Nous le remplaçons par les `cap_add` strictement nécessaires pour forger des trames réseau, sécurisant ainsi la VM hôte.

```yaml
name: scanopy

services:
  # --- LA BASE DE DONNEES ---
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: scanopy
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-password}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      - scanopy

  # --- LE SERVEUR CENTRAL ---
  server:
    image: ghcr.io/scanopy/scanopy/server:latest
    ports:
      - "60072:60072"
    environment:
      SCANOPY_LOG_LEVEL: ${SCANOPY_LOG_LEVEL:-info}
      SCANOPY_DATABASE_URL: postgresql://postgres:${POSTGRES_PASSWORD:-password}@postgres:5432/scanopy
      SCANOPY_WEB_EXTERNAL_PATH: /app/static
      SCANOPY_PUBLIC_URL: ${SCANOPY_PUBLIC_URL:-http://10.0.0.20:60072}
      SCANOPY_INTEGRATED_DAEMON_URL: http://host.docker.internal:60073
    volumes:
      - ./data:/data
    extra_hosts:
      - "host.docker.internal:host-gateway"
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - scanopy

  # --- LA SONDE LOCALE (DMZ) ---
  daemon:
    image: ghcr.io/scanopy/scanopy/daemon:latest
    container_name: scanopy-daemon
    network_mode: host # Obligatoire pour le scan ARP
    cap_add:           # Remplace "privileged: true" (Moindre Privilège)
      - NET_RAW
      - NET_ADMIN
    restart: unless-stopped
    environment:
      SCANOPY_LOG_LEVEL: ${SCANOPY_LOG_LEVEL:-info}
      SCANOPY_SERVER_URL: http://127.0.0.1:60072
    volumes:
      - daemon-config:/root/.config/daemon
      - /var/run/docker.sock:/var/run/docker.sock:ro # Lecture seule des conteneurs

volumes:
  postgres_data:
  daemon-config:

networks:
  scanopy:
    driver: bridge
```

Démarrage de l'infrastructure :
`docker compose up -d`

L'interface Web est désormais accessible sur `http://10.0.0.20:60072`.

---

## 2️⃣ Configuration IPAM (Interface Web)

Avant de déployer une sonde sur un autre sous-réseau, il faut déclarer ce réseau dans l'application pour générer une identité sécurisée.

1. **Création du Réseau Logique :**
      - Aller dans **Networks** \> Add Network.
      - Nom : `Network LAN`.
2. **Création du Sous-réseau Physique :**
      - Aller dans **Subnets** \> Add Subnet.
      - Type : `LAN`.
      - CIDR : `192.168.1.0/24`.
      - L'attacher au parent `Network LAN`.
3. **Génération de l'Identité du Démon :**
      - Aller dans **Daemons** \> Create a daemon.
      - Sélectionner le réseau `Network LAN`.
      - Nom : `scanopy-daemon-network-lan`.
      - **Copier la commande d'installation et le Token API (Clé).**

---

## 3️⃣ Déploiement de la Sonde Déportée (LXC)

Création d'un conteneur LXC Debian 13 (Trixie) non privilégié (Unprivileged) sur l'hyperviseur Proxmox.

- **CPU :** 1 Cœur | **RAM :** 512 Mo | **Disque :** 2 Go
- **Réseau :** `vmbr0` | **IP :** `192.168.1.243/24` | **Gateway :** `192.168.1.254`

### A. Le Franchissement de Pare-feu (pfSense & Routage)

La sonde LAN (`192.168.1.0/24`) doit contacter le serveur DMZ (`10.0.0.0/24`). La Bbox domestique ne gérant pas le routage interne, nous appliquons un **Routage Statique Local (Host-based routing)**.

`nano /etc/network/interfaces`
Ajouter la ligne d'interface `up` pour forcer le trafic DMZ vers pfSense :

```text
auto eth0
iface eth0 inet static
    address 192.168.1.243/24
    gateway 192.168.1.254
    up ip route add 10.0.0.0/24 via 192.168.1.251
```

Reboot le conteneur ou injecter à chaud : `ip route add 10.0.0.0/24 via 192.168.1.251`.

**Sur l'interface de pfSense (Firewall \> Rules \> WAN) :**
Créer une règle autorisant le flux :

- Action: **Pass** | Protocol: **TCP** | Source: **192.168.1.243** | Destination: **10.0.0.20** | Port: **60072**

### B. Installation et Configuration du Service Systemd

Installation des prérequis et lancement du script officiel (en modifiant l'URL localhost par l'IP de la DMZ) :

```bash
apt update && apt install -y curl sudo

bash -c "$(curl -fsSL https://raw.githubusercontent.com/scanopy/scanopy/refs/heads/main/install.sh)" && sudo scanopy-daemon --server-url http://10.0.0.20:60072 --network-id VOTRE_ID --daemon-api-key VOTRE_TOKEN --user-id VOTRE_USER_ID --name scanopy-daemon-network-lan --mode daemon_poll
```

> ⚙️ **Optimisation Sysadmin (Fiabilisation)**
> Le script installe le binaire mais les variables d'environnement peuvent sauter au redémarrage. On configure le fichier Systemd en dur. De plus, la syntaxe exige un format strict (`daemon_poll` et non `DaemonPoll`).

`nano /etc/systemd/system/scanopy-daemon.service`

Vider le bloc `[Service]` et le remplacer par l'exécution directe :

```ini
[Unit]
Description=Scanopy Network Discovery Daemon
After=network.target

[Service]
Type=simple
# Exécution stricte avec les variables issues de l'interface Web
ExecStart=/usr/local/bin/scanopy-daemon --server-url http://10.0.0.20:60072 --network-id VOTRE_ID --daemon-api-key VOTRE_TOKEN --user-id VOTRE_USER_ID --name scanopy-daemon-network-lan --mode daemon_poll
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**Démarrage final :**

```bash
systemctl daemon-reload
systemctl restart scanopy-daemon
journalctl -fu scanopy-daemon
```

Dans les journaux, le message `Connected to server` confirmera que le routage, le pare-feu et l'authentification sont fonctionnels. La découverte réseau démarre automatiquement sur l'interface Web \! 🎉

![connecting](/images/2026-03-17-12-46-54.png)

![daemons](/images/2026-03-17-13-25-01.png)
