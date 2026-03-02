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

Vérification en se connectant en SSH depuis l'IP enregistrée


### SSH Hardening (durcissement)
