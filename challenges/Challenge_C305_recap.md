# Récapitulatif des configurations firewall et durcissement SSH 🧱 C305 02/03/2026

Ce document présente des configurations de firewall simples et sécurisées pour un serveur exposé uniquement en SSH, avec trois outils différents : iptables, nftables et UFW.

Dans la seconde partie nous effectuerons un durcissement de connection avec clef SSH.

[Challenge C305](./Challenge_C305.md)

[Cours C305.](/RESUME.md#-c305)

---

## Configurations firewall

### Notes et bonnes pratiques

- Tester toujours la connexion SSH avant d'appliquer des règles DROP, sinon risque blocage.
- Les connexions établies sont gérées automatiquement par nftables et UFW.
- Pour plus de sécurité, on peut ajouter un blocage des paquets invalides ou limiter les tentatives SSH (anti-bruteforce).

---

### SSH sur machine locale

```sh
# Sur le serveur
sudo sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh
ip a

# en Local (se connecter en utilisant le mot de passe root)
ssh -o PubkeyAuthentication=no root@10.0.0.99
```

---

## iptables

### Installation de la persistance

```bash
# Installer le paquet pour sauvegarder les règles iptables au redémarrage
apt install -y iptables-persistent
```

### Nettoyage des règles existantes

```bash
# Vider toutes les chaînes de la table filter (INPUT, OUTPUT, FORWARD)
iptables -F

# Supprimer les chaînes personnalisées
iptables -X

# Vider les chaînes de la table nat
iptables -t nat -F
iptables -t nat -X
```

### Politique par défaut

```bash
# Bloquer tout le trafic entrant par défaut
iptables -P INPUT DROP

# Bloquer le transit de paquets entre interfaces
iptables -P FORWARD DROP

# Autoriser tout le trafic sortant
iptables -P OUTPUT ACCEPT

# Vérifier les politiques par défaut
iptables -L -n --line-numbers -v
```

### Autoriser le loopback

```bash
# Autoriser les communications internes (127.0.0.1)
iptables -A INPUT -i lo -j ACCEPT
```

### Autoriser les connexions déjà établies

```bash
# Permettre les réponses aux connexions déjà initiées (ex: réponses HTTP, DNS...)
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

### Autoriser SSH uniquement depuis une IP spécifique

```bash
# Remplacer 10.42.0.2 par l'adresse IP autorisée (ex: poste administrateur)
iptables -A INPUT -p tcp -s 10.42.0.2 --dport 22 -m conntrack --ctstate NEW -j ACCEPT
```

### Autoriser les connexions HTTP/HTTPS (optionnel, si besoin d'exposer un service web)

```bash
# Autoriser HTTP (port 80) et HTTPS (port 443) depuis n'importe quelle IP
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
```

### Optionnel : drop des paquets invalides

```bash
# Rejeter les paquets dans un état de connexion invalide (protection supplémentaire)
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
```

### Vérification et sauvegarde

```bash
# Afficher les règles actives avec statistiques
iptables -L -n -v

# Sauvegarder les règles pour qu'elles persistent après redémarrage
netfilter-persistent save
```

---

## nftables

### Installation et activation nftables

```bash
# Installer nftables
apt install -y nftables

# Activer et démarrer le service nftables
systemctl enable nftables
systemctl start nftables
```

### Exemple de configuration (`/etc/nftables.conf`)

```conf
#!/usr/sbin/nft -f

# Supprimer toutes les règles existantes avant d'appliquer la nouvelle configuration
flush ruleset

table inet filter {

    chain input {
        # Politique par défaut : bloquer tout le trafic entrant
        type filter hook input priority 0; policy drop;

        # Autoriser le trafic loopback (communications internes)
        iif lo accept

        # Autoriser les connexions déjà établies ou liées
        ct state established,related accept

        # Autoriser SSH uniquement depuis l'IP 10.42.0.2 (remplacer par l'IP réelle)
        tcp dport 22 ip saddr 10.42.0.2 ct state new accept

        # Autoriser les connexions HTTP/HTTPS (optionnel)
        tcp dport { 80, 443 } accept

        # Rejeter les paquets avec un état de connexion invalide
        ct state invalid drop
    }

    chain forward {
        # Politique par défaut : bloquer le transit de paquets
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        # Politique par défaut : autoriser tout le trafic sortant
        type filter hook output priority 0; policy accept;
    }
}
```

### Vérification et persistance nftables

```bash
# Afficher le ruleset actif
nft list ruleset

# Charger la configuration depuis le fichier
nft -f /etc/nftables.conf

# Sauvegarder le ruleset actuel dans le fichier de configuration
nft list ruleset > /etc/nftables.conf
```

---

## UFW

### Installation et activation UFW

```bash
# Installer UFW (Uncomplicated Firewall)
apt install -y ufw

# Activer UFW au démarrage et appliquer les règles
ufw enable
```

### Réinitialiser les règles existantes UFW

```bash
# Supprimer toutes les règles existantes et désactiver UFW temporairement
ufw reset
```

### Politique par défaut UFW

```bash
# Bloquer tout le trafic entrant par défaut
ufw default deny incoming

# Bloquer le routage/transit de paquets
ufw default deny routed

# Autoriser tout le trafic sortant
ufw default allow outgoing
```

### Autoriser le loopback (optionnel, géré par défaut) UFW

```bash
# Autoriser explicitement le trafic sur l'interface loopback
ufw allow in on lo
```

### Autoriser SSH uniquement depuis une IP spécifique UFW

```bash
# Remplacer 10.42.0.2 par l'adresse IP autorisée (ex: poste administrateur)
ufw allow from 10.42.0.2 to any port 22 proto tcp
```

### Autoriser les connexions HTTP/HTTPS (optionnel, si besoin d'exposer un service web) UFW

```bash
# Autoriser HTTP (port 80) et HTTPS (port 443) depuis n'importe quelle IP
ufw allow 80/tcp
ufw allow 443/tcp
```

### Vérification UFW

```bash
# Afficher le statut d'UFW et toutes les règles actives
ufw status verbose
```

![ufw](/images/2026-03-02-12-25-35.png)

---

## Sécurité SSH

### Prérequis

Prendre la machine sur laquelle était déjà installé UFW (voir démo précédente) et s'assurer que le port SSH est ouvert :

```bash
ufw allow 22/tcp
```

### Générer une paire de clés SSH sur l'hôte

```bash
ssh-keygen -t ed25519 -C "admin@monserveur"
```

### Copier la clé publique sur le serveur

```bash
# Remplacer par l'IP ou le nom de votre serveur
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@10.0.0.100
```

### Gérer les permissions du dossier .ssh

```bash
# Sur le serveur, vérifier les permissions du dossier .ssh
ls -ld ~/.ssh
# Si nécessaire, corriger les permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519
```

### Tester la connexion SSH

```bash
ssh -o PubkeyAuthentication=yes root@10.0.0.100
```

### Création d'un nouvel utilisateur avec accès SSH

```bash
# Sur le serveur, créer un nouvel utilisateur
sudo adduser admin
sudo usermod -aG sudo admin
# Copier la clé publique pour le nouvel utilisateur
sudo mkdir -p /home/admin/.ssh
sudo touch /home/admin/.ssh/authorized_keys
sudo cp ~/.ssh/authorized_keys /home/admin/.ssh/authorized_keys
sudo chown -R admin:admin /home/admin/.ssh
sudo chmod 700 /home/admin/.ssh
sudo chmod 600 /home/admin/.ssh/authorized_keys
```

### Modifier la configuration SSH

```bash
# Sur le serveur, éditer le fichier de configuration SSH
sudo nano /etc/ssh/sshd_config

# Exemple de modifications recommandées :
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers admin

# Redémarrer le service SSH pour appliquer les changements
sudo systemctl restart ssh
```

### Vérifier que le mot de passe est désactivé

```bash
ssh -o PubkeyAuthentication=no admin@10.0.0.100
# Permission denied (publickey).
```

### Durcissement supplémentaire (optionnel)

```bash
# Limiter les tentatives de connexion pour prévenir les attaques par force brute
MaxAuthTries 3
LoginGraceTime 30
# Changer le port
Port 2222
# Timeout pour les connexions inactives
ClientAliveInterval 300
ClientAliveCountMax 0
# Désactiver les méthodes inutiles
ChallengeResponseAuthentication no
UsePAM no
X11Forwarding no
AllowTcpForwarding no

## Ne pas oublier de mettre à jour les règles de pare-feu si vous changez le port SSH
ufw allow 2222/tcp

## Redémarrer le poste après les modifications (changement du port SSH, etc.)
reboot
```

### Vérifier la configuration

```bash
# Vérifier la syntaxe du fichier de configuration SSH
sudo sshd -t

# Redémarrer le service SSH après les modifications
sudo systemctl restart ssh
```
