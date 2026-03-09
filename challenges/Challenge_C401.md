# Challenge C401 09/03/2026

## 🧑‍🏫 Pitch de l’exercice : Déploiement Docker et découverte

Challenge : <https://github.com/O-clock-Aldebaran/E01-SC4-exo-docker-GitFreed/tree/master>

[Cours C401.](/RESUME.md#-c401-introduction-à-la-conteneurisation-et-à-docker)

> 📚 **Ressources** :
>
> - Dockerdocs : <https://docs.docker.com/engine/install/debian/>

---

### 1. Spécifications et Déploiement : VM Docker (Debian 13) sur Proxmox VE

```sh
OS : Debian 13 (trixie)
SCSI VirtIO SCSI single
QEMU Guest Agent ✅
Disque Virtuel : 32 Go
Discard ✅
SSD emulation ✅
Processeur (vCPU) : 2 cœurs.
CPU Type : host (au lieu du défaut kvm64)
Mémoire (RAM) : 2048 Mo (2 Go)
Network : 10.0.0.20/24
Model : vmbr2 en VirtIO Para

```

### 2. Phase de Post-Installation et Durcissement Initial

```sh
# Basculer sur le compte super-administrateur (entrer le mot de passe root)
su -

# Mettre à jour la liste des paquets et installer sudo
apt update && apt install sudo -y

# Ajouter l'utilisateur standard au groupe sudo
usermod -aG sudo freed

# Quitter la session root pour revenir à l'utilisateur standard
exit

# Recharger la session de l'utilisateur standard
su - freed

# Tester l'élévation de privilèges (le mot de passe de l'utilisateur standard sera demandé)
sudo -l

# Ouverture du fichier avec l'éditeur de texte nano avec les droits d'administration
sudo nano /etc/network/interfaces

# Configuration statique pour le réseau DMZ (Remplacer 'ens18' si nécessaire)
allow-hotplug ens18
iface ens18 inet static
    address 10.0.0.5/24
    gateway 10.0.0.1
    # Facultatif si resolv.conf est déjà configuré, mais recommandé :
    dns-nameservers 10.0.0.1 9.9.9.9

# Redémarrage du service réseau pour appliquer les modifications du noyau
sudo systemctl restart networking

# Vérification de la nouvelle adresse
ip a

# Mise à jour des dépôts et installation du serveur OpenSSH
sudo apt update && sudo apt install openssh-server -y

# S'assurer que le service démarre automatiquement avec la machine virtuelle
sudo systemctl enable --now ssh

# Vérification de l'état du démon (doit afficher "active (running)")
sudo systemctl status ssh

# Commande à taper depuis le poste de travail
ssh freed@10.0.0.20

# Installation de l'agent depuis les dépôts officiels
sudo apt install qemu-guest-agent -y

# Activation et démarrage immédiat du service
sudo systemctl enable --now qemu-guest-agent

# Vérification du statut du service (il doit être "active (running)")
sudo systemctl status qemu-guest-agent
```

### 3. Déploiement de Docker Engine 🐳

```sh
# Suppression des anciennes versions ou des forks (podman, docker.io) potentiellement présents
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-doc podman-docker containerd runc | cut -f1)
```

```sh
# Mise à jour des index locaux et installation des outils de requête web et de certificats
sudo apt-get update
sudo apt-get install ca-certificates curl -y

# Création du répertoire sécurisé pour les clés
sudo install -m 0755 -d /etc/apt/keyrings

# Téléchargement de la clé GPG officielle pour Debian
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc

# Application des droits de lecture stricts sur la clé
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

```sh
# Ajout du dépôt aux sources d'APT
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

```sh
# Mise à jour des index pour inclure le nouveau dépôt Docker
sudo apt update

# Installation des composants
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

```sh
# Vérification du statut du service en arrière-plan
sudo systemctl status docker

# Téléchargement et exécution d'un conteneur de test éphémère
sudo docker run hello-world
```

![hello](/images/2026-03-09-20-10-52.png)

### 4. Exercice : bases de la CLI Docker

```sh
docker run -p 8888:80 bdelphin/hello-docker
```

Sur <http://10.0.0.20:8888/>

![hello](/images/2026-03-09-20-16-30.png)

Pour lancer en tâche de fond :  `docker run -d -p 8888:80 bdelphin/hello-docker`

`sudo docker ps` pour voir les conteneurs

![ps](/images/2026-03-09-20-20-34.png)

`docker stop <ID>` permet de stopper le container

![stop](/images/2026-03-09-20-22-19.png)

On peut nommer les containers pour s'y retrouver avec l'argument `--name`

On peut facilement le stop maintenant avec le nom, et pour le relancer il suffit de faire `docker start`, ou le supprimer avec `docker rm` pour le relancer avec `docker run`

![name](/images/2026-03-09-20-25-11.png)

💡 *Astuce : on peut stopper et supprimer un conteneur en une seule commande : docker rm -f "name" ou docker rm -f "ID"*

### 4. Compiler nos propres images Docker

Pour compiler une image Docker, nous allons avoir besoin de créer un fichier `Dockerfile`

```sh
# Préparation de l'arborescence
mkdir -p ~/challenge-c401/src
cd ~/challenge-c401

# Création du fichier source (PHP)
nano src/index.php
# avec le code suivant :
<?php
echo "<h1>Hello-world !</h1>";
echo "<p>Ceci est un conteneur Docker tournant sur Debian Trixie.</p>";
phpinfo();
?>

# Création de la docker file
nano Dockerfile

# Instructions du fichier
FROM php:7.2-apache
WORKDIR /var/www/html
COPY src .
```

La première instruction `FROM php:7.2-apache` indique à Docker que nous allons baser notre image sur une image existante : `php:7.2-apache`. Le nom de l'image c'est `php`, et après le `:` on vient préciser le **tag** de cette image que l'on souhaite utiliser. Les tags sont des "versions" des images, et dans notre cas la version `7.2-apache` est une image qui embarque PHP en version 7.2 avec le serveur web apache préconfiguré.

La deuxième instruction `WORKDIR /var/www/html` permet de changer le dossier dans lequel on se trouve **à l'intérieur de l'image** (c'est comme si on lançait la commande `cd /var/www/html`). On indique ici qu'il faut se positionner dans le dossier `/var/www/html`, dossier "racine" du serveur web apache.

La dernière instruction `COPY src .` effectue, comme son nom l'indique ... une copie ! C'est comme si on avait lancé la commande `cp src/* ./`, on demande à docker de copier le contenu du dossier `src/` (sur notre hôte !) dans un dossier à l'intérieur de l'image. Le dossier de destination `./` correspond au **dossier courant**, c'est à dire `/var/www/html`, vu qu'on vient de s'y déplacer avec l'instruction précédente.

![docker](/images/2026-03-09-20-54-48.png)

```sh
# On instancie le conteneur en mappant le port 8888 de la VM vers le port 80 du conteneur
sudo docker run -d -p 8888:80 --name challenge-container my-hello-docker

# On verifie le container
sudo docker ps
```

![ps](/images/2026-03-09-21-16-06.png)

Vérification sur <http://10.0.0.20:8888/>

![apache](/images/2026-03-09-21-08-08.png)

### 5. DockerHub & Killercoda

Sur <https://hub.docker.com/repositories/9cw0aidl0hzvviowpdrvcn0w> on choisi un nom pour notre dépôt, et clic sur Create. Sur la droite DockerHub nous donne la procédure à suivre pour pousser une image sur ce dépôt :

```sh
docker tag local-image:tagname new-repo:tagname
docker push new-repo:tagname
```

**Docker & DockerHub fonctionnenent comme Git & GitHub** : nous avons des dépôts sur un serveur distant (DockerHub/GitHub), sur lesquels nous allons héberger nos images, mais aussi une sorte de **dépôt local** sur notre hôte, pour stocker les images que nous compilons et celles que nous récupérons depuis DockerHub.

On peut voir les images en local sur notre machine en tapant la commande `docker image ls`. On peut supprimer une image locale avec `docker image rm <nom_image>` ou `docker image rm <id_image>`.

On peut retrouver dans la liste des images locales notre image `my-hello-docker`, et on peut remarquer qu'elle a un tag `latest`, c'est le tag par défaut.

Pour pousser cette image sur notre dépôt il faut se connecter depuis le terminal avec `docker login -u <user_dockerhub>`

![login](/images/2026-03-09-21-35-12.png)

On peut maintenant lancler la commande pour push notre image

```sh
docker tag my-hello-docker:latest <user_dockerhub>/my-hello-docker:latest
docker push <user_dockerhub>/my-hello-docker:latest
```

![ok](/images/2026-03-09-22-44-52.png)

Pour tester notre image, on va lancer une session Docker dans Killercoda

On va lancer dans ce terminal la commande `docker run -dp 8888:80 <user_dockerhub>/my-hello-docker`

On peut voir Docker récupérer et lancer l'image et vérifier avec `docker ps`

![killercoda](/images/2026-03-09-22-50-25.png)
