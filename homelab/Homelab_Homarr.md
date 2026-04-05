# 🏠 LAB : Dashboard Centralisé avec Homarr

```txt
 _   _                                
| | | | ___  _ __ ___   __ _ _ __ _ __ 
| |_| |/ _ \| '_ ` _ \ / _` | '__| '__|
|  _  | (_) | | | | | | (_| | |  | |   
|_| |_|\___/|_| |_| |_|\__,_|_|  |_|   
```

![Type](https://img.shields.io/badge/Environnement-Docker-2496ED?style=flat-square&logo=docker&logoColor=white)
![OS](https://img.shields.io/badge/OS-Debian%2013%20Trixie-A81D33?style=flat-square&logo=debian&logoColor=white)
![Service](https://img.shields.io/badge/Service-Homarr%20v1-F5A97F?style=flat-square)
![Role](https://img.shields.io/badge/Rôle-Dashboard%20%26%20Monitoring-005085?style=flat-square)
![Network](https://img.shields.io/badge/Réseau-DMZ-purple?style=flat-square&logo=web&logoColor=white)

**Rôle :** Administrateur d'Infrastructures Sécurisées / DevOps

**Mission :** Déployer Homarr, un dashboard centralisé permettant de piloter l'ensemble de l'infrastructure depuis une interface web unique. Homarr agrège les données des services critiques via leurs API (Proxmox, Plex, AdGuard Home, Docker) et fournit une vue temps réel de l'état de santé du lab : charge CPU/RAM de l'hyperviseur, conteneurs actifs, requêtes DNS bloquées, flux multimédia en cours. L'objectif est de remplacer la navigation entre de multiples interfaces d'administration par un point d'entrée unique et personnalisé.

> - [GitHub Homarr](https://github.com/homarr-org/homarr)
> - [Documentation Officielle](https://homarr.dev/docs/getting-started/introduction)

---

## L'intérêt technique 🎯

1. **Centralisation (Single Pane of Glass) :** Agréger la supervision de services hétérogènes (hyperviseur, pare-feu, DNS, multimédia, conteneurs) dans une seule interface, éliminant la navigation entre 6+ interfaces d'administration différentes.
2. **Intégrations API (REST) :** Configurer des connexions authentifiées vers les API de chaque service (API Tokens Proxmox, Plex Token, clés AdGuard) — compétence directement transposable en environnement professionnel (ITSM, NOC).
3. **DevOps (Docker Compose) :** Déploiement déclaratif avec persistance des données via volumes Docker, mise à jour sans perte de configuration, et gestion du cycle de vie des conteneurs.
4. **Personnalisation :** Thème Catppuccin Macchiato cohérent avec l'ensemble de l'écosystème, DNS rewrites AdGuard pour un accès via URL propre (`homarr.home`).

---

## 🛠️ Architecture du Lab

- **Environnement :** VM Docker sur Proxmox VE
- **Hébergement :** Conteneur Docker sur la VM `10.0.0.20` (Zone DMZ, sur `vmbr2`)
- **Port d'écoute :** `7575` TCP
- **Accès :** `http://10.0.0.20:7575` ou `http://homarr.home` (via DNS rewrite AdGuard)
- **Intégrations configurées :**

| Service | Protocole | Authentification |
| --- | --- | --- |
| Proxmox VE | API REST (HTTPS) | API Token (`homarr@pve!homarr`) |
| Plex | API REST (HTTP) | Plex Token (PlexOnlineToken) |
| AdGuard Home | API REST (HTTP) | Identifiants AdGuard |
| Docker | Socket Unix | `/var/run/docker.sock` (lecture seule) |

---

## 1️⃣ Déploiement (Docker Compose)

Depuis la VM Docker (`10.0.0.20`), créer le répertoire et le fichier d'orchestration :

```bash
mkdir -p ~/homarr && cd ~/homarr
nano docker-compose.yml
```

```yaml
services:
  homarr:
    image: ghcr.io/homarr-org/homarr:latest
    container_name: homarr
    restart: unless-stopped
    ports:
      - "7575:7575"
    volumes:
      - homarr_data:/appdata
      - /var/run/docker.sock:/var/run/docker.sock:ro  # Monitoring Docker (lecture seule)
    environment:
      - TZ=Europe/Paris

volumes:
  homarr_data:
```

> ⚠️ **Sécurité :** le socket Docker est monté en **lecture seule** (`ro`). Homarr peut lister les conteneurs et leurs métriques mais ne peut pas les modifier ni en créer. C'est l'application du moindre privilège pour l'accès au démon Docker.

```bash
docker compose up -d
```

L'interface est accessible sur `http://10.0.0.20:7575` 🎉

---

## 2️⃣ Configuration Initiale

Lors de la première connexion :

1. **Créer un compte administrateur** avec un mot de passe fort
2. **Paramètres > Apparence** → sélectionner le thème sombre (Catppuccin Macchiato)
3. Commencer à ajouter les applications et intégrations sur le tableau de bord

---

## 3️⃣ Intégrations API

### Proxmox VE

**Côté Proxmox** (prérequis) :

1. Créer un utilisateur `homarr@pve` avec le rôle `PVEAuditor` (lecture seule) sur le chemin `/`
2. Créer un API Token : **Datacenter > Permissions > API Tokens > Add** → Token ID `homarr`, **décocher** "Privilege Separation"
3. Copier le Token Secret (affiché une seule fois)

**Côté Homarr** → Intégrations > Proxmox :

| Champ | Valeur |
| --- | --- |
| URL | `https://192.168.1.240:8006` |
| Nom d'utilisateur | `homarr@pve` |
| ID du jeton | `homarr` |
| Clé API | `<TOKEN_SECRET>` |
| Domaine | `pve` |

> ⚠️ Proxmox utilise un certificat auto-signé. Homarr peut demander de télécharger le certificat CA. Le récupérer depuis Proxmox (`/etc/pve/pve-root-ca.pem`) et l'importer dans Homarr. Si l'erreur persiste, ajouter `NODE_TLS_REJECT_UNAUTHORIZED=0` dans les variables d'environnement du conteneur.

**Données remontées :** nœuds (CPU, RAM), VMs (état, ressources), LXCs (état, ressources), stockage (utilisation).

### Plex

**Récupérer le Plex Token** depuis le LXC Plex (`10.0.0.10`) :

```bash
grep -oP 'PlexOnlineToken="\K[^"]+' '/var/lib/plexmediaserver/Library/Application Support/Plex Media Server/Preferences.xml'
```

**Côté Homarr** → Intégrations > Plex :

| Champ | Valeur |
| --- | --- |
| URL | `http://10.0.0.10:32400` |
| Clé API | `<PLEX_TOKEN>` |

> ⚠️ Le token visible dans l'URL du navigateur Plex (`X-Plex-Token=...`) n'est pas toujours le bon. Utiliser la méthode `grep` sur le fichier `Preferences.xml` pour le token définitif.

**Données remontées :** sessions en cours, derniers médias ajoutés, utilisateurs actifs.

### AdGuard Home

**Côté Homarr** → Intégrations > AdGuard Home :

| Champ | Valeur |
| --- | --- |
| URL | `http://192.168.1.250` |
| Nom d'utilisateur | `<ADMIN_ADGUARD>` |
| Mot de passe | `<MOT_DE_PASSE>` |

**Données remontées :** requêtes DNS (total, bloquées), pourcentage de blocage, domaines sur liste noire.

### Docker

L'intégration Docker est automatique grâce au montage du socket dans le `docker-compose.yml`. Homarr détecte et liste tous les conteneurs de la VM.

**Données remontées :** conteneurs (nom, état, CPU, RAM), actions disponibles (démarrer, arrêter, redémarrer).

---

## 4️⃣ DNS Rewrite (Accès par URL propre)

Pour accéder à Homarr via `http://homarr.home` au lieu de `http://10.0.0.20:7575` :

Dans l'interface **AdGuard Home** → **Filtres > Réécriture DNS** → Ajouter :

| Domaine | Réponse |
| --- | --- |
| `homarr.home` | `10.0.0.20` |

> 💡 L'accès par port (`7575`) reste nécessaire tant que le reverse proxy [Nginx](./Homelab_Infra.md) n'est pas déployé. Une fois Nginx en place, il suffira de router `homarr.home:80` vers `10.0.0.20:7575` en interne.

---

## 🔄 Maintenance

### Mise à jour

```bash
cd ~/homarr

# Télécharger la dernière image
docker compose pull homarr

# Recréer le conteneur (les données sont dans le volume)
docker compose up -d homarr

# Nettoyer les anciennes images
docker image prune -f
```

### Sauvegarde

Les données Homarr (configuration, tableau de bord, intégrations) sont stockées dans le volume Docker `homarr_data`. Pour sauvegarder :

```bash
# Identifier le chemin du volume
docker volume inspect homarr_homarr_data --format '{{ .Mountpoint }}'

# Copier le contenu
cp -r $(docker volume inspect homarr_homarr_data --format '{{ .Mountpoint }}') ~/backup/homarr_$(date +%Y%m%d)/
```

---

## 📋 Résumé

- **Service :** Homarr — dashboard centralisé d'infrastructure
- **Hébergement :** Docker sur VM `10.0.0.20:7575` (DMZ)
- **Intégrations :** Proxmox (API Token), Plex (PlexOnlineToken), AdGuard Home (credentials), Docker (socket ro)
- **Accès :** `http://homarr.home` (DNS rewrite AdGuard)
- **Thème :** Catppuccin Macchiato
- **Sécurité :** socket Docker en lecture seule, API tokens avec moindre privilège (PVEAuditor), mots de passe non stockés en clair
- **Maintenance :** `docker compose pull && up -d` pour les mises à jour

---

## 📚 Références

- GitHub Homarr : <https://github.com/homarr-org/homarr>
- Documentation : <https://homarr.dev/docs/getting-started/introduction>
- Catppuccin Theme : <https://catppuccin.com/palette/>
- Proxmox API : <https://pve.proxmox.com/pve-docs/api-viewer/>

---

![dashboard](/images/2026-04-05-23-22-17.png)
