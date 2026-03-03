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

### Installation
