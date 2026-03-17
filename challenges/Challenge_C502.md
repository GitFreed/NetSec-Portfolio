# Challenge C502 17/03/2026

## 🧑‍🏫 Pitch de l’exercice

Le challenge du jour consiste à :

- Réaliser les [3 premiers lab de PortSwigger sur les LFI](https://portswigger.net/web-security/all-labs#path-traversal),
- Réaliser le [challenge LFI root-me](https://www.root-me.org/fr/Challenges/Web-Serveur/Local-File-Inclusion)
- Réaliser le [challenge LFI double encoding root-me](https://www.root-me.org/fr/Challenges/Web-Serveur/Local-File-Inclusion-Double-encoding).

[Cours C501.](/RESUME.md#-c501-introduction-au-pentesting--faille-xss)

> 📚 **Ressources** :
>
> - Portswigger explication du Cross-site scripting (XXS) : <https://portswigger.net/web-security/cross-site-scripting>
> - Owasp explication détaillée du path traversal : <https://owasp.org/www-community/attacks/Path_Traversal>
> - Référence complète sur toutes les techniques LFI/RFI  : <https://hacktricks.wiki/en/pentesting-web/file-inclusion/index.html>
> - Liste exhaustive de payloads classés par technique : <https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion>
> - Documentation officielle PHP sur tous les wrappers disponibles : <https://www.php.net/manual/fr/wrappers.php>

---

### 📂 Résolution : Lab PortSwigger - File path traversal, simple case

<https://portswigger.net/web-security/file-path-traversal/lab-simple>

**Contexte :**
Le site charge des images pour illustrer ses produits. On va intercepter la requête qui demande cette image au serveur pour modifier le chemin du fichier ciblé.

#### **Étape 1 : Interception de la requête**

- Activer Burp Suite (ou l'inspecteur réseau du navigateur) et afficher les détails d'un produit.
- Repérer la requête qui va chercher l'image dans la Target. Elle contient un paramètre vulnérable : `filename=nom_de_l_image.jpg`.

![image](/images/2026-03-17-23-49-08.png)

#### **Étape 2 : L'attaque (Path Traversal)**

- Pour "remonter" dans l'arborescence du serveur, on utilise la technique du Path Traversal avec les caractères `../`.
- Dans le Repeater on remplace le nom de l'image par notre payload : `../../../etc/passwd`.
- *Explication :* Chaque `../` fait remonter le serveur d'un niveau dans ses dossiers. On remonte 3 fois pour atteindre la racine globale du serveur Linux (`/`), puis on redescend dans le dossier `/etc/` pour y lire le fichier `passwd`.
- Envoyer la requête modifiée au serveur.
- Le serveur s'exécute et, au lieu de renvoyer une image, il affiche le contenu brut du fichier `/etc/passwd` [la liste des utilisateurs du système Linux]. Le lab est validé !

![payload](/images/2026-03-17-23-50-14.png)

![solved](/images/2026-03-17-23-50-35.png)

---

### 📂 Résolution : Lab PortSwigger - File path traversal, sequences blocked with absolute path bypass

<https://portswigger.net/web-security/file-path-traversal/lab-absolute-path-bypass>

**Contexte :**
Le serveur possède une protection basique : il bloque ou supprime les séquences `../`. On ne peut donc plus utiliser de chemin "relatif" pour remonter l'arborescence.

#### **Étape 1 : Préparation dans Burp Suite**

- Comme pour le lab précédent, on envoie la requête de l'image (ligne `/image?filename=...`) dans le **Repeater**.

#### **Étape 2 : Le contournement (Chemin Absolu)**

- Puisque les `../` sont bloqués, on va donner au serveur le chemin direct et absolu du fichier, en partant de la racine `/`.
- On remplace la valeur du paramètre `filename` par : `/etc/passwd`
- La ligne de la requête devient donc : `GET /image?filename=/etc/passwd HTTP/2`
- On clique sur "Send".
- La sécurité du serveur est aveuglée car elle ne cherchait que des `../`. Le serveur lit le chemin absolu et affiche le contenu du fichier `/etc/passwd` dans la réponse. Le lab est validé !

![payload](/images/2026-03-17-23-58-30.png)

![solved](/images/2026-03-17-23-50-35.png)

---

### 📂 Résolution : Lab PortSwigger - File path traversal, sequences stripped non-recursively

**Contexte :**
Le serveur possède un filtre de sécurité qui repère et efface automatiquement les séquences `../` de l'URL. Cependant, ce filtre est "non-récursif" : il ne nettoie le texte qu'une seule fois et ne vérifie pas le résultat après son passage.

#### **Étape 1 : Le contournement (La technique de la poupée russe)**

- Pour tromper le filtre, on imbrique un `../` à l'intérieur d'un autre, ce qui donne : `....//`
- *L'astuce :* Quand le filtre du serveur lit `....//`, il supprime le `../` du milieu. Mais en faisant cela, les caractères restants à gauche (`..`) et à droite (`/`) se recollent. Magie : ça forme un nouveau `../` tout neuf ! Comme le filtre est déjà passé, ce nouveau `../` passe incognito.

#### **Étape 2 : L'attaque dans Burp Suite**

- Toujours dans le Repeater, sur la requête de l'image, on modifie le paramètre `filename`.
- On remplace le nom du fichier par notre payload imbriquée : `....//....//....//etc/passwd`
- La ligne 1 devient donc : `GET /image?filename=....//....//....//etc/passwd HTTP/2`
- En cliquant sur "Send", le filtre nettoie notre payload, la transformant en un classique `../../../etc/passwd` en arrière-plan.
- Le serveur remonte l'arborescence et affiche le fichier `/etc/passwd`. Le lab est validé !

![payload](/images/2026-03-18-00-03-27.png)

![solved](/images/2026-03-17-23-50-35.png)

---

### 📂 Résolution : Challenge Root-Me - Local File Inclusion (File viewer)

<https://www.root-me.org/fr/Challenges/Web-Serveur/Local-File-Inclusion>

**Contexte :**
L'application permet de naviguer dans des dossiers via le paramètre `?files=` (ex: `?files=sysadm`). Le but est d'exploiter une faille de type Path Traversal pour fouiller l'arborescence du serveur et trouver des informations confidentielles.

#### **Étape 1 : Reconnaissance (Path Traversal)**

- On remplace le nom du dossier légitime par `..` pour forcer le serveur à remonter au dossier parent.
- L'URL devient : `?files=..`
- Le serveur affiche le contenu de son répertoire parent. On y découvre un dossier caché très suspect nommé `admin`.

![admin](/images/2026-03-18-00-19-55.png)

#### **Étape 2 : Exploration du dossier secret**

- On modifie à nouveau l'URL pour demander au serveur de lister le contenu de ce dossier spécifique :
  `?files=../admin`
- Le serveur affiche les fichiers contenus dans la zone d'administration.
- En cliquant sur le fichier d'authentification listé dans le dossier `admin`, le serveur inclut son code PHP dans la page.
- En lisant le code brut, on découvre un tableau contenant les identifiants codés en dur : `$users = array('admin' => 'XXXXXX');`.

![admin](/images/2026-03-18-00-28-13.png)

- Le mot de passe `XXXXXX` est le flag ! 🏁

![OK](/images/2026-03-18-00-45-31.png)

---

### 📂 Résolution : Challenge Root-Me - LFI Double encoding

**Contexte :**
L'application est vulnérable aux inclusions de fichiers locaux (LFI) via le paramètre `?page=`. Cependant, un filtre de sécurité strict (WAF) bloque les caractères spéciaux et les tentatives classiques de lecture. L'objectif est de lire le code source du fichier de configuration local `conf.inc.php` (le script cible ajoutant automatiquement l'extension `.inc.php`).

#### **Étape 1 : Contournement par Wrapper PHP et Double Encodage**

- Pour lire le contenu d'un fichier PHP sans l'exécuter, la technique classique est d'utiliser un wrapper : `php://filter/convert.base64-encode/resource=conf`
- Pour passer sous le radar du pare-feu qui bloque les `:`, `/` et `=`, on encode l'URL deux fois de suite (le caractère d'échappement `%` devenant lui-même `%25`).
- La payload finale double-encodée devient : `php%253A%252F%252Ffilter%252Fconvert%252ebase64-encode%252Fresource%253Dconf`
- On l'injecte dans le paramètre vulnérable : `?page=php%253A%252F%252Ffilter%252Fconvert%252ebase64-encode%252Fresource%253Dconf`

![php](/images/2026-03-18-00-36-30.png)

#### **Étape 2 : Exfiltration et Décodage**

- Le pare-feu lit le premier niveau d'encodage, ne détecte aucune menace, et laisse passer. Le serveur web décode le deuxième niveau, exécute le wrapper, et renvoie le contenu du fichier `conf.inc.php` encodé en Base64.
- On copie la longue chaîne de caractères affichée sur la page.
- On la colle dans un outil comme [CyberChef](https://gchq.github.io/CyberChef/) avec la recette "From Base64".
- Le code source PHP s'affiche en clair. Tout en haut du fichier, avant le code CSS de la page, se trouve le tableau de configuration contenant le mot de passe.
- Le flag est : **XXXXXXX!** 🏁

![cooked](/images/2026-03-18-00-44-24.png)

![OK](/images/2026-03-18-00-45-31.png)

---
