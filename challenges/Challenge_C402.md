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

# Prérequis d'installation
RUN apt update
RUN apt upgrade -y

# Installation de NodeJS
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.4/install.sh | bash
RUN \. "$HOME/.nvm/nvm.sh"
RUN nvm install 24

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

### Comparaison des versions

La commande suivante permet de lister les images et d'observer la différence de taille en mégaoctets :

```sh
sudo docker images
```

C'est ici qu'intervient la commande qui exploite le formatage Go-template pour extraire la métrique exacte :

```sh
sudo docker image inspect dockerdemo:v1 --format "V1 Layers: {{len .RootFS.Layers}}"
sudo docker image inspect dockerdemo:v2 --format "V2 Layers: {{len .RootFS.Layers}}".
```
