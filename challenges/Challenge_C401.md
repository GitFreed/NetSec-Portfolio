# Challenge C401 09/03/2026

## 🧑‍🏫 Pitch de l’exercice : Déploiement Docker et découverte

Challenge : <https://github.com/O-clock-Aldebaran/E01-SC4-exo-docker-GitFreed/tree/master>

[Cours C401.](/RESUME.md#️-c401)

> 📚 **Ressources** :
>
> - Dockerdocs : <https://docs.docker.com/engine/install/debian/>

---

### 1. Spécifications et Déploiement : VM Docker (Debian 13) sur Proxmox VE

```sh
OS : Debian 13 (trixie)
SCSI VirtIO SCSI single
cocher QEMU Guest Agent
Disque Virtuel : 32 Go
cocher la case Discard
cochern SSD emulation
Processeur (vCPU) : 2 cœurs.
CPU Type : host (au lieu du défaut kvm64)
Mémoire (RAM) : 2048 Mo (2 Go)
Network : 
Model : vmbr2 VirtIO (paravirtualisé)

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

💡 *Astuce : on peut stopper et supprimer un conteneur en une seule commande : docker rm -f <name> ou docker rm -f <ID>*

### 4. Compiler nos propres images Docker
