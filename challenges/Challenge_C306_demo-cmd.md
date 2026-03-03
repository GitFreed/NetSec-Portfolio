# Récapitulatif de configurations 🔐 C306 03/03/2026

Ce document présente des configurations PAM, Google Authenticator, last, lastb, fail2ban, knockd et crodwsec.

[Challenge C306](./Challenge_C306.md)

[Cours C306.](/RESUME.md#️-c306-pam-logs-fail2ban--port-knocking)

---

## Démonstration de PAM et Google Authenticator

### Installation de Google Authenticator

```bash
# Installer le module PAM pour Google Authenticator
apt install -y libpam-google-authenticator
# Installer l'application mobile "Google Authenticator" sur votre smartphone
```

### Configuration de Google Authenticator pour un utilisateur

```bash
# Se connecter en tant que l'utilisateur pour lequel vous souhaitez configurer 2FA
google-authenticator
```

### Configuration de PAM pour SSH

```bash
# Éditer le fichier de configuration PAM pour SSH
sudo nano /etc/pam.d/sshd

# Ajouter la ligne suivante pour activer Google Authenticator
auth required pam_google_authenticator.so

# On peut aussi rajouter l'option "nullok" pour permettre aux utilisateurs sans 2FA de se connecter (optionnel)
# auth required pam_google_authenticator.so nullok

# Désactiver le common-auth pour éviter les conflits
# Commenter ou supprimer la ligne suivante (si présente)
# @include common-auth
```

### Configuration de SSH pour exiger l'authentification à deux facteurs

```bash
# Éditer le fichier de configuration SSH
sudo nano /etc/ssh/sshd_config

# Ajouter ou modifier les lignes suivantes
KdbInteractiveAuthentication yes
ChallengeResponseAuthentication yes
AuthenticationMethods publickey,keyboard-interactive

# Redémarrer le service SSH pour appliquer les changements
sudo systemctl restart ssh
```

### Test de la connexion SSH avec 2FA

```bash
# Se connecter en SSH, vous serez invité à entrer le code généré par l'application Google
ssh -o PubkeyAuthentication=yes root@10.0.0.100
```

### Autre exemple - définir la taille des mots de passe

#### Installer le paquet pour la gestion des mots de passe

```bash
apt install -y libpam-pwquality
```

#### Configurer les exigences de mot de passe

```bash
sudo nano /etc/security/pwquality.conf

# Ajouter ou modifier les lignes suivantes pour exiger des mots de passe d'au moins 12 caractères
minlen = 12
# Vous pouvez aussi ajouter d'autres exigences, par exemple :
# dcredit = -1  # Exiger au moins un chiffre
# ucredit = -1  # Exiger au moins une lettre majuscule
# lcredit = -1  # Exiger au moins une lettre minuscule
# ocredit = -1  # Exiger au moins un caractère spécial

# Nombre minimum de caractères différents
minclass = 4

# Interdire les mots de passe communs
dictcheck = 1

# Nombre minimum de caractères spéciaux
minspecial = 1
```

### Modifier la configuration PAM pour les mots de passe

```bash
# Éditer le fichier de configuration PAM pour les mots de passe
sudo nano /etc/pam.d/common-password

# Ajouter la ligne suivante pour exiger des mots de passe d'au moins 12 caractères
password requisite pam_pwquality.so retry=3 minlen=12 minclass=4 difok=4
```

### Tester la création d'un nouvel utilisateur avec un mot de passe

```bash
# Créer un nouvel utilisateur
sudo adduser testuser
# Vous serez invité à entrer un mot de passe pour cet utilisateur, respectant les exigences définies
```

---

## Démo de last, lastb, fail2ban et knockd

### Installation d'une machine virtuelle de test

Créer une machine LXC sur proxmox.  
Attention à bien partir sur une image Ubuntu 24.04 ou Debian. Le package lastb n'est pas disponible sur les versions plus récentes d'Ubuntu (25.04+).

### SSH sur machine locale

```sh
# Mettre à jour la machine
sudo apt update && sudo apt upgrade -y

# Sur le serveur
sudo sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh
ip a

# en Local (se connecter en utilisant le mot de passe root)
ssh -o PubkeyAuthentication=no root@10.0.0.99
```

### Afficher les dernières connexions réussies

```bash
last

# Les 10 dernières connexions
last -n 10

# Connexion avec un utilisateur spécifique
last root

# Avec les adresses IP
last -a
```

### Afficher les dernières connexions échouées

```bash
sudo lastb
```

### Inspecter les logs d'authentification

```bash
sudo tail -f /var/log/auth.log
```

### Installation de fail2ban

```bash
sudo apt install -y fail2ban
```

### Configuration de fail2ban

```bash
# Créer une copie du fichier de configuration par défaut
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail

# Éditer le fichier de configuration
sudo nano /etc/fail2ban/jail

# Exemple de configuration pour SSH
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 5
# Durée du bannissement en secondes (1 heure)
bantime = 3600
# Durée pendant laquelle les tentatives sont comptées (10 minutes)
findtime = 600
# Action à prendre en cas de bannissement (bloquer avec iptables)
action = iptables-multiport
# Ignorer certaines IP (ex: votre IP d'administration)
ignoreip = 127.0.0.1/8
```

### Redémarrer fail2ban pour appliquer la configuration

```bash
sudo systemctl restart fail2ban
```

### Vérifier le statut de fail2ban

```bash
sudo fail2ban-client status sshd
```

On peut aussi vérifier les IP bannies :

```bash
sudo fail2ban-client status sshd
sudo fail2ban-client get sshd banned

# Débannir une IP spécifique
sudo fail2ban-client set sshd unbanip <IP_ADDRESS>

# Confirmer la règle firewall pour l'IP bannie
sudo iptables -L -n -v | grep <IP_ADDRESS>
```

### Installation de knockd (port knocking)

```bash
sudo apt install -y knockd

## Activer le service knockd
nano /etc/default/knockd

# Modifier la ligne suivante pour activer knockd au démarrage
START_KNOCKD=1
# Interfaces à écouter (optionnel, par défaut toutes les interfaces)
# KNOCKD_OPTS="-i eth0"

sudo systemctl enable knockd
sudo systemctl start knockd
```

### Préparer la configuration de knockd

```bash
# Réinitialiser les règles iptables pour s'assurer que SSH est bloqué par défaut
sudo iptables -F
sudo iptables -X
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# S'assurer que SSH est bloqué par défaut (ex: avec iptables)
sudo iptables -A INPUT -p tcp --dport 22 -j DROP
```

### Configuration de knockd

```bash
# Éditer le fichier de configuration de knockd
sudo nano /etc/knockd.conf

# Exemple de configuration pour ouvrir le port SSH après une séquence de ports
[options]
    UseSyslog
[openSSH]
    sequence    = 7000,8000,9000
    seq_timeout = 5
    # Penser à changer ici le -A en -I si vous avez déjà une règle iptables qui bloque SSH
    command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
    tcpflags    = syn
[closeSSH]
    sequence    = 9000,8000,7000
    seq_timeout = 5
    command     = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
    tcpflags    = syn
```

### Vérifier que SSH est bien bloqué par défaut

```bash
nmap -p 22 server_ip
# Port 22 doit être fermé (filtered)
```

### Tester knockd

```bash
# Depuis une machine cliente, envoyer la séquence de ports pour ouvrir SSH
knock server_ip 7000 8000 9000

# Alternativement, utiliser nmap pour envoyer les "knocks"
nmap -Pn --max-retries=0 --host-timeout=1000ms --send-ip server_ip -p 7000,8000,9000

# Côté serveur, on vérifie 
sudo tail -f /var/log/syslog

2026-03-01T10:15:10.505851+00:00 fail2ban knockd: 10.42.0.2: openSSH: Stage 1
2026-03-01T10:15:10.505956+00:00 fail2ban knockd: 10.42.0.2: openSSH: Stage 2
2026-03-01T10:15:10.505983+00:00 fail2ban knockd: 10.42.0.2: openSSH: Stage 3
2026-03-01T10:15:10.506007+00:00 fail2ban knockd: 10.42.0.2: openSSH: OPEN SESAME
2026-03-01T10:15:10.506153+00:00 fail2ban knockd: openSSH: running command: /sbin/iptables -A INPUT -s 10.42.0.2 -p tcp --dport 22 -j ACCEPT

# Vérifier que le port SSH est maintenant accessible
ssh -o PubkeyAuthentication=no root@server_ip

# Après avoir terminé, envoyer la séquence inverse pour fermer SSH
knock server_ip 9000 8000 7000

# Alternativement, utiliser nmap pour envoyer les "knocks" de fermeture
nmap -Pn --max-retries=0 --host-timeout=1000ms --send-ip server_ip -p 9000,8000,7000
```

### Bonus : crowdsec à la place de fail2ban

Une page est disponible en ligne pour présenter crowdsec, une alternative moderne à fail2ban avec une approche plus communautaire et des fonctionnalités avancées de détection d'attaques.

[Protéger son serveur Linux des attaques avec crowdsec](https://www.it-connect.fr/comment-proteger-son-serveur-linux-des-attaques-avec-crowdsec/)

Réinitialisation d'abord les règles iptables pour supprimer les règles ajoutées par fail2ban et knockd :

```bash
sudo iptables -F
sudo iptables -X
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT
```

Ensuite, on peut installer crowdsec qui est une alternative moderne à fail2ban avec une approche plus communautaire et des fonctionnalités avancées de détection d'attaques.

```bash
# Désinstaller fail2ban
sudo apt remove -y fail2ban
# Installer crowdsec
curl -s https://install.crowdsec.net | sudo sh
apt install -y crowdsec

# Vérifier le statut de crowdsec
sudo cscli metrics
sudo cscli decisions list
```

### Test de la protection avec crowdsec

```bash
# Test avec hydra pour simuler des tentatives de connexion échouées
sudo apt install -y hydra

# Télécharger le dictionnaire de mots de passe (ex: rockyou.txt)
wget https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt

# Lancer une attaque de force brute contre SSH
hydra -l root -P rockyou.txt -f -o hydra_output.txt ssh://server_ip

## Par défaut, en local, crowdsec ne bannira pas l'IP car elle fait partie de la liste d'ignore (réseau local). Pour tester le bannissement, il faudrait faire les tentatives de connexion échouées depuis une machine cliente avec une IP différente de celle du serveur.

## On veut quand même pouvoir le simuler
sudo nano /etc/crowdsec/parsers/s02-enrich/whitelists.yaml

# Chercher la partie
name: private ipv4/ipv6 ip/ranges

# Et commenter les lignes suivantes pour permettre le bannissement des IP locales
# name: private ipv4/ipv6 ip/ranges

sudo systemctl restart crowdsec

# Relancer l'attaque de force brute avec hydra depuis la machine cliente
hydra -l root -P rockyou.txt -f -o hydra_output.txt ssh

# On va voir que crowdsec fait une détection de l'attaque et ajoute une décision de bannissement pour l'IP de la machine cliente (mais ne bloque pas l'IP du serveur)
# Néammoins, on va voir que l'attaque ralentit considérablement après quelques tentatives échouées, ce qui montre que crowdsec est efficace pour limiter les attaques de force brute même si l'IP du serveur n'est pas bannie.
sudo cscli decisions list

# Pour voir le scenario brute force
nano /etc/crowdsec/scenarios/ssh-bf.yaml

# Supprimer les décisions de bannissement pour réinitialiser l'état
sudo cscli decisions delete --all

# Vérifier les décisions après l'attaque
sudo cscli decisions list

# Installer le bouncer pour bloquer les IP bannies avec iptables
sudo cscli bouncers add crowdsec-firewall-bouncer 
nano /etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml
# Modifier la clé d'API et activer le bouncer
# api_key: <clé_d'API_générée_précédemment>
sudo systemctl restart crowdsec

# Vérifier que tout est bien pris en compte
journalctl -u crowdsec-firewall-bouncer -f
iptables -L -n -v

# Vérifier les IP bannies dans iptables
ipset list crowdsec-blacklists-0

```

---
