# 🔀 LAB : Déploiement Nginx Reverse Proxy — Point d'Entrée Unifié

```txt
 _   _       _            
| \ | | __ _(_)_ __ __  __
|  \| |/ _` | | '_ \\ \/ /
| |\  | (_| | | | | |>  < 
|_| \_|\__, |_|_| |_/_/\_\
       |___/               
```

![Type](https://img.shields.io/badge/Environnement-Conteneur%20LXC-FFA500?style=flat-square&logo=linux&logoColor=white)
![OS](https://img.shields.io/badge/OS-Debian%2013%20Trixie-A81D33?style=flat-square&logo=debian&logoColor=white)
![Service](https://img.shields.io/badge/Service-Nginx-009639?style=flat-square&logo=nginx&logoColor=white)
![Role](https://img.shields.io/badge/Rôle-Reverse%20Proxy-005085?style=flat-square)
![Network](https://img.shields.io/badge/Réseau-LAN-purple?style=flat-square&logo=web&logoColor=white)

**Rôle :** Administrateur d'Infrastructures Sécurisées

**Mission :** Déployer un **reverse proxy** Nginx en LXC dans la zone LAN, servant de point d'entrée unique pour tous les services du homelab. Le reverse proxy écoute sur le port 80 (HTTP standard) et redirige les requêtes vers le bon service backend en fonction du **nom de domaine** demandé (`homarr.home`, `plex.home`, etc.). Combiné aux réécritures DNS d'AdGuard Home, cette architecture permet à **tous les appareils du réseau** (PC, téléphones, tablettes WiFi) d'accéder aux services internes par un simple nom, sans retenir d'IP ni de port, et sans configurer de route statique sur chaque client.

> - [Documentation Officielle Nginx](https://nginx.org/en/docs/)
> - [Guide Reverse Proxy Nginx](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)

---

## L'intérêt technique 🎯

1. **Routage Couche 7 (Applicatif) :** Là où pfSense route en fonction du port TCP (couche 3/4), Nginx route en fonction du **header `Host:`** de la requête HTTP (couche 7). C'est l'équivalent applicatif du NAT/PAT — un seul point d'entrée, plusieurs services derrière.
2. **Pont LAN ↔ DMZ Transparent :** La Bbox FAI ne supporte pas les routes statiques. En plaçant Nginx dans le LAN, il fait office de passerelle applicative vers la DMZ : les clients LAN n'ont besoin d'aucune route spéciale, c'est Nginx qui possède la route vers `10.0.0.0/24` via pfSense.
3. **Suppression des Ports :** Sans reverse proxy, on accède aux services via `IP:port` (`10.0.0.20:7575`, `192.168.1.240:8006`). Avec Nginx, on tape simplement `homarr.home` ou `proxmox.home` — le port 80 étant le port HTTP par défaut, le navigateur ne l'affiche pas.
4. **Centralisation de la Sécurité :** Point unique pour appliquer des headers de sécurité (HSTS, X-Frame-Options), du rate limiting, et à terme un bouncer CrowdSec dédié pour la protection applicative.

---

## 🛠️ Architecture du Lab

- **Environnement :** Serveur Proxmox VE (HP ProDesk 600 G4)
- **Conteneur :** LXC Debian 13 (Trixie) — Non privilégié (Unprivileged)
- **IP :** `192.168.1.244/24` (Zone LAN, sur `vmbr0`)
- **Gateway :** `192.168.1.254` (Box FAI)
- **DNS :** `192.168.1.250` (AdGuard Home)
- **Ressources :** 1 vCPU, 256 Mo RAM, 4 Go disque

### Flux Réseau

```txt
┌─────────────────────────────────────────────────────────────────────┐
│  Appareil WiFi / PC LAN                                             │
│  Tape : homarr.home                                                 │
└──────────────────────┬──────────────────────────────────────────────┘
                       │ Requête DNS
                       ▼
┌──────────────────────────────────┐
│  AdGuard Home (192.168.1.250)    │
│  Réécriture DNS :                │
│  homarr.home → 192.168.1.244    │
└──────────────────────┬───────────┘
                       │ Requête HTTP (port 80)
                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Nginx Reverse Proxy (192.168.1.244) — LXC LAN                      │
│                                                                      │
│  server_name homarr.home   → proxy_pass http://10.0.0.20:7575       │
│  server_name plex.home     → proxy_pass http://10.0.0.10:32400      │
│  server_name scanopy.home  → proxy_pass http://10.0.0.20:60072      │
│  server_name proxmox.home  → proxy_pass https://192.168.1.240:8006  │
│  server_name checkmk.home  → proxy_pass http://192.168.1.241:5000   │
│  server_name adguard.home  → proxy_pass http://192.168.1.250:3000   │
│                                                                      │
│  Route statique : 10.0.0.0/24 via 192.168.1.251 (pfSense)          │
└──────────────────────────────────────────────────────────────────────┘
```

---

> Documentation : <https://nginx.org/en/docs/>

---

## Prérequis

- Proxmox VE opérationnel avec template Debian 13
- pfSense (`192.168.1.251`) opérationnel avec accès DMZ
- AdGuard Home (`192.168.1.250`) en tant que serveur DNS du LAN
- Règle pfSense autorisant le trafic de `192.168.1.244` vers `10.0.0.0/24` (Firewall > Rules > WAN)

---

## 1️⃣ Création du Conteneur LXC

Depuis l'interface Proxmox, créer un nouveau conteneur :

```sh
CT ID : 104
Hostname : nginx-proxy
Template : debian-13-standard
Disque : 4 Go
CPU : 1 cœur
RAM : 256 Mo
Réseau : vmbr0 (LAN)
IPv4 : 192.168.1.244/24
Gateway : 192.168.1.254
DNS : 192.168.1.250
Unprivileged : ✅
Start at boot : ✅
```

---

## 2️⃣ Installation de Nginx

Se connecter au LXC et installer Nginx :

```bash
apt update && apt install -y nginx
systemctl enable nginx
systemctl start nginx
```

Vérifier que Nginx répond : ouvrir `http://192.168.1.244` depuis un navigateur → la page par défaut Nginx doit s'afficher.

![welcome](/images/2026-04-06-13-22-15.png)

---

## 3️⃣ Configuration de la Route vers la DMZ

Le LXC Nginx doit pouvoir joindre les services en DMZ (`10.0.0.0/24`) via pfSense. Ajouter la route dans la configuration réseau persistante :

```bash
nano /etc/network/interfaces
```

```sh
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
        address 192.168.1.244/24
        gateway 192.168.1.254
        up ip route add 10.0.0.0/24 via 192.168.1.251
```

Injecter la route à chaud (sans reboot) :

```bash
ip route add 10.0.0.0/24 via 192.168.1.251
```

Vérifier la connectivité vers la DMZ :

```bash
ping 10.0.0.20   # VM Docker (Homarr, Scanopy)
ping 10.0.0.10   # Plex
```

> ⚠️ Si le ping ne passe pas, vérifier qu'une règle pfSense autorise le flux :
> **Firewall > Rules > WAN** → Pass | TCP/ICMP | Source `192.168.1.244` | Destination `10.0.0.0/24`

---

## 4️⃣ Configuration des Virtual Hosts (Reverse Proxy)

Supprimer le site par défaut :

```bash
rm /etc/nginx/sites-enabled/default
```

### Homarr (`homarr.home`)

```bash
cat > /etc/nginx/sites-available/homarr.home << 'EOF'
server {
    listen 80;
    server_name homarr.home;

    location / {
        proxy_pass http://10.0.0.20:7575;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        # Support WebSocket (nécessaire pour Homarr)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
EOF
```

### Plex (`plex.home`)

```bash
cat > /etc/nginx/sites-available/plex.home << 'EOF'
server {
    listen 80;
    server_name plex.home;

    location / {
        proxy_pass http://10.0.0.10:32400;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
EOF
```

### Scanopy (`scanopy.home`)

```bash
cat > /etc/nginx/sites-available/scanopy.home << 'EOF'
server {
    listen 80;
    server_name scanopy.home;

    location / {
        proxy_pass http://10.0.0.20:60072;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
```

### Proxmox (`proxmox.home`)

```bash
cat > /etc/nginx/sites-available/proxmox.home << 'EOF'
server {
    listen 80;
    server_name proxmox.home;

    location / {
        # Proxmox utilise HTTPS nativement — proxy_ssl_verify off
        # car le certificat est auto-signé
        proxy_pass https://192.168.1.240:8006;
        proxy_ssl_verify off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        # WebSocket support (console noVNC de Proxmox)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
EOF
```

### Checkmk (`checkmk.home`)

```bash
cat > /etc/nginx/sites-available/checkmk.home << 'EOF'
server {
    listen 80;
    server_name checkmk.home;

    location / {
        proxy_pass http://192.168.1.241:5000/mkmonitor/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
```

### AdGuard Home (`adguard.home`)

```bash
cat > /etc/nginx/sites-available/adguard.home << 'EOF'
server {
    listen 80;
    server_name adguard.home;

    location / {
        proxy_pass http://192.168.1.250:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
```

---

## 5️⃣ Activation et Validation

Activer tous les virtual hosts via des liens symboliques :

```bash
ln -s /etc/nginx/sites-available/homarr.home /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/plex.home /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/scanopy.home /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/proxmox.home /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/checkmk.home /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/adguard.home /etc/nginx/sites-enabled/
```

Tester la syntaxe et recharger :

```bash
# Vérification de la syntaxe (doit afficher "syntax is ok")
nginx -t

# Rechargement à chaud (sans coupure)
systemctl reload nginx
```

![syntaxOK](/images/2026-04-06-13-25-23.png)

---

## 6️⃣ Configuration DNS (AdGuard Home)

Dans l'interface AdGuard Home (`http://192.168.1.250:3000`) → **Filtres > Réécriture DNS**, toutes les entrées doivent pointer vers l'IP du reverse proxy :

| Domaine | Réponse |
| --- | --- |
| `homarr.home` | `192.168.1.244` |
| `plex.home` | `192.168.1.244` |
| `scanopy.home` | `192.168.1.244` |
| `proxmox.home` | `192.168.1.244` |
| `checkmk.home` | `192.168.1.244` |
| `adguard.home` | `192.168.1.244` |
| `pfsense.home` | `192.168.1.251` |

> ℹ️ `pfsense.home` reste pointé directement vers pfSense (`192.168.1.251`) car l'interface web de pfSense utilise HTTPS avec un certificat lié à son IP — le reverse proxy n'apporte pas de valeur ajoutée ici.

---

## 🎯 Validation de l'Installation

Depuis **n'importe quel appareil** connecté au réseau WiFi/LAN :

```sh
http://homarr.home    → Dashboard Homarr ✅
http://plex.home      → Interface web Plex ✅
http://proxmox.home   → Interface Proxmox ✅
http://checkmk.home   → Dashboard Checkmk ✅
http://scanopy.home   → Interface Scanopy ✅
http://adguard.home   → Interface AdGuard Home ✅
```

Aucun port à retenir, aucune route à configurer sur les clients. Le DNS résout vers Nginx, Nginx proxy vers le bon service.

---

## ⚙️ Commandes Utiles (Aide-Mémoire)

```bash
# --- Gestion du service ---
systemctl start|stop|restart|reload nginx
systemctl status nginx

# --- Vérification de la configuration ---
nginx -t                              # Test syntaxe des vhosts
nginx -T                              # Affiche la config complète compilée

# --- Logs ---
tail -f /var/log/nginx/access.log     # Requêtes en temps réel
tail -f /var/log/nginx/error.log      # Erreurs

# --- Gestion des sites ---
ls /etc/nginx/sites-available/        # Sites configurés
ls /etc/nginx/sites-enabled/          # Sites actifs (liens symboliques)

# --- Ajouter un nouveau service ---
# 1. Créer le fichier dans /etc/nginx/sites-available/nouveau.home
# 2. ln -s /etc/nginx/sites-available/nouveau.home /etc/nginx/sites-enabled/
# 3. nginx -t && systemctl reload nginx
# 4. Ajouter la réécriture DNS dans AdGuard Home → 192.168.1.244
```

---

## 🔐 Points de Sécurité

- **Exposition limitée au LAN :** Nginx n'écoute que sur le réseau local (`192.168.1.0/24`). Aucun port n'est ouvert sur Internet — le reverse proxy n'est pas exposé côté WAN.
- **Headers de sécurité :** Pour un durcissement futur, il est possible d'ajouter des headers dans chaque bloc `server` :

  ```nginx
  add_header X-Frame-Options "SAMEORIGIN" always;
  add_header X-Content-Type-Options "nosniff" always;
  add_header X-XSS-Protection "1; mode=block" always;
  ```

- **Proxy SSL Verify Off :** Utilisé pour Proxmox car son certificat est auto-signé. En production, il faudrait déployer un certificat CA interne et activer la vérification.
- **CrowdSec :** Un bouncer Nginx pourra être ajouté ultérieurement pour analyser les requêtes HTTP et bloquer les comportements malveillants au niveau applicatif — en complément du bouncer pfSense qui agit au niveau réseau.
- **Limitation de la Bbox :** Cette architecture (Nginx LAN + DNS AdGuard) est un contournement de l'impossibilité de configurer des routes statiques sur la Bbox Bouygues (F@st 5688b). Le reverse proxy fait office de passerelle applicative entre le LAN et la DMZ.

---

## 📝 Ajout d'un Nouveau Service

Pour chaque nouveau service à proxyfier :

1. Créer le fichier vhost :

   ```bash
   nano /etc/nginx/sites-available/nouveau.home
   ```

2. Activer le site :

   ```bash
   ln -s /etc/nginx/sites-available/nouveau.home /etc/nginx/sites-enabled/
   ```

3. Tester et recharger :

   ```bash
   nginx -t && systemctl reload nginx
   ```

4. Ajouter la réécriture DNS dans AdGuard Home : `nouveau.home → 192.168.1.244`
