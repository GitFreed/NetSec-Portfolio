# Challenge C402 10/03/2026

## 🧑‍🏫 Pitch de l’exercice : Build Dockerfile

Challenge : <.>

[Cours C401.](/RESUME.md#-c402)

> 📚 **Ressources** :
>
> - Dockerdocs : <https://docs.docker.com/engine/install/debian/>

---

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
