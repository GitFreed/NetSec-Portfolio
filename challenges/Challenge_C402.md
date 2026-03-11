# Challenge C402 10/03/2026

## 🧑‍🏫 Pitch de l’exercice : 🐋 Déployer GLPI avec Docker Compose

Challenge : <https://github.com/O-clock-Aldebaran/SC04E02-Deployer-GLPI-GitFreed/blob/master/README.md>

[Cours C402.](/RESUME.md#️-c402-construction-dimages-et-orchestration-avec-docker-compose)

## 🗂 Contexte

Vous êtes administrateur système dans une PME. Votre responsable vous demande de déployer **GLPI**, l'outil de gestion de parc informatique open-source, de façon reproductible et conteneurisée.

Plutôt que d'installer GLPI directement sur un serveur, vous allez utiliser **Docker Compose** pour orchestrer plusieurs services : l'application GLPI elle-même, sa base de données MariaDB, et en bonus un outil de gestion de BDD via une interface web.

> 💡 **Pourquoi Docker Compose ?**
>
> - Tout l'environnement est décrit dans un seul fichier : `docker-compose.yml`
> - Une seule commande pour tout démarrer, tout arrêter, tout reconstruire
> - L'environnement est identique sur tous les postes de l'équipe

---

## 🎯 Objectifs

À la fin de cet exercice, vous aurez :

- Rédigé un fichier `docker-compose.yml` fonctionnel de zéro
- Configuré un service MariaDB avec variables d'environnement
- Déployé GLPI et réalisé sa configuration initiale via le navigateur
- Mis en place la persistance des données avec des volumes Docker
- *(Bonus)* Ajouté Adminer pour administrer la base de données

---

## 📋 Contraintes & Règles du jeu

> ⚠️ **Important — À respecter impérativement**
>
> ✗ Ne pas copier-coller un `docker-compose.yml` tout fait depuis Internet  
> ✗ Ne pas utiliser d'image GLPI non-officielle ou préconfigurée  
> ✓ Partir des images officielles : `mariadb` et `diouxx/glpi` ou `glpi/glpi`  
> ✓ Construire votre fichier étape par étape en consultant la documentation  
> ✓ Tester chaque ajout avant de passer à l'étape suivante

---

## 🔍 Indices & Documentation

Consultez ces ressources si vous êtes bloqués — mais essayez d'abord par vous-même !

| Ressource | URL / Commande |
| --- | --- |
| Doc Docker Compose | `docs.docker.com/compose/` |
| Image MariaDB (Docker Hub) | `hub.docker.com/_/mariadb` |
| Image GLPI | `hub.docker.com/r/diouxx/glpi` ou `hub.docker.com/r/glpi/glpi` |
| Image Adminer | `hub.docker.com/_/adminer` |
| Variables MariaDB | Voir section *Environment* dans la doc de l'image |
| Logs d'un service | `docker compose logs -f glpi` |
| Entrer dans un conteneur | `docker compose exec db bash` |
| Lister les conteneurs | `docker compose ps` |

> 📚 **Ressources** :
>
> - GLPI Docker Images : <https://hub.docker.com/r/glpi/glpi>
> - Docker Cheatsheet : <https://cheatography.com/christian-knell/cheat-sheets/docker-docker-compose-and-docker-swarm/>

---

[⏬ Aller à : 🐋 Déployer GLPI avec Docker Compose](#-déployer-glpi-avec-docker-compose)

---

## Tests & démo Docker Build & Docker Compose

### Préparation

```sh
git clone https://github.com/pmaldi/docker-avancee-app.git
cd docker-avancee-app
```

### Version 1

```sh
nano Dockerfile.v1
```

```sh
FROM ubuntu:24.04

# Prérequis d'installation et 
RUN apt update
RUN  apt upgrade -y


# Installation de NodeJS
RUN  apt install nodejs -y
RUN  apt install npm -y 

# Je copie les fichiers de mon application dans mon conteneur
COPY . /app

# J'installe Vite
WORKDIR /app
RUN npm install

# Expose mon port 5173 (Attention il c'est le port COTE CONTENEUR et pas coté hote)
EXPOSE 5173

# Je lance mon application
CMD npm run prod
```

Build V1 :

```sh
sudo docker build -t dockerdemo:v1 -f Dockerfile.v1 .
```

![v1](/images/2026-03-10-12-58-10.png)

### Version 2

```sh
FROM ubuntu:24.04

# Prérequis d'installation et Installation de NodeJS
RUN apt update && \
apt upgrade -y && \
apt install nodejs -y --no-install-recommends --no-install-suggests && \
apt install npm -y --no-install-recommends --no-install-suggests

# Je copie les fichiers de mon application dans mon conteneur
COPY . /app

# J'installe Vite
WORKDIR /app
RUN npm install

# Expose mon port 5173 (Attention il c'est le port COTE CONTENEUR et pas coté hote)
EXPOSE 5173

# Je lance mon application
CMD npm run prod
```

Build V2 : `sudo docker build -t dockerdemo:v2 -f Dockerfile.v2 .`

![v2](/images/2026-03-10-12-40-42.png)

### Comparaison des versions

La commande suivante permet de lister les images et d'observer la différence de taille en mégaoctets :

```sh
sudo docker images
```

C'est ici qu'intervient la commande qui exploite le formatage Go-template pour extraire la métrique exacte :

```sh
sudo docker image inspect dockerdemo:v1 --format "V1 Layers: {{len .RootFS.Layers}}"
sudo docker image inspect dockerdemo:v2 --format "V2 Layers: {{len .RootFS.Layers}}"
```

![images](/images/2026-03-10-12-59-21.png)

### Version 3

```sh
# ---- Stage 1 : Build ----
FROM node:alpine AS builder

WORKDIR /app

# On copie uniquement les fichiers de dépendances d'abord (cache Docker optimisé)
COPY package*.json ./

# Installation de TOUTES les dépendances (dev incluses, nécessaires pour le build)
RUN npm ci

# On copie le reste du code
COPY . .

# Build de l'application
RUN npm run build

# ---- Stage 2 : Production ----
FROM node:alpine AS production

WORKDIR /app

COPY package*.json ./

# On installe UNIQUEMENT les dépendances de production
RUN npm ci --omit=dev

# On récupère uniquement le build depuis le stage précédent
COPY --from=builder /app/dist ./dist

EXPOSE 5173

CMD ["node", "dist/index.js"]
```

Build V2 : `sudo docker build -t dockerdemo:v3 -f Dockerfile.v3 .`

![v3](/images/2026-03-10-14-15-40.png)

### Docker Compose

```sh
nano docker-compose.yaml
```

```sh
services:
  web:
    image: php:8.2-apache
    ports:
      - "8080:80"
    volumes:
      - ./src:/var/www/html
    depends_on:
      - db

  db:
    image: mariadb:11
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: myapp
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

```sh
sudo docker compose up -d
```

![compose](/images/2026-03-10-14-38-36.png)

---
---
---

## 🐋 Déployer GLPI avec Docker Compose

### Étape 1 — Mise en place du projet

- Créez un dossier dédié pour votre projet : `mkdir glpi-docker && cd glpi-docker`
- Créez le fichier `docker-compose.yml` vide et préparez la structure de votre projet
- Réfléchissez aux services dont vous aurez besoin avant de commencer à écrire

**Création du fichier `.env**` (pour stocker les variables de manière sécurisée) :

```bash
nano .env

```

*Contenu à insérer :*

```env
MYSQL_ROOT_PASSWORD=Rootpassword!
MYSQL_DATABASE=glpidb
MYSQL_USER=glpiuser
MYSQL_PASSWORD=Glpiuserpassword!

```

**Création d'un fichier `.gitignore**` (pour empêcher l'exportation accidentelle des mots de passe sur un dépôt de code) :

```bash
echo ".env" > .gitignore

```

**Création du fichier principal vide :**

```bash
touch docker-compose.yml

```

### Étape 2 — Service MariaDB

- Ajoutez un service `mariadb` dans votre `docker-compose.yml`
- Définissez les variables d'environnement nécessaires : `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`
- Montez un volume pour persister les données de la base
- Testez que le conteneur démarre correctement avec : `docker compose up -d db`

Avec `nano docker-compose.yml` on édit le fichier Yaml

```sh
services:
  # ÉTAPE 2 : Service MariaDB
  db:
    image: mariadb:10.11
    container_name: glpi-db
    environment:
      # Appel sécurisé des variables depuis le fichier .env
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - glpi-net

volumes:
  db_data:

networks:
  glpi-net:
```

![db](/images/2026-03-10-17-08-16.png)

### Étape 3 — Service GLPI

- Ajoutez le service `glpi` en utilisant l'image `glpi/glpi ou diouxx/glpi`
- Exposez le port 80 du conteneur sur un port de votre machine
- Configurez la dépendance vers le service `db` avec `depends_on`
- Montez les volumes nécessaires pour les fichiers GLPI (config, fichiers uploadés...)

```sh
  # ÉTAPE 3 : Service GLPI
  glpi:
    image: glpi/glpi:latest
    container_name: glpi-app
    ports:
      - "8080:80"
    environment:
      TIMEZONE: 'Europe/Paris'
      # Ajout des variables exigées par l'Entrypoint de l'image GLPI
      MARIADB_HOST: db
      MARIADB_DATABASE: ${MYSQL_DATABASE}
      MARIADB_USER: ${MYSQL_USER}
      MARIADB_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - glpi_data:/var/www/html
    depends_on:
      db:
        condition: service_healthy
    networks:
      - glpi-net

volumes:
  db_data:
  glpi_data:

networks:
  glpi-net:

```

### Étape 4 — Réseau & Communication

- Créez un réseau Docker dédié pour que vos services puissent communiquer
- Rattachez chaque service à ce réseau
- Vérifiez que GLPI peut joindre MariaDB : le nom d'hôte à utiliser est le **nom du service** `db`

```sh
# ÉTAPE 4 : Déclaration formelle des volumes et du réseau isolé
volumes:
  db_data:
  glpi_data:

networks:
  glpi-net:
    driver: bridge
```

### Étape 5 — Démarrage & Configuration initiale

- Lancez l'ensemble des services : `docker compose up -d`
- Ouvrez votre navigateur sur `http://10.0.0.30:8080`
- Suivez l'assistant d'installation GLPI en renseignant les informations de connexion à la BDD
- Connectez-vous avec les identifiants par défaut (`glpi` / `glpi`) et changez-les !

**Lancement des conteneurs en arrière-plan :**

```bash
sudo docker compose up -d

```

**Vérification de l'état des services** (il faut s'assurer que `db` est en statut *healthy* et les autres en *running*) :

```bash
sudo docker compose ps

```

![ok](/images/2026-03-10-17-32-24.png)

**Configuration via le navigateur web :**

- Accéder à l'interface GLPI : `http://10.0.0.30:8080` (ou l'IP de la machine virtuelle si le navigateur est sur l'hôte physique).

- Lors de l'assistant d'installation, renseigner la base de données :
        - **Serveur SQL :** `db` *(Le DNS interne de Docker se charge de résoudre ce nom en adresse IP).*
        - **Utilisateur / Base / Mot de passe :** Ceux inscrits dans le fichier `.env`.

- *Sécurité stricte :* Une fois connecté avec `glpi`/`glpi`, il est impératif de modifier immédiatement les mots de passe des comptes par défaut (`glpi`, `tech`, `normal`, `post-only`) et de supprimer le fichier d'installation (ou laisser GLPI avertir de ce risque de sécurité).

![glpi](/images/2026-03-10-17-48-32.png)

![slpiok](/images/2026-03-10-18-18-45.png)

### 🏆 Bonus 1 — Adminer

Ajoutez le service **Adminer** à votre stack. Adminer est une interface web légère pour administrer des bases de données. Exposez-le sur le port `8081` et connectez-vous avec les identifiants de votre base GLPI.

Il faut compose down `sudo docker compose down -v` puis modifier le fichier yaml

![down](/images/2026-03-10-18-22-56.png)

```sh
# BONUS 1 : Adminer
  adminer:
    image: adminer:latest
    container_name: glpi-adminer
    ports:
      - "8081:8080"
    depends_on:
      - db
    networks:
      - glpi-net
```

![admirer](/images/2026-03-10-18-23-48.png)

**Test de l'interface d'administration de la BDD (Bonus) :**

- Accéder à Adminer : `http://10.0.0.30:8081`.
- Serveur : `db`, Utilisateur : `root` ou `glpiuser`.

![admirer](/images/2026-03-10-18-53-08.png)

### 🏆 Bonus 2 — Fichier `.env`

Déplacez tous les mots de passe et variables sensibles dans un fichier `.env` et utilisez la syntaxe `${VARIABLE}` dans votre `docker-compose.yml`. Ajoutez `.env` à un fichier `.gitignore` pour ne jamais le commiter.

[Voir Etape 1](#étape-1--mise-en-place-du-projet)

### 🏆 Bonus 3 — Healthcheck

Ajoutez un `healthcheck` sur le service `db` pour que GLPI n'essaie de démarrer qu'une fois que MariaDB est réellement prêt à accepter des connexions.

> Indice : `condition: service_healthy` dans `depends_on`

```sh
# Bonus 3 : S'assurer que le service SQL est prêt avant de lancer l'application
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--su-mysql", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 5
```
