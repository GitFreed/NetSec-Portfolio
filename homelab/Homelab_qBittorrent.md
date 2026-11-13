# 🧲 LAB : Seedbox VPN avec qBittorrent et Gluetun en Conteneur LXC

```txt
       ____  _ _   _                            _   
  __ _| __ )(_) |_| |_ ___  _ __ _ __ ___ _ __ | |_ 
 / _` |  _ \| | __| __/ _ \| '__| '__/ _ \ '_ \| __|
| (_| | |_) | | |_| || (_) | |  | | |  __/ | | | |_ 
 \__, |____/|_|\__|\__\___/|_|  |_|  \___|_| |_|\__|
    |_|                                             
```

![Type](https://img.shields.io/badge/Environnement-LXC%20%2B%20Docker-FFA500?style=flat-square&logo=linux&logoColor=white)
![OS](https://img.shields.io/badge/OS-Debian%2013%20Trixie-A81D33?style=flat-square&logo=debian&logoColor=white)
![Service](https://img.shields.io/badge/Service-qBittorrent%205.3.0-2F67BA?style=flat-square&logo=qbittorrent&logoColor=white)
![VPN](https://img.shields.io/badge/VPN-Proton%20WireGuard-6D4AFF?style=flat-square&logo=protonvpn&logoColor=white)
![Role](https://img.shields.io/badge/Rôle-Seedbox%20%26%20Kill%20Switch-005085?style=flat-square)
![Network](https://img.shields.io/badge/Réseau-DMZ-purple?style=flat-square&logo=web&logoColor=white)

**Rôle :** Administrateur d'Infrastructures Sécurisées / DevOps

**Mission :** Déployer une seedbox isolée en DMZ, capable de partager la bibliothèque multimédia existante via BitTorrent **exclusivement à travers un tunnel VPN WireGuard** (Proton VPN). Le service doit garantir qu'aucun paquet ne quitte le lab hors du tunnel, y compris en cas de coupure du VPN (**kill switch**), sans jamais disposer d'un droit d'écriture sur la bibliothèque. La redirection de port fournie par le VPN est récupérée et appliquée automatiquement à chaque reconnexion, afin que le client reste joignable par les autres pairs.

> - [Gluetun (GitHub)](https://github.com/qdm12/gluetun)
> - [Image linuxserver/qbittorrent](https://docs.linuxserver.io/images/docker-qbittorrent/)

---

## L'intérêt technique 🎯

1. **Défense en profondeur (double kill switch) :** Deux barrières indépendantes empêchent toute fuite hors tunnel. Le pare-feu interne de Gluetun n'autorise que le tunnel WireGuard, et pfSense n'autorise le conteneur à sortir que vers le port UDP 51820. Si l'une des deux couches est mal configurée ou contournée, l'autre bloque.
2. **Moindre privilège :** LXC **non privilégié**, bibliothèque montée en **lecture seule** au niveau de l'hyperviseur (`ro=1`, impossible à lever depuis le conteneur), WebUI joignable uniquement depuis le LAN, aucun port torrent exposé sur la Box.
3. **Ingénierie réseau :** Filtrage pfSense par **alias** (hôte, plages RFC 1918, groupe de ports) et logique « Internet uniquement » par **négation** (`! RFC1918`), qui interdit au conteneur de joindre le LAN comme le reste de la DMZ.
4. **DevOps (Docker Compose) :** Pile déclarative où qBittorrent partage la pile réseau de Gluetun (`network_mode: service:gluetun`), démarrage conditionné à la santé du tunnel, et automatisation du **port forwarding NAT-PMP** via l'API Web de qBittorrent.
5. **Démarche de validation :** Chaque garantie de sécurité est prouvée par un test reproductible (IP de sortie, port annoncé aux pairs, coupure volontaire du tunnel).

---

## 🛠️ Architecture du Lab

- **Hôte :** Proxmox VE (`192.168.1.240`)
- **Stockage :** HDD 6 To externe monté sur l'hôte (`/mnt/pve/HDD-plex`), partagé avec [Plex](./Homelab_Plex.md)
- **Nœud applicatif :** LXC `1012` `qbit-srv`, Debian 13, **Unprivileged**
  - IP : `10.0.0.12` (Zone DMZ, sur `vmbr2`)
  - CPU : 2 vCPU | RAM : 2 Go | Disque : 8 Go
- **Passerelle et DNS :** pfSense `10.0.0.1`
- **Conteneurs Docker :** `gluetun` (client VPN + pare-feu) et `qbittorrent` (client BitTorrent + WebUI `:8080`)
- **Accès admin :** `https://torrent.home` via le reverse proxy [Nginx](./Homelab_Nginx.md) (LAN uniquement)

### Flux réseau

```txt
                    [Pairs BitTorrent / Trackers]
                                ▲
                                │  (tout le trafic BitTorrent est DANS le tunnel)
                     [Serveur Proton VPN P2P] ◄─ port redirigé (NAT-PMP)
                                ▲
                                │  WireGuard UDP 51820 — seul flux sortant autorisé
                                │
[Box FAI] ◄── [pfSense VM] ── Règles QBIT_SRV : DNS → 10.0.0.1 | UDP 51820 → !RFC1918
                  │                             TCP 80/443 → !RFC1918 | BLOCK + LOG le reste
                  │ vmbr2 (DMZ 10.0.0.0/24)
                  ▼
     [LXC qbit-srv - 10.0.0.12] (Unprivileged)
         ├─> gluetun ── tun0 + pare-feu interne (kill switch n°1)
         │     └─> qBittorrent (network_mode: service:gluetun) — WebUI :8080
         └─> /medias  ◄── bind mount ro=1 ── /mnt/pve/HDD-plex/Medias (hôte)

[PC LAN] ── torrent.home (rewrite AdGuard) ──> [Nginx 192.168.1.244] ──> 10.0.0.12:8080
```

> 💡 Dans ce lab, l'interface pfSense nommée **« LAN »** porte la DMZ (`10.0.0.1`) et l'interface **« WAN »** fait face au LAN domestique (`192.168.1.251`). Les règles du conteneur se créent donc dans l'onglet **LAN** de pfSense.

---

## 1️⃣ Préparation de l'Hôte (Proxmox)

Gluetun s'appuie sur le module noyau WireGuard. Un conteneur LXC partageant le noyau de l'hôte, le module doit être chargé sur Proxmox.

```bash
# Charger le module immédiatement
modprobe wireguard

# Le rendre persistant au redémarrage
echo wireguard > /etc/modules-load.d/wireguard.conf

# Vérification : une ligne "wireguard" doit apparaître
lsmod | grep wireguard
```

---

## 2️⃣ Création du Conteneur et Bind Mount

### Déploiement du LXC

```bash
pct create 1012 HDD-data:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst \
  --hostname qbit-srv \
  --unprivileged 1 \
  --features nesting=1,keyctl=1 \
  --cores 2 --memory 2048 --swap 512 \
  --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr2,ip=10.0.0.12/24,gw=10.0.0.1 \
  --nameserver 10.0.0.1 \
  --onboot 1 \
  --password
```

- `nesting=1,keyctl=1` : requis pour faire tourner Docker dans un LXC non privilégié.
- Le template n'a pas besoin d'être à jour : un `apt full-upgrade` est appliqué au premier démarrage.

### Bibliothèque en lecture seule

```bash
# La bibliothèque est montée en lecture seule, verrouillée côté hyperviseur
pct set 1012 -mp0 /mnt/pve/HDD-plex/Medias,mp=/medias,ro=1
```

> ⚠️ **Moindre Privilège :** le client BitTorrent est le service le plus exposé du lab (il dialogue avec des pairs inconnus). Même en cas de compromission complète du conteneur, la bibliothèque ne peut être ni modifiée, ni chiffrée par un rançongiciel, ni supprimée : le `ro=1` est appliqué par l'hôte.

### Accès au périphérique TUN

```bash
# Gluetun crée l'interface tun0 : le conteneur doit accéder à /dev/net/tun
cat >> /etc/pve/lxc/1012.conf <<'EOF'
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
EOF

pct start 1012
```

---

## 3️⃣ Kill Switch Réseau (pfSense)

Le conteneur naît verrouillé : les règles sont créées **avant** l'installation de Docker.

### Alias (Firewall > Aliases)

| Onglet | Nom | Type | Contenu | Description |
| --- | --- | --- | --- | --- |
| IP | `QBIT_SRV` | Host(s) | `10.0.0.12` | LXC 1012 qbit-srv (qBittorrent + gluetun) en DMZ |
| IP | `RFC1918` | **Network(s)** | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | Plages IP privées : jamais Internet |
| Ports | `WEB_PORTS` | Port(s) | `80`, `443` | HTTP/HTTPS pour apt et images Docker |

> 💡 Un **alias** est l'équivalent d'un *object-group* Cisco : il rend les règles lisibles, se modifie en un seul endroit, et permet d'exprimer plusieurs plages dans une seule règle.

### Règles (Firewall > Rules > LAN)

Placées **sous l'Anti-Lockout et au-dessus de « Default allow LAN to any »**, dans cet ordre (pfSense applique la première règle correspondante) :

| # | Action | Proto | Source | Destination | Port | Description |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Pass | TCP/UDP | `QBIT_SRV` | LAN address | 53 | qbit DNS |
| 2 | Pass | UDP | `QBIT_SRV` | `! RFC1918` | 51820 | qbit WireGuard Proton |
| 3 | Pass | TCP | `QBIT_SRV` | `! RFC1918` | `WEB_PORTS` | qbit apt + Docker |
| 4 | **Block** + Log | Any | `QBIT_SRV` | any | any | qbit kill switch |

- La règle 3 ne sert qu'aux mises à jour système et aux images Docker. Elle peut être désactivée hors maintenance pour durcir encore le conteneur.
- La règle 4 bloque tout le reste : LAN domestique, autres hôtes de la DMZ, et Internet hors tunnel.

### Vérifier ce qui est réellement appliqué

```sh
# pfSense > Diagnostics > Command Prompt
pfctl -sr | grep -i qbit      # la règle "block ... qbit kill switch" doit être la DERNIÈRE
pfctl -t QBIT_SRV -T show     # contenu effectif de l'alias
```

> ⚠️ Après un réordonnancement par glisser-déposer, cliquer sur **Save** en bas de la liste **avant** Apply Changes. Sans ce Save, la page web affiche le bon ordre mais pf charge l'ancien (voir *Problèmes rencontrés*).

### Validation du filtrage

```bash
# Dans le LXC
apt update                       # doit passer (règles 1 et 3)
ping -c 2 -W 2 192.168.1.254     # doit échouer (LAN bloqué par la règle 4)
ping -c 2 -W 2 1.1.1.1           # doit échouer (ICMP hors tunnel bloqué)
```

---

## 4️⃣ Installation de Docker

```bash
# Mises à jour de sécurité automatiques
apt update && apt full-upgrade -y
apt install -y curl ca-certificates unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades

# Docker via le script officiel (le relire avant exécution)
curl -fsSL https://get.docker.com -o get-docker.sh
less get-docker.sh
sh get-docker.sh

# Vérification
docker run --rm hello-world
ls /medias
```

> 💡 **Choix d'architecture :** Proxmox recommande une VM pour Docker. Le LXC a été retenu ici pour trois raisons : le `ro=1` d'un bind mount n'est pas levable depuis le conteneur (une VM imposerait virtiofs, NFS ou SMB), le kill switch pfSense repose sur une IP dédiée au seul client torrent, et le service le plus exposé reste séparé de la VM Docker qui héberge [Homarr](./Homelab_Homarr.md) et ses clés API. La contrepartie (Docker dans un LXC est plus sensible aux mises à jour de runc/containerd) est couverte par une sauvegarde `vzdump` avant chaque mise à jour.

---

## 5️⃣ Configuration Proton VPN

Sur le compte Proton (offre Plus, requise pour le P2P et la redirection de port) : **Téléchargements > Configuration WireGuard**

| Paramètre | Valeur |
| --- | --- |
| Plateforme | GNU/Linux |
| NAT-PMP (Port Forwarding) | ✅ Activé |
| NAT modéré | ❌ Désactivé |
| Serveur | Un serveur marqué P2P |

Seule la valeur `PrivateKey` du fichier généré est conservée :

```bash
mkdir -p /opt/torrent && cd /opt/torrent
nano .env          # une ligne : WIREGUARD_PRIVATE_KEY=<clé>
chmod 600 .env     # lisible par root uniquement
```

> ⚠️ **Secret :** la clé n'apparaît ni dans le `docker-compose.yml`, ni dans un dépôt Git. Elle est aussi présente dans les sauvegardes `vzdump` du conteneur, à ne jamais copier hors du lab sans chiffrement.

---

## 6️⃣ Déploiement (Docker Compose)

```yaml
services:
  gluetun:
    image: qmcgaw/gluetun:v3                      # version majeure figée
    container_name: gluetun
    cap_add:
      - NET_ADMIN                                 # gestion de tun0 et des règles iptables
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - VPN_SERVICE_PROVIDER=protonvpn
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY}
      - SERVER_COUNTRIES=Switzerland,Netherlands
      - PORT_FORWARD_ONLY=on                      # uniquement des serveurs avec redirection de port
      - VPN_PORT_FORWARDING=on
      - FIREWALL_INPUT_PORTS=8080                 # WebUI joignable depuis Nginx
      # Pousse le port attribué + l'interface tun0 dans qBittorrent à chaque (re)connexion
      - VPN_PORT_FORWARDING_UP_COMMAND=/bin/sh -c 'wget -O- --retry-connrefused --post-data "json={\"listen_port\":{{PORTS}},\"current_network_interface\":\"tun0\",\"random_port\":false,\"upnp\":false}" http://127.0.0.1:8080/api/v2/app/setPreferences 2>&1'
      # Tunnel perdu : qBittorrent est rebasculé sur la boucle locale
      - VPN_PORT_FORWARDING_DOWN_COMMAND=/bin/sh -c 'wget -O- --retry-connrefused --post-data "json={\"listen_port\":0,\"current_network_interface\":\"lo\"}" http://127.0.0.1:8080/api/v2/app/setPreferences 2>&1'
      - TZ=Europe/Paris
    ports:
      - 8080:8080                                 # WebUI uniquement, aucun port torrent publié
    volumes:
      - ./gluetun:/gluetun
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:5.3.0  # version figée, acceptée par le tracker
    container_name: qbittorrent
    network_mode: "service:gluetun"               # aucune route hors du tunnel
    depends_on:
      gluetun:
        condition: service_healthy                # démarre seulement quand le tunnel est monté
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
      - WEBUI_PORT=8080
    volumes:
      - ./qbittorrent:/config                     # configuration (écriture)
      - /medias:/medias:ro                        # bibliothèque (lecture seule)
    restart: unless-stopped
```

```bash
docker compose config --quiet && echo "compose OK"   # validation de la syntaxe
docker compose up -d
docker logs gluetun 2>&1 | grep -iE "healthy|port forwarded"
```

> 💡 **Pourquoi figer qBittorrent :** les trackers privés n'acceptent qu'une liste de versions de clients. Une mise à jour automatique via `:latest` peut entraîner un refus du tracker.

---

## 7️⃣ Accès à la WebUI (Nginx + AdGuard)

### Vhost Nginx (LXC 104)

```bash
nano /etc/nginx/sites-available/torrent.conf
```

```nginx
server {
    listen 443 ssl;
    server_name torrent.home;

    ssl_certificate     /etc/nginx/ssl/proxmox.crt;    # certificat interne auto-signé
    ssl_certificate_key /etc/nginx/ssl/proxmox.key;

    # LAN uniquement : la WebUI n'est jamais servie à Internet
    allow 192.168.1.0/24;
    deny  all;

    location / {
        proxy_pass         http://10.0.0.12:8080;
        proxy_http_version 1.1;
        proxy_set_header   Host              $proxy_host;   # attendu par la protection Host de qBittorrent
        proxy_set_header   X-Forwarded-Host  $http_host;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_cookie_path  /                 "/; Secure";
    }
}
```

```bash
ln -s /etc/nginx/sites-available/torrent.conf /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```

### Réécriture DNS (AdGuard Home)

**Filtres > Réécritures DNS** :

| Domaine | Réponse |
| --- | --- |
| `torrent.home` | `192.168.1.244` |

> 💡 Un nom en `.home` n'est résolu que par AdGuard : la WebUI n'a aucune existence dans le DNS public.

Le flux Nginx (`192.168.1.244`) → `10.0.0.12:8080` traverse pfSense par l'interface WAN, déjà autorisée pour le LAN. Le kill switch n'intervient pas : il filtre les connexions **initiées** par le conteneur, pas les réponses.

---

## 8️⃣ Réglages qBittorrent

Mot de passe temporaire de la première connexion :

```bash
docker logs qbittorrent 2>&1 | grep -i password
```

Puis **Outils > Options** :

| Onglet | Réglage | Raison |
| --- | --- | --- |
| IU Web | Identifiants personnels, mot de passe long et unique | Le mot de passe temporaire change à chaque redémarrage |
| IU Web | ✅ Ignorer l'authentification pour les clients sur localhost | Gluetun pousse le port via `127.0.0.1` (sinon `403 Forbidden`) |
| IU Web | ❌ Pas d'exemption pour les sous-réseaux | Toute connexion via Nginx reste authentifiée |
| IU Web | ✅ Proxy inverse, proxy de confiance `192.168.1.244` | Journalisation de la vraie IP cliente |
| Avancé | Interface réseau : `tun0` | Liaison applicative au tunnel (3e barrière) |
| Connexion | ❌ UPnP / NAT-PMP | Le port est géré par Gluetun |
| Téléchargements | Répertoire par défaut : `/medias`, ❌ préallocation | Seed direct depuis la bibliothèque |
| BitTorrent | Chiffrement : **Autoriser** | Le trafic est déjà chiffré par WireGuard, « Exiger » réduirait le nombre de pairs |
| Vitesse | Limite d'envoi globale (~330 Mbit/s au départ) | Le disque USB est partagé avec Plex : priorité à la fluidité du streaming |

Redémarrer la pile, puis vérifier que le port affiché dans **Options > Connexion** est celui annoncé par Gluetun, sans erreur `403` :

```bash
cd /opt/torrent && docker compose down && docker compose up -d
docker logs gluetun 2>&1 | grep -iE "port forwarded|403|exit status" | tail -5
```

---

## 9️⃣ Tests de Validation

| # | Test | Méthode | Résultat attendu | Résultat |
| --- | --- | --- | --- | --- |
| 1 | IP de sortie | `docker exec qbittorrent curl -s https://icanhazip.com` | IP Proton, jamais l'IP fixe | ✅ |
| 2 | Port redirigé | Logs Gluetun vs Options > Connexion | Même numéro des deux côtés | ✅ |
| 3 | Kill switch | Désactiver la règle pfSense « qbit WireGuard Proton » + kill des states `10.0.0.12` | `curl` en échec (code 28 : timeout) pendant la coupure | ✅ |
| 4 | Reprise | Réactiver la règle | IP Proton de retour, nouveau port poussé automatiquement | ✅ |
| 5 | IP vue par les pairs | Magnet de test [ipleak.net](https://ipleak.net) (section *Torrent Address detection*), enregistré dans `/config/test` | IP Proton + port redirigé | ✅ |

> ⚠️ **Piège du test 3 :** un `docker stop gluetun` « réussit » toujours, car il supprime la pile réseau de qBittorrent. Le seul test probant coupe le tunnel **pendant que Gluetun tourne**, comme lors d'une vraie panne.

![ipleak](/images/qbit-ipleak.png)

---

## 🔟 Seed depuis la Bibliothèque

La bibliothèque étant en lecture seule, qBittorrent ne peut que **vérifier** puis **partager** des fichiers déjà présents.

### Procédure

1. Ajouter le fichier `.torrent` (pas le magnet, pour conserver la passkey du tracker), cocher **Ne pas démarrer automatiquement**.

2. Choisir l'agencement et l'emplacement :

    | Cas | Agencement du contenu | Emplacement |
    | --- | --- | --- |
    | Film (fichier unique) | Original | Dossier qui contient le fichier (`/medias/Films`) |
    | Série rangée dans un dossier renommé (épisodes au nom d'origine) | **Ne pas créer de sous-dossier** | Dossier qui contient les épisodes |

3. Clic droit > **Forcer la vérification**, puis **Démarrer** une fois à 100 %. Le statut passe à « En partage ».

### Limites

- Un torrent ne peut être partagé que si **tous** ses fichiers sont présents et identiques. Un `.nfo` ou un *sample* supprimé bloque la vérification sous 100 % (les pièces à cheval sur le fichier manquant échouent).
- Un fichier réencodé ou dont les pistes ont été modifiées ne correspond plus au torrent.
- Fichiers renommés : soit les renommer dans la fenêtre d'ajout, soit recréer l'arborescence d'origine par liens symboliques dans un volume en écriture.

---

## ⚠️ Problèmes Rencontrés

| Symptôme | Cause | Résolution |
| --- | --- | --- |
| Alias `RFC1918` refusé : *Subnet is too large to expand into individual host IP addresses* | Alias créé en type **Host(s)** : pfSense tente de développer chaque réseau en adresses individuelles | Type **Network(s)**, une ligne par réseau avec son masque |
| Règles qbit créées avec la source `*` (relevé avant Apply) | Champ Source laissé par défaut : le block aurait coupé toute la DMZ, Plex compris | Source `QBIT_SRV` sur les 4 règles, relecture systématique avant Apply |
| `apt update` figé à 0 %, DNS bloqué par « qbit kill switch » malgré une règle DNS correcte | Glisser-déposer enregistré sans **Save** : pf chargeait le block en premier | Save puis Apply Changes, contrôle par `pfctl -sr` |
| `pct snapshot` : *snapshot feature is not available* | Proxmox refuse les snapshots d'un CT avec bind mount | Point de retour par `vzdump --mode stop` (le bind mount est exclu automatiquement) |
| Gluetun : `403 Forbidden` sur `setPreferences`, *exit status 8* | L'API Web de qBittorrent exige une authentification | Option « Ignorer l'authentification pour localhost » |
| `grep -r ssl_certificate /etc/nginx/sites-enabled/` ne trouve rien | `sites-enabled` ne contient que des liens symboliques, que `-r` ne suit pas | `grep -R` |
| `nginx -t` : *invalid number of arguments in "ssl_certificate"* | Gabarit `<chemin ...>` non remplacé | Chemins réels du certificat. Ne jamais laisser une config en échec : le prochain redémarrage de Nginx couperait tous les services |
| `ipinfo.io` répond `403 Forbidden` | Le service filtre certaines IP de VPN | Service alternatif : `icanhazip.com` |
| Homarr : pastille rouge malgré l'IP correcte | URL de ping `http://10.0.0.12` → port 80, sur lequel rien n'écoute | `http://10.0.0.12:8080` |

---

## 🔄 Maintenance

### Sauvegarde (avant toute mise à jour)

```bash
# Sur pve-server : arrêt, copie, redémarrage automatique (bibliothèque exclue)
vzdump 1012 --storage HDD-data --mode stop --compress zstd --notes-template 'qbit-ok'

# Restauration
pct restore 1012 /mnt/pve/HDD-data/dump/vzdump-lxc-1012-<date>.tar.zst --force
```

Après une restauration, vérifier la présence de la ligne `mp0` et des deux lignes TUN dans `/etc/pve/lxc/1012.conf`.

### Mise à jour des conteneurs

```bash
cd /opt/torrent
# Modifier le tag qBittorrent uniquement vers une version acceptée par le tracker
docker compose pull
docker compose up -d
docker image prune -f
```

### Surveillance

```bash
docker logs gluetun 2>&1 | grep -i "port forwarded" | tail -1      # port actuel
docker exec qbittorrent curl -s https://icanhazip.com              # IP de sortie
```

Côté pfSense, les paquets refusés sont visibles dans **Status > System Logs > Firewall**, filtrés sur la source `10.0.0.12`.

---

## 📋 Résumé

- **Service :** qBittorrent 5.3.0 derrière Gluetun (Proton VPN WireGuard) sur LXC Debian 13 non privilégié
- **Réseau :** DMZ `10.0.0.12`, seule sortie autorisée : UDP 51820 vers Internet (+ DNS et mises à jour)
- **Kill switch :** double barrière (pare-feu Gluetun + règles pfSense), validée par coupure réelle du tunnel
- **Port forwarding :** NAT-PMP Proton, port poussé automatiquement dans qBittorrent à chaque reconnexion
- **Stockage :** bind mount en lecture seule de la bibliothèque Plex, imposé par l'hyperviseur
- **Accès :** `https://torrent.home` via Nginx, LAN uniquement, WebUI authentifiée
- **Sécurité :** moindre privilège, secret hors du compose, versions figées, sauvegarde avant mise à jour

---

## 📚 Références

- Gluetun : <https://github.com/qdm12/gluetun>
- Wiki Gluetun (Proton VPN, port forwarding) : <https://github.com/qdm12/gluetun-wiki>
- Image linuxserver/qbittorrent : <https://docs.linuxserver.io/images/docker-qbittorrent/>
- Proxmox LXC Mount Points : <https://pve.proxmox.com/wiki/Linux_Container#pct_mount_points>
- Test de fuite torrent : <https://ipleak.net>
- ANSSI : *Recommandations relatives à l'interconnexion d'un système d'information à Internet* (filtrage des flux sortants, cloisonnement)
