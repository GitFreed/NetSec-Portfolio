# Challenge C305 02/03/2026

## Pitch de l’exercice 🧑‍🏫

## 🛡️ Sécurisation d’un serveur Debian exposé sur Internet

### 🎯 Contexte

Vous venez d’intégrer l’équipe infrastructure d’une mairie de votre région.

Un nouveau serveur sous **Debian** doit être déployé en urgence pour héberger un futur service interne.
Avant sa mise en production, l’équipe sécurité exige un **durcissement minimal du système** et une **restriction stricte des accès SSH**.

Le responsable sécurité vous transmet les exigences suivantes :

Votre mission consiste à préparer le serveur conformément aux exigences de sécurité de base.

---

### 🖥️ Environnement technique

* 1 machine virtuelle vierge sous Debian (installation minimale)
* Accès console root

---

[Challenge C305](https://github.com/O-clock-Aldebaran/SC3E06-ssh-hardening/)

[Cours C305.](/RESUME.md#-c305-sécurité-linux-pare-feu--ssh)

[Récap des commandes Pare-feu & SSH](./Challenge_C305_recap.md)

---

### Installation et configuration de SSH

```sh
# Installation du serveur SSH
apt update
apt install -y openssh-server

# Vérification du statut et du port d'écoute
systemctl status ssh
ss -tlnp | grep ssh

# Activation au démarrage
systemctl enable ssh
```

![status](/images/2026-03-02-23-05-56.png)

### Mise en place d’une règle de filtrage avec iptables

```sh
# Pré-requis : installation de la persistance des règles
apt install -y iptables-persistent

# Nettoyage des règles existantes
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X

# Politique par défaut : tout bloquer en entrée
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Autoriser le loopback
iptables -A INPUT -i lo -j ACCEPT

# Autoriser les connexions déjà établies
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Autoriser SSH uniquement depuis notre IP sur le port 22
iptables -A INPUT -p tcp -s 192.168.1.5 --dport 22 -j ACCEPT

# Vérification des règles
iptables -L -n -v

# Sauvegarde pour persistance
netfilter-persistent save
```

![iptable](/images/2026-03-02-23-09-01.png)

Vérification en se connectant en SSH depuis l'IP enregistrée ✅

Et depuis une autre IP ❌

### SSH Hardening (durcissement)

```sh
# Générer une paire de clés sur votre poste (si besoin)
ssh-keygen -t ed25519 -C "admin@mairie"

# Copier la clé publique sur le serveur
ssh-copy-id root@192.168.1.151

# Sur windows on utilise une autre commande pour copier la clé publique sur le serveur
 type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh root@192.168.1.151 "cat >> .ssh/authorized_keys"

# Éditer la configuration SSH côté serveur
nano /etc/ssh/sshd_config
```

Dans `sshd_config`

```sh
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers admin
```

Création d'un nouvel utilisateur avec accès SSH

```sh
# Sur le serveur, créer un nouvel utilisateur
adduser admin
usermod -aG admin
# Copier la clé publique pour le nouvel utilisateur
mkdir -p /home/admin/.ssh
touch /home/admin/.ssh/authorized_keys
cp ~/.ssh/authorized_keys /home/admin/.ssh/authorized_keys
chown -R admin:admin /home/admin/.ssh
chmod 700 /home/admin/.ssh
chmod 600 /home/admin/.ssh/authorized_keys
```

Vérifier et relancer SSH

```sh
# Vérifier la syntaxe
sshd -t

# Recharger la configuration
systemctl restart ssh

# Tester une connexion en conservant une session ouverte
ssh admin@192.168.1.151
```

![OK](/images/2026-03-03-00-09-02.png)

### Mise en place d’un utilisateur admin avec sudo

```sh
# Installation du paquet sudo
apt update && apt install sudo -y

# Lier l'utilisateur au groupe Sudo
usermod -aG sudo admin
```

Test de validation

![sudo](/images/2026-03-03-00-17-17.png)

### Changer le port SSH

Changer le port par défaut (22) permet d'éliminer 99 % du bruit de fond généré par les bots qui scannent internet à la recherche de serveurs ouverts.

```sh
sudo nano /etc/ssh/sshd_config

Port 22 # Décommenter et passer en 22222

#On désactive et on stoppe l'écoute imposée par systemd du port 22
sudo systemctl disable --now ssh.socket

sudo systemctl restart ssh

iptables -A INPUT -p tcp -s 192.168.1.5 --dport 22222 -j ACCEPT
```

Pour se reconnecter on utilisera `ssh admin@192.168.1.151 -p 22222`

![port22222](/images/2026-03-03-00-33-03.png)

### Mettre en place une politique iptables plus complète (ESTABLISHED, RELATED)

Un pare-feu "Stateful" garde en mémoire l'état des connexions (le fameux TCP 3-way handshake)

* **ESTABLISHED** : Le paquet fait partie d'une connexion bidirectionnelle déjà validée.

* **RELATED** : Le paquet initie une nouvelle connexion, mais elle est directement liée à une connexion existante et légitime (comme le flux de données FTP lié à un flux de contrôle).

L'ordre est important pour ne pas se bloquer

```sh
# 1. Autoriser tout le trafic sur la boucle locale (indispensable au système)
sudo iptables -A INPUT -i lo -j ACCEPT

# 2. La règle Stateful : accepter les paquets des connexions légitimes en cours
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 3. Ouvrir explicitement la porte pour ton NOUVEAU port SSH (remplace 22222)
sudo iptables -A INPUT -p tcp --dport 22222 -j ACCEPT

# 4. Verrouiller tout le reste (La politique de Drop par défaut)
sudo iptables -P INPUT DROP

netfilter-persistent save
```

Avec cette configuration, le serveur devient invisible de l'extérieur, sauf sur le port 22222. Si un paquet arrive, il est d'abord scanné par la table de suivi des connexions (conntrack). S'il n'appartient à aucune session connue et qu'il ne vise pas le port SSH, il est détruit silencieusement.
