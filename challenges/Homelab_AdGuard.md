# 🛡️ LAB : Maîtrise du flux DNS et Sécurisation avec AdGuard Home

```txt
   _       _   ___                     _ 
  /_\   __| | / _ \_   _  __ _ _ __ __| |
 //_\\ / _` |/ /_\/ | | |/ _` | '__/ _` |
/  _  \ (_| / /_\\| |_| | (_| | | | (_| |
\_/ \_/\__,_\____/ \__,_|\__,_|_|  \__,_|
                                         
```

![Hardware](https://img.shields.io/badge/Matériel-Raspberry%20Pi%203B-C51A4A?style=flat-square&logo=raspberrypi&logoColor=white)
![OS](https://img.shields.io/badge/OS-Raspberry%20Pi%20OS%20Lite-005085?style=flat-square&logo=linux&logoColor=white)
![Service](https://img.shields.io/badge/Service-AdGuard%20Home-68bc71?style=flat-square&logo=adguard&logoColor=white)
![Role](https://img.shields.io/badge/Rôle-Serveur%20DNS%20%2B%20DHCP-FFA500?style=flat-square)
![Network](https://img.shields.io/badge/Réseau-LAN%20192.168.1.0%2F24-purple?style=flat-square&logo=web&logoColor=white)

**Rôle :** Administrateur Réseau

**Mission :** AdGuard Home est un **DNS sinkhole** réseau. Il intercepte toutes les requêtes DNS du réseau local et redirige celles pointant vers des domaines de publicité, de trackers et de malwares vers un « puits noir » (sinkhole), empêchant les appareils d'établir une connexion avec ces serveurs. Le filtrage s'applique au niveau DNS — avant même que le navigateur ou l'application n'envoie une seule requête HTTP — ce qui protège l'ensemble des appareils connectés (PC, smartphones, tablettes, téléviseurs, IoT) sans nécessiter d'installation logicielle sur chaque appareil.

> 📚 Documentation : <https://github.com/AdguardTeam/AdguardHome>

---

## L'intérêt technique 🎯

1. **Visibilité Réseau (Layer 7) :** Intercepter, analyser et filtrer le trafic DNS de l'ensemble du réseau. Chaque requête est journalisée — on voit exactement quel appareil communique avec quel domaine.
2. **Performance (Caching) :** AdGuard Home conserve en cache les réponses DNS. Les requêtes suivantes sont servies localement en **~1ms** au lieu de **~20ms** via un résolveur distant. Les domaines bloqués ne génèrent aucun trafic réseau.
3. **Sécurité (Première ligne de défense) :** Les domaines malveillants, de phishing ou de C2 (Command & Control) sont bloqués avant même que le pare-feu n'ait à traiter le paquet IP. Un appareil IoT compromis qui tente de contacter son serveur C2 sera bloqué au niveau DNS.
4. **Contrôle DHCP :** En reprenant le rôle de serveur DHCP de la box FAI, AdGuard Home garantit que chaque appareil du réseau utilise exclusivement le DNS filtrant. Aucun contournement possible.

---

## 🛠️ Architecture du Lab

* **Matériel :** Raspberry Pi 3B
* **OS :** Raspberry Pi OS Lite (sans interface graphique, dédié à la performance réseau)
* **Position :** Serveur DNS et DHCP unique du réseau LAN, en remplacement des services de la box FAI
* **Réseau :** `192.168.1.0/24`
* **Passerelle FAI :** `192.168.1.254`
* **IP Raspberry Pi :** `192.168.1.250`

### Schéma de flux DNS

```txt
                    ┌──────────────────────────────────────────┐
                    │        Raspberry Pi (192.168.1.250)      │
                    │                                          │
Appareils LAN ────► │  AdGuard Home                            │
  (DHCP :53)        │    ├── Filtrage DNS (pubs/trackers/C2)   │
                    │    ├── Cache local (~1ms)                │
                    │    ├── DHCP Server (.50 → .150)          │
                    │    └── Upstream → Unbound :5335          │
                    │                    └── Serveurs racine   │
                    └──────────────────────────────────────────┘
                                      │
                              Box FAI (192.168.1.254)
                              Conditional forwarding
                              (.bytel.fr / .lan uniquement)
```

> 💡 **Note :** les résolutions DNS en amont sont gérées par **Unbound** (résolveur récursif local) depuis avril 2026. Voir la fiche dédiée : [LAB Unbound](/challenges/Homelab_Unbound.md)

---

## 🚀 Installation

### Pré-requis : Raspberry Pi OS Lite

> 📚 **Ressources :**
>
> * Raspberry Pi OS Lite : <https://www.raspberrypi.com/software/operating-systems/>
> * Raspberry Pi Imager : <https://www.raspberrypi.com/software/>

La version **Lite** (sans bureau graphique) est impérative pour un serveur DNS. Pas d'interface graphique inutile qui consomme de la RAM et du CPU — le Pi est dédié à la performance réseau.

Lancer **Raspberry Pi Imager** pour flasher la carte micro SD.

Personnaliser la configuration : hostname (`adguard-pi`), user admin, mot de passe, et **activer le SSH** (indispensable pour l'administration à distance).

![done](/images/2026-01-21-00-20-36.png)

Insérer la carte micro SD dans le Pi, le brancher en Ethernet à la box, et le démarrer.

---

### 1. Attribution d'une IP statique

Avant d'installer quoi que ce soit, il faut figer l'adresse IP du Pi. Un serveur DNS dont l'IP changerait rendrait tout le réseau inaccessible.

Sur l'interface de la box (`192.168.1.254`) → Réglages avancés → DHCP → Attribution d'adresse IP statique.

Pour identifier le Pi dans la liste des appareils, chercher le hostname `adguard-pi` ou une adresse MAC commençant par `b8:27:eb` ou `dc:a6:32` (préfixes OUI du fabricant Raspberry Pi Foundation).

![DHCP](/images/2026-01-21-11-45-54.png)

Débrancher/rebrancher le câble Ethernet pour forcer le renouvellement du bail.

Vérification : `ping 192.168.1.250`

---

### 2. Installation d'AdGuard Home (SSH)

Connexion SSH au Pi :

```bash
ssh user@192.168.1.250
```

Lancer le script d'installation officiel :

```bash
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v
```

> ⚠️ **Sécurité :** configurer également une IP statique côté OS (en plus du bail statique de la box), pour garantir la stabilité même en cas de redémarrage de la box.

Lancer `sudo nmtui` pour configurer l'interface réseau :

![nmt](/images/2026-01-21-16-17-15.png)

Passer l'IPv4 en **Manuel**, renseigner l'adresse IP, la passerelle et le DNS en `127.0.0.1` (le Pi utilisera son propre service AdGuard pour résoudre les noms). Valider et redémarrer : `sudo reboot`

Installation de **Btop** pour le monitoring des ressources en temps réel :

```bash
sudo apt update && sudo apt install btop -y
```

![Btop](/images/2026-01-21-16-53-37.png)

---

### 3. Configuration initiale (Interface Web)

Ouvrir un navigateur sur `http://192.168.1.250:3000` pour accéder à l'assistant de configuration.

Configuration des interfaces d'écoute :

> ⚠️ **Le serveur DNS doit impérativement écouter sur le port `53` (UDP/TCP).** C'est le port standard que tous les appareils utilisent par défaut pour les requêtes DNS.

Une fois l'assistant terminé, l'interface d'administration est accessible sur le port 80 :

![login](/images/2026-01-21-12-14-17.png)

---

### 4. Bascule DNS du réseau

#### 4.1 — Redirection DNS sur la box

Sur l'interface de la box → Réglages avancés → DHCP → Options : renseigner `192.168.1.250` comme serveur DNS distribué aux clients.

![options](/images/2026-01-21-12-28-03.png)

#### 4.2 — Alias DNS local

💡 Création d'une **réécriture DNS** pour accéder à AdGuard via un nom convivial. Dans AdGuard Home → Filtres → Réécritures DNS, ajouter une entrée pointant `adguard.home` vers l'IP du Pi.

L'interface est désormais accessible à l'adresse `http://adguard.home` depuis n'importe quel appareil du réseau.

![dash](/images/2026-01-21-13-33-19.png)

#### 4.3 — Serveurs DNS upstream

AdGuard Home ne sait pas résoudre les noms locaux du réseau (hostnames de la box, `.lan`, `.bytel.fr`). Ces domaines n'existent pas dans le DNS public — seule la box les connaît. Il faut ajouter des règles de **conditional forwarding** dans les paramètres DNS upstream :

```sh
[/bytel.fr/]192.168.1.254
[/lan/]192.168.1.254
```

Ces lignes indiquent à AdGuard de rediriger les requêtes pour les domaines locaux vers la box FAI (`192.168.1.254`), qui est la seule à pouvoir les résoudre.

Le DNS upstream principal utilisait initialement **Quad9** en DoH (DNS-over-HTTPS, port 443) pour chiffrer les requêtes DNS sur le réseau. La version `9.9.9.10` "Unsecured" laisse AdGuard Home gérer lui-même le filtrage de sécurité sans doublon avec celui de Quad9.

Des DNS de repli (Cloudflare et Quad9 classique, toujours en DoH) étaient configurés pour éviter un **SPOF** (Single Point Of Failure).

> 💡 **Note :** depuis avril 2026, Quad9 et Cloudflare ont été remplacés par **Unbound** (résolveur récursif local) comme unique upstream. Voir la fiche [LAB Unbound](/challenges/Homelab_Unbound.md) pour le détail et les raisons de ce changement plus que conseillé.

---

### 5. Bascule DHCP — Contrôle total du réseau

#### Le problème

La box Bouygues impose son propre serveur DNS à tous les appareils via le DHCP, même si un DNS alternatif est configuré. Cette règle du FAI n'est pas modifiable :

![DNS](/images/2026-01-21-15-52-31.png)

#### La solution

Désactiver le serveur DHCP de la box et activer celui d'AdGuard Home. Ainsi, tous les appareils obtiennent leur configuration réseau (IP, passerelle, DNS) directement depuis le Pi. Aucun appareil ne peut contourner le filtrage DNS.

Configuration DHCP dans AdGuard Home :

| Paramètre | Valeur |
| --- | --- |
| Interface | `eth0` |
| Passerelle | `192.168.1.254` (box FAI) |
| Plage IPv4 | `192.168.1.50` → `192.168.1.150` |
| Masque | `255.255.255.0` |
| Durée du bail | `86400s` (24h) |
| Plage IPv6 (ULA) | `fd00::10` → `fd00::ff` |

> ⚠️ **Procédure critique :** désactiver le DHCP de la box **puis** activer immédiatement celui d'AdGuard Home. Pendant la bascule, aucun appareil ne pourra obtenir de nouvelle adresse IP. Il est recommandé de garder une fenêtre SSH ouverte sur le Pi pour intervenir rapidement.

---

### 6. Problème rencontré : renouvellement DHCP post-bascule

Après la bascule DHCP, plusieurs appareils ont rencontré des problèmes de connectivité.

**Symptôme 1 — PC Windows :** impossible de renouveler l'adresse IP après redémarrage. Erreur NCB (Network Control Block) dans les logs :

![NCB](/images/2026-01-21-18-51-29.png)

**Tentatives de résolution (sans succès) :**

```powershell
# Vidange du cache DNS
ipconfig /flushdns
# Reset du catalogue Winsock
netsh winsock reset
# Réinitialisation de la pile TCP/IP
netsh int ip reset
# Désinstallation/réinstallation de la carte réseau via le Gestionnaire de périphériques
```

**Solution :** passage du PC en IP fixe (hors plage DHCP).

**Symptôme 2 — Appareils Wi-Fi :** après un reboot de la box, la plupart des appareils Wi-Fi ne parvenaient plus à se reconnecter. Forcer la reconnexion d'un appareil en IP fixe a semblé déclencher un renouvellement de la table ARP, après quoi les autres appareils ont suivi.

**Leçon retenue :** les appareils principaux (PC, serveurs, NAS) doivent être en IP fixe ou en bail statique, hors de la plage DHCP. Cela élimine les problèmes de renouvellement de bail et garantit l'accessibilité permanente des services critiques.

---

## 🔒 Listes de filtrage

Quatre listes complémentaires couvrent le spectre publicité + tracking + sécurité :

| Liste | Cible | Description |
| --- | --- | --- |
| **AdGuard DNS filter** | Pubs & Trackers | Filtre natif et généraliste. Assure la base du blocage des publicités et traceurs sur le web moderne. |
| **AdAway Default Blocklist** | Pubs mobiles | Liste légère et historique, particulièrement efficace contre les publicités au sein des applications mobiles (Android/iOS). |
| **OISD (The Big One)** | Pubs, Trackers, Télémétrie | La référence : liste massive qui agrège des milliers de sources tout en garantissant un taux de faux positifs quasi nul. <https://big.oisd.nl> |
| **Abuse.ch / URLHaus** | Malwares & Botnets | Projet communautaire de référence pour traquer et bloquer les domaines servant à la distribution de malwares, virus et botnets. <https://urlhaus.abuse.ch/downloads/hostfile/> |

---

## 🔄 Sauvegarde

### Sauvegarde manuelle

Toute la configuration d'AdGuard Home est contenue dans un seul fichier : `/opt/AdGuardHome/AdGuardHome.yaml`. Une sauvegarde rapide via SSH et `scp` (Secure Copy) :

```bash
# Sur le Pi : copier le fichier et ajuster les permissions
sudo cp /opt/AdGuardHome/AdGuardHome.yaml ~/AdGuardHome_backup.yaml
sudo chown freed:freed ~/AdGuardHome_backup.yaml
```

```powershell
# Depuis un terminal Windows : récupérer le fichier
scp freed@192.168.1.250:~/AdGuardHome_backup.yaml .
```

### Script de sauvegarde automatisé (PowerShell)

Pour automatiser la sauvegarde, création d'une **clé SSH dédiée** (ed25519, sans passphrase) réservée aux scripts :

```powershell
# Génération de la clé (Entrée à tout pour valider sans passphrase)
ssh-keygen -t ed25519 -f "C:\Users\<user>\.ssh\<key>"

# Déploiement de la clé publique sur le Pi
Get-Content "C:\Users\<user>\.ssh\<key>.pub" | ssh freed@192.168.1.250 "cat >> ~/.ssh/authorized_keys"
```

> ⚠️ **Sécurité :** cette clé sans passphrase est strictement dédiée aux scripts de sauvegarde. L'accès SSH interactif utilise une clé distincte protégée par passphrase.

Le script sauvegarde à la fois la configuration AdGuard Home et la configuration Unbound. Il utilise un tableau de fichiers extensible :

```powershell
# --- CONFIGURATION ---
$User = "freed"
$IP = "192.168.1.250"
$KeyPath = "C:\Users\<user>\.ssh\<key>"
$Date = Get-Date -Format "yyyyMMdd"
$LocalPath = $PSScriptRoot

# Fichiers à sauvegarder (source sur le Pi → nom local)
$Files = @(
    @{ Remote = "/opt/AdGuardHome/AdGuardHome.yaml"; Local = "AdGuardHome_backup_$Date.yaml" },
    @{ Remote = "/etc/unbound/unbound.conf.d/pi-unbound.conf"; Local = "Unbound_backup_$Date.conf" }
)

foreach ($File in $Files) {
    $FileName = Split-Path $File.Remote -Leaf
    $TempFile = "~/$($File.Local)"

    Write-Host "--- Sauvegarde de $FileName ---" -ForegroundColor Cyan

    Write-Host "  [1/3] Preparation sur le Pi..." -ForegroundColor Gray
    ssh -i $KeyPath $User@$IP "sudo cp $($File.Remote) $TempFile && sudo chown ${User}:${User} $TempFile"

    Write-Host "  [2/3] Telechargement..." -ForegroundColor Gray
    scp -i $KeyPath "$User@${IP}:$TempFile" "$LocalPath\$($File.Local)"

    Write-Host "  [3/3] Nettoyage..." -ForegroundColor Gray
    ssh -i $KeyPath $User@$IP "rm $TempFile"

    Write-Host "  OK : $($File.Local)" -ForegroundColor Green
    Write-Host ""
}

Write-Host "--- Sauvegarde Terminee ! ---" -ForegroundColor Green
Read-Host "Appuyez sur Entree pour fermer..."
```

---

## 📊 Statistiques

Moyenne des blocages DNS sur une semaine de fonctionnement :

![blocked](/images/2026-02-24-00-19-54.png)

---

## 📋 Résumé

* **Infrastructure :** Raspberry Pi 3B sous Raspberry Pi OS Lite
* **Services :** AdGuard Home (DNS sinkhole + DHCP) + Unbound (résolveur récursif)
* **Réseau :** Intégration transparente dans le LAN `192.168.1.0/24` — contrôle total du DNS et du DHCP
* **Sécurité :** Filtrage multicouche AdGuard DNS + AdAway + OISD + URLHaus (pubs, trackers, malwares, botnets)
* **Privacy :** Résolution DNS souveraine via Unbound (aucun DNS tiers dans la chaîne)
* **Supervision :** Btop (ressources temps réel) + tableau de bord AdGuard Home (stats DNS)
* **Automatisation :** Script PowerShell de sauvegarde (AdGuard + Unbound) via clé SSH dédiée

---

## 📚 Références

* AdGuard Home : <https://github.com/AdguardTeam/AdguardHome>
* Raspberry Pi OS : <https://www.raspberrypi.com/software/>
* OISD Blocklist : <https://oisd.nl/>
* URLHaus (Abuse.ch) : <https://urlhaus.abuse.ch/>
* LAB Unbound (résolveur récursif) : [Homelab_Unbound.md](/challenges/Homelab_Unbound.md)

---

![OK](/images/2026-01-22-08-17-56.png)

![requetes](/images/2026-01-22-08-18-37.png)
