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

## Protection automatisée contre les attaques avec Fail2ban

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

## Mise en place du port-knocking avec Knockd

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

## Défense collaborative et IPS avec CrowdSec

### Installation du moteur CrowdSec

L'installation nécessite d'ajouter le dépôt officiel, puis d'installer le paquet de base.

```bash
# Ajouter le dépôt officiel CrowdSec
apt install curl -y
curl -s https://install.crowdsec.net | sh

# Installer l'agent principal
apt install crowdsec -y

```

Une fois installé, CrowdSec détecte automatiquement les services présents sur la machine (comme SSH) et commence à surveiller leurs journaux.

![crowdsec](/images/2026-03-03-18-35-14.png)

### Installation du Pare-feu (Le Bouncer)

```bash
# Installer le bouncer iptables
apt install crowdsec-firewall-bouncer-iptables -y

```

*(Note : Sur système récent comme Debian13 tournant exclusivement sous nftables, le paquet `crowdsec-firewall-bouncer-nftables` est également disponible).*

Vérifier que le Bouncer est bien enregistré auprès du moteur :

```bash
cscli bouncers list

```

*(On doit voir ton firewall-bouncer avec la coche valide ✔️).*

![bouncer](/images/2026-03-03-18-36-03.png)

### Vérification de l'intégration Réseau

Dès le démarrage du Bouncer, il a créé de nouvelles chaînes de filtrage dans le pare-feu pour intercepter les paquets malveillants en priorité.

```bash
iptables -L -n

```

*(On doit y voir des chaînes nommées `crowdsec-blacklists`).*

Avec nftable :

```bash
nft list ruleset

```

*(On doit y voir une `table` nommée `crowdsec` (souvent `ip crowdsec` ou `inet crowdsec`), contenant un `set` (ensemble d'adresses IP) appelé `crowdsec-blacklists` et une règle ordonnant un `drop` immédiat pour les adresses de ce set).*

![rules](/images/2026-03-03-18-43-30.png)

### Test du système

Pour simuler la réaction du pare-feu en bannissant manuellement l'IP d'un attaquant test

```bash
cscli decisions add -i 192.168.1.151 -r "Test de validation IPS"
```

Pour simuler la réaction du pare-feu face à une attaque brute force, il faut désactiver l'immunité de notre lan

**Sur le serveur Crowdsec :**

```bash
nano /etc/crowdsec/parsers/s02-enrich/whitelists.yaml
```

```YAML
cidrs:
    - "127.0.0.1/8"
    - "::1/128"
    - "10.0.0.0/8"
    - "172.16.0.0/12"
    # - "192.168.0.0/16"
```

```bash
systemctl restart crowdsec
```

**Sur la VM attaquante :**

Cette commande va tenter de se connecter 7 fois de suite avec un utilisateur inexistant pour générer des erreurs dans les logs du serveur

```sh
for i in {1..7}; do ssh -o StrictHostKeyChecking=no Hackerman@192.168.1.153; done
```

Dès la 6ème tentative ratée, l'agent CrowdSec détecte le comportement anormal et ordonne immédiatement au Bouncer d'injecter la règle de Drop dans `nftables`.

![sshtry](/images/2026-03-03-19-08-35.png)

**Vérifier les menaces :**

```sh
# Cette commande liste tout l'historique des attaques locales que le moteur a détectées et sanctionnées.
cscli alerts list

# L'analyse détaillée
cscli alerts inspect <ID_DE_L_ALERTE>
```

![alert](/images/2026-03-03-19-26-42.png)

**Pour lever la sanction (et retrouver l'accès) :**

```bash
cscli decisions delete -i 192.168.1.151
# ou delete --all

```
