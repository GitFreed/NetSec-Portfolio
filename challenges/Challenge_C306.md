# Challenge C306 03/03/2026

## 🧑‍🏫 Pitch de l’exercice : 🛡️ Sécurisation d'un serveur Linux contre les attaques par force brute

[Challenge C306](https://github.com/O-clock-Aldebaran/SC03E07-challenge-fail2ban)

[Cours C306.](/RESUME.md#️-c306-pam-logs-fail2ban--port-knocking)

[Récap des commandes et démo config](./Challenge_C306_demo-cmd.md)

---

### 🎯 Contexte

Vous êtes administrateur système dans une PME.

Un serveur Linux (Debian/Ubuntu) est exposé sur Internet et héberge un accès SSH. Les logs d'authentification révèlent des dizaines de tentatives de connexion échouées chaque nuit depuis des adresses IP inconnues.

Le responsable sécurité vous demande de mettre en place une protection automatisée contre ces attaques et, en bonus, de masquer complètement le port SSH aux scanners extérieurs.

### 🖥️ Environnement technique

* 1 machine virtuelle sous Debian ou Ubuntu (installation minimale)
* Accès console ou SSH root
* SSH déjà installé et fonctionnel (SSH de base suffit, il doit autorisé les connexions par mot de passe pour simuler les attaques)
* Une machine cliente pour effectuer les tests

### 🔎 Points de vigilance

* Ne jamais se bannir soi-même : toujours renseigner `ignoreip` avec votre IP ou votre réseau
* Toujours garder une **session console** ouverte avant de bloquer SSH au niveau firewall
* Tester la configuration dans une session séparée **avant** de fermer la session active
* `knockd` nécessite un accès **console de secours** en cas d'oubli de la séquence

---

## Protection automatisée contre les attaques avec fail2ban

### Installation VM & SSH

Montage d'un container Ubuntu24.04 LXC, IP statique 192.168.1.152

Installation et connection SSH

```sh
# Installation du serveur SSH
apt update && upgrade -y
apt install -y openssh-server

# Vérification du statut et du port d'écoute
systemctl status ssh

# Activation au démarrage
systemctl enable ssh

# Autoriser le Root login par mdp
sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin yes/' /etc/ssh/sshd_config

# Depuis l'hôte
# Suppression de la clef SSH utilisé pour l'ancienne VM sur l'IP 192.168.1.151
ssh-keygen -R 192.168.1.152

# Connexion par mot de passe avec root
ssh -o PubkeyAuthentication=no root@192.168.1.152
```

### Audit préalable : lire les logs de connexion

```sh
# Dernières connexions réussies
last -n 10

# Tentatives de connexion échouées (nécessite root)
lastb -n 10

# Surveiller auth.log en temps réel
tail -f /var/log/auth.log | grep "Failed password"
```

![lastlastb](/images/2026-03-03-17-08-44.png)

### Installation et configuration de fail2ban

```sh
apt update
apt install -y fail2ban

# Activer et démarrer le service
systemctl enable fail2ban
systemctl start fail2ban

# Vérifier que le service est actif
systemctl status fail2ban
```

Créer le fichier de configuration local (ne jamais modifier `jail.conf` directement) :

```sh
nano /etc/fail2ban/jail.local
```

Contenu minimal à mettre en place :

```ini
[DEFAULT]
# Durée du ban
bantime  = 1h

# Fenêtre d'observation
findtime = 10m

# Nombre d'échecs avant ban
maxretry = 3

# Ne jamais se bannir soi-même
ignoreip = 127.0.0.1/8 ::1

[sshd]
enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 1h
findtime = 10m
```

Puis redémarrer fail2ban pour appliquer la configuration :

```sh
systemctl restart fail2ban
```

### Vérification et test de fail2ban

#### Vérifier l'état du jail SSH

```sh
# Statut général
fail2ban-client status

# Statut du jail SSH spécifiquement
fail2ban-client status sshd
```

#### Tester le bannissement

Depuis la **machine cliente**, effectuer plusieurs tentatives de connexion avec un mauvais mot de passe :

```sh
# Répéter cette commande jusqu'à atteindre maxretry
ssh hackerman@192.168.1.152
```

Puis, **côté serveur**, vérifier que l'IP est bien bannie :

```bash
# Vérifier les IP bannies
fail2ban-client status sshd

# Confirmer la règle firewall générée
iptables -L -n -v | grep "fail2ban"
```

![ban](/images/2026-03-03-17-32-06.png)

#### Débannir une IP si nécessaire

```bash
fail2ban-client set sshd unbanip 192.168.1.5
```

### Ban progressif pour les récidivistes (recommandé)

Ajouter dans `jail.local` un jail qui détecte les IPs déjà bannies et prolonge leur exclusion :

```ini
[recidive]
enabled  = true
logpath  = /var/log/fail2ban.log
filter   = recidive
bantime  = 1w
findtime = 1d
maxretry = 3
```

> **Traduction :** 3 bans en 24h → banni pour 1 semaine.

---

## Mise en place du port-knocking avec knockd

Le port-knocking masque complètement SSH aux scanners : tant que la bonne séquence de ports n'est pas envoyée, le port 22 n'est tout simplement **pas visible**.

### Installation

```bash
apt install -y knockd
```

### Fermer SSH par défaut dans iptables

⚠️ **Gardez une session ouverte ou un accès console avant d'exécuter ces commandes !**

```bash
# Réinitialiser les règles iptables
iptables -F
iptables -X
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Autoriser le loopback et les connexions établies
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# SSH fermé par défaut — knockd ouvrira dynamiquement pour l'IP autorisée
iptables -A INPUT -p tcp --dport 22 -j DROP
```

### Configuration de knockd

```bash
nano /etc/knockd.conf
```

```ini
[options]
    UseSyslog

[openSSH]
    sequence    = 7000,8000,9000
    seq_timeout = 5
    command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
    tcpflags    = syn

[closeSSH]
    sequence    = 9000,8000,7000
    seq_timeout = 5
    command     = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
    tcpflags    = syn
```

> Passer -A en -I

### Activer knockd

```bash
# Modifier la configuration de démarrage
nano /etc/default/knockd
```

S'assurer que les lignes suivantes sont présentes :

```text
START_KNOCKD=1
KNOCKD_OPTS="-i eth0"
```

> Remplacer `eth0` par l'interface réseau réelle du serveur.

```bash
systemctl enable knockd
systemctl start knockd
```

### Test du port-knocking

**Côté client — vérifier que SSH est invisible :**

```bash
# Installer nmap
apt install nmap -y

nmap -p 22 192.168.1.152
# Résultat attendu : 22/tcp  filtered
```

![nmap](/images/2026-03-03-17-59-11.png)

**Côté client — envoyer la séquence d'ouverture :**

```bash
# Avec le client knock
knock 192.168.1.152 7000 8000 9000

# Alternative avec nmap si knock n'est pas disponible
for port in 7000 8000 9000; do
    nmap -Pn --max-retries=0 --host-timeout=1000ms -p $port 192.168.1.152
done
```

**Côté client — vérifier que SSH est maintenant accessible :**

```bash
nmap -p 22 192.168.1.152
# Résultat attendu : 22/tcp  open

ssh root@192.168.1.152
```

![ok](/images/2026-03-03-18-21-43.png)

**Côté serveur — vérifier les logs de knockd :**

```bash
tail -f /var/log/syslog | grep knockd
```

![OPEN](/images/2026-03-03-18-22-50.png)

**Côté client — refermer SSH après usage :**

```bash
knock 192.168.1.152 9000 8000 7000
```

---

## Défense Collaborative et IPS avec CrowdSec
