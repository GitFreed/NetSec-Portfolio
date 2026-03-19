# Challenge C504 19/03/2026

## 🧑‍🏫 Pitch de l’exercice

Le challenge du jour consiste à mettre en pratique vos compétences offensives en maîtrisant le framework Metasploit, l'exploitation de la faille EternalBlue, le piratage de réseaux Wi-Fi et l'élévation de privilèges sous Linux en bonus !

Réaliser les labs suivants sur TryHackMe :

- Metasploit <https://tryhackme.com/room/metasploitintro>
- Eternal Blue <https://tryhackme.com/room/blue> (optionnel)
- Wifi Hack <https://tryhackme.com/room/wifihacking101>
- Linus Privilege <https://tryhackme.com/room/linprivesc> (optionnel)

[Cours C504.](/RESUME.md#️-c504---vulnérabilités-attaques-wi-fi-et-sécurité-active-directory)

> 📚 **Ressources** :
>

---

## 💾 Résolution Lab Metasploit Framework

[TryHackMe - Metasploit Introduction](https://tryhackme.com/module/metasploit)

### **Task 1 : Introduction to Metasploit**

- **Concept :** Présentation du framework le plus utilisé en pentest, couvrant toutes les phases (de la reconnaissance à la post-exploitation).
- **Versions :** Metasploit Pro (payant, avec interface graphique) et Metasploit Framework (gratuit, open-source, en ligne de commande via `msfconsole`).

### **Task 2 : Main Components of Metasploit**

- **Composants :** Le framework se divise en 3 piliers : la console (`msfconsole`), les modules (exploits, payloads, scanners) et les outils annexes (comme `msfvenom`).
- **Vocabulaire :** * *Vulnerability* : La faille système.
  - *Exploit* : Le code qui attaque la faille.
  - *Payload* : Le code (Single ou Staged) qui s'exécute après l'exploitation pour atteindre ton but.

### **Task 3 : Msfconsole**

- **Fonctionnement :** C'est un terminal spécifique fonctionnant par "contexte". Quand on sélectionne un exploit, on "entre" dedans.
- **Navigation :** Utilisation des commandes `search` (pour trouver un module via mot-clé ou CVE), `use` (pour le sélectionner), `info` (pour lire ses détails et sa fiabilité/rank) et `back` (pour quitter le module). L'auto-complétion (touche TAB) est ton meilleure amie.

![console](/images/2026-03-19-17-47-58.png)

### **Task 4 : Working with modules**

- **Configuration :** Une fois dans un module, on utilise `show options` pour voir les paramètres requis (comme RHOSTS pour l'IP cible ou LPORT pour le port local).
- **Variables :** On assigne les valeurs avec `set` (valeur locale au module) ou `setg` (valeur globale conservée entre les modules).
- **Action :** On lance l'attaque avec `exploit` (ou `run`), et on peut gérer les accès obtenus avec `background` (pour masquer la session) et `sessions -i` (pour interagir avec une cible piratée).

### **Task 5 : Summary**

- **Conclusion :** Metasploit simplifie l'attaque en 3 étapes : trouver l'exploit, le personnaliser, et l'exécuter.

---

### 📝 Fiche Récap : Commandes Essentielles Metasploit

#### **1. Le Vocabulaire de base**

- **Vulnerability (Vulnérabilité) :** Une faille de conception ou de code sur le système cible.
- **Exploit :** Le bout de code qui tire parti de cette vulnérabilité (le bélier).
- **Payload (Charge utile) :** Le code qui s'exécute sur la cible une fois la faille exploitée pour obtenir un accès.
  - *Singles :* Payloads autonomes, tout-en-un (ex: `.../pingback_reverse_tcp`).
  - *Staged :* Payloads envoyés en plusieurs petits morceaux (ex: `.../meterpreter/reverse_tcp`).

#### **2. Navigation et Recherche (msfconsole)**

- **`msfconsole`** : Lance le framework Metasploit depuis ton terminal.
- **`search <mot-clé>`** : Cherche un module (ex: `search apache`, `search type:auxiliary telnet`, `search ms17-010`).
- **`use <chemin_ou_numero>`** : Sélectionne un module et entre dans son "contexte".
- **`info`** : Affiche les détails complets du module sélectionné (auteur, description, fiabilité/rank).
- **`back`** : Quitte le module actuel pour revenir au menu principal.

#### **3. Configuration des paramètres**

- **`show options`** : Affiche les paramètres requis pour faire fonctionner le module (RHOSTS, LPORT, etc.).
- **`show payloads`** : Affiche les charges utiles compatibles avec ton exploit actuel.
- **`set <paramètre> <valeur>`** : Configure une option (ex: `set LPORT 6666`).
- **`setg <paramètre> <valeur>`** : Configure une option *globalement* pour qu'elle reste sauvegardée (ex: `setg RHOSTS 10.10.19.23`).
- **`unset <paramètre>`** (ou `unset all`) : Efface la valeur d'un ou plusieurs paramètres.

#### **4. Exploitation et Gestion des Sessions**

- **`exploit`** (ou `run`) : Lance l'attaque.
- **`exploit -z`** : Lance l'attaque et met directement la session en arrière-plan dès qu'elle s'ouvre.
- **`background`** (ou `CTRL+Z`) : Met ta session active en arrière-plan pour te rendre la main sur la console Metasploit.
- **`sessions`** : Liste toutes les sessions (machines compromises) actives en arrière-plan.
- **`sessions -i <ID>`** : Interagit avec une session spécifique pour reprendre le contrôle (ex: `sessions -i 2`).

---

## 🟦 Résolution Lab Ethernal Blue

[TryHackMe - Metasploit Introduction](https://tryhackme.com/room/blue)

![blue](/images/2026-03-19-18-14-26.png)