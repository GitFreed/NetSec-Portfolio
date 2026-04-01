# 🔓 LAB : Souveraineté DNS — Résolution Récursive avec Unbound

```txt
 _   _       _                           _ 
| | | |_ __ | |__   ___  _   _ _ __   __| |
| | | | '_ \| '_ \ / _ \| | | | '_ \ / _` |
| |_| | | | | |_) | (_) | |_| | | | | (_| |
 \___/|_| |_|_.__/ \___/ \__,_|_| |_|\__,_|
                                             
```

![Hardware](https://img.shields.io/badge/Matériel-Raspberry%20Pi-C51A4A?style=flat-square&logo=raspberrypi&logoColor=white)
![OS](https://img.shields.io/badge/OS-Raspbian-005085?style=flat-square&logo=linux&logoColor=white)
![Service](https://img.shields.io/badge/Service-Unbound%201.22-3776AB?style=flat-square&logo=dns&logoColor=white)
![Role](https://img.shields.io/badge/Rôle-Résolveur%20DNS%20Récursif-FFA500?style=flat-square)
![Sécurité](https://img.shields.io/badge/Sécurité-DNSSEC%20%2B%20QNAME%20Min.-red?style=flat-square)
![Network](https://img.shields.io/badge/Réseau-LAN%20-purple?style=flat-square&logo=web&logoColor=white)

**Rôle :** Administrateur Réseau / Sécurité

**Mission :** Unbound est un résolveur DNS récursif open-source qui interroge directement les serveurs racine d'Internet, sans passer par aucun intermédiaire tiers (Google, Cloudflare, Quad9). Couplé à AdGuard Home, il constitue une chaîne DNS entièrement souveraine : AdGuard filtre les publicités et trackers, Unbound résout les noms depuis la racine. Aucune entité extérieure ne peut imposer de blocage ou de censure sur les résolutions DNS du réseau.

> 📚 Documentation : <https://github.com/NLnetLabs/unbound>

---

## Contexte : Pourquoi Unbound ? 🎯

### Le problème : les DNS publics ne sont plus neutres

Depuis 2024, la justice française (sous impulsion de Canal+ et de l'ARCOM) contraint les fournisseurs de DNS alternatifs publics à bloquer l'accès à certains domaines. Chronologie de l'escalade :

| Année | Mesure |
| --- | --- |
| 2022 | Blocage des sites pirates par les **FAI** (Orange, Free, SFR, Bouygues) |
| 2024 | Extension aux **DNS alternatifs** (Google DNS, Cloudflare 1.1.1.1, Quad9, OpenDNS) |
| 2025 | Extension aux **VPN commerciaux** (NordVPN, ExpressVPN, ProtonVPN, CyberGhost, Surfshark) et aux **CDN/Proxy** |
| 2026 | Confirmation en appel (27 mars 2026) — validation du blocage DNS comme mesure proportionnée. Projet de **blocage par IP** prévu pour juin 2026 |

**Conséquence technique :** des services DNS autrefois neutres deviennent des **DNS menteurs** soumis aux injonctions des ayants droit. Quad9, fondation suisse à but non lucratif, a été contrainte d'appliquer les blocages **mondialement** faute de pouvoir filtrer géographiquement ses utilisateurs. OpenDNS (Cisco) a choisi de **quitter le marché français** plutôt que de se conformer.

### La solution : résoudre soi-même depuis la racine

Un résolveur récursif local comme Unbound interroge directement les serveurs racine du DNS mondial (les 13 root servers), puis descend l'arborescence DNS jusqu'au serveur autoritaire de chaque domaine. Aucun intermédiaire dans la chaîne → aucun point de censure externe.

### L'intérêt technique

1. **Souveraineté DNS :** Résolution directe depuis les serveurs racine, sans dépendance à un tiers.
2. **Sécurité (DNSSEC) :** Validation cryptographique des réponses DNS — détecte les réponses falsifiées ou altérées.
3. **Privacy (QNAME Minimisation) :** N'envoie que le strict nécessaire à chaque serveur autoritaire au lieu du nom de domaine complet (RFC 9156).
4. **Performance (Cache local) :** Première requête ~50-200ms (résolution complète depuis la racine), requêtes suivantes ~0ms (cache).
5. **Anti-rebinding :** Protection contre les attaques DNS rebinding (réponse externe contenant une IP privée).

---

## 🛠️ Architecture du Lab

* **Matériel :** Raspberry Pi 3B (même machine que [AdGuard Home](./Homelab_AdGuard.md))
* **OS :** Raspberry Pi OS (Lite)
* **Position :** En aval d'AdGuard Home, en amont des serveurs racine
* **Port d'écoute :** 5335/UDP+TCP (localhost uniquement)
* **Réseau :** 192.168.1.0/24
* **Serveur DNS/DHCP :** 192.168.1.250 (Raspberry Pi)

### Chaîne DNS complète

```txt
                        ┌─────────────────────────────────────────────┐
                        │          Raspberry Pi (192.168.1.250)       │
                        │                                             │
Appareils LAN ────────► │  AdGuard Home :53                           │
                        │    │ Filtrage pubs / trackers / malwares    │
                        │    │                                        │
                        │    ├──► Unbound :5335                       │
                        │    │      │ Résolution récursive directe    │
                        │    │      │ DNSSEC + QNAME Minimisation     │
                        │    │      └──► Serveurs racine DNS ────► Internet
                        │    │                                        │
                        │    └──► Box FAI :53 (192.168.1.254)         │
                        │           Conditional forwarding            │
                        │           (.bytel.fr / .lan uniquement)     │
                        └─────────────────────────────────────────────┘
```

### Comparatif : Avant / Après

| | Avant (Quad9 DoH) | Après (Unbound) |
| --- | --- | --- |
| **Résolveur** | Quad9 (tiers, Suisse) | Local (Raspberry Pi) |
| **Intermédiaire** | Oui — requêtes envoyées à Quad9 | Non — résolution directe depuis la racine |
| **Censure externe** | Possible (injonctions ARCOM) | Impossible (aucun tiers dans la chaîne) |
| **DNSSEC** | Validé par Quad9 | Validé localement par Unbound |
| **Latence cache** | ~20ms (réseau) | ~0ms (localhost) |
| **Latence miss** | ~50ms (DoH vers Quad9) | ~50-200ms (résolution récursive complète) |
| **SPOF externe** | Quad9 down = plus de DNS | Aucun — les 13 serveurs racine sont redondants |

---

## 🚀 Installation

### Pré-requis

* Raspberry Pi avec AdGuard Home installé et fonctionnel ([cf. fiche LAB AdGuard](./Homelab_AdGuard.md))
* Accès SSH au Pi
* AdGuard Home en écoute sur le port 53

### 1. Installer Unbound

```bash
sudo apt update && sudo apt install unbound -y
```

> ⚠️ Unbound peut échouer au premier démarrage car le port 53 est déjà occupé par AdGuard Home. C'est normal, il sera configuré sur le port 5335.

### 2. Télécharger les Root Hints

Le fichier **root hints** contient les adresses IP des 13 serveurs racine DNS mondiaux (a.root-servers.net à m.root-servers.net). Unbound en a besoin comme point de départ pour résoudre n'importe quel nom de domaine.

```bash
sudo wget -O /var/lib/unbound/root.hints https://www.internic.net/domain/named.root
```

### 3. Créer la configuration

Créer le fichier `/etc/unbound/unbound.conf.d/pi-unbound.conf` :

```bash
sudo nano /etc/unbound/unbound.conf.d/pi-unbound.conf
```

Contenu :

```yaml
server:
    # === INTERFACE ===
    # Écoute locale uniquement, port 5335 pour ne pas entrer en conflit avec AdGuard (port 53)
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    do-ip6: no

    # === SÉCURITÉ ===
    # Empêche les enregistrements "glue" hors de la zone autoritaire
    harden-glue: yes
    # Exige la validation DNSSEC — refuse les réponses si la chaîne de confiance est cassée
    harden-dnssec-stripped: yes
    # Protège contre les réponses DNS qui redirigent vers des serveurs non légitimes
    harden-referral-path: yes
    # Randomise la casse des noms de domaine (protection anti-spoofing, RFC draft 0x20)
    use-caps-for-id: yes
    # Durée minimale de cache — évite les TTL trop courts utilisés par certains trackers
    cache-min-ttl: 3600
    # Durée maximale de cache (1 jour)
    cache-max-ttl: 86400

    # === PRIVACY ===
    # QNAME minimisation (RFC 9156) : n'envoie que le strict nécessaire aux serveurs autoritaires
    # Au lieu de demander "www.example.com" à chaque niveau, demande juste "example.com" au root
    qname-minimisation: yes
    # Utilise les preuves NSEC de manière agressive pour prouver qu'un domaine n'existe pas
    # sans recontacter le serveur autoritaire (réduit le trafic DNS sortant)
    aggressive-nsec: yes

    # === PERFORMANCE ===
    # Prefetch : renouvelle le cache avant expiration pour les domaines fréquemment demandés
    prefetch: yes
    # 1 thread suffit largement pour un Pi 3B en usage domestique
    num-threads: 1
    # Taille du cache messages et enregistrements
    msg-cache-size: 50m
    rrset-cache-size: 100m

    # === ROOT HINTS ===
    root-hints: "/var/lib/unbound/root.hints"

    # === ANTI DNS REBINDING ===
    # Empêche un serveur DNS externe de répondre avec une IP privée (attaque DNS rebinding)
    private-address: 192.168.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
```

### 4. Résoudre le problème DNSSEC (trust anchor)

> ⚠️ **Problème rencontré :** au premier lancement, Unbound retourne `SERVFAIL` sur toutes les requêtes à cause d'un **trust anchor DNSSEC** invalide ou absent.

**Diagnostic :** la requête échoue normalement mais fonctionne avec le flag `+cd` (DNSSEC check disabled) :

```bash
# Échec avec validation DNSSEC
dig @127.0.0.1 -p 5335 google.com        # → status: SERVFAIL

# OK sans validation DNSSEC → le problème vient du trust anchor
dig @127.0.0.1 -p 5335 +cd google.com    # → status: NOERROR ✅
```

> 💡 Le binaire `unbound-anchor` n'est pas disponible sur cette version Debian. On régénère manuellement le trust anchor depuis le fichier système.

**Correction :**

```bash
# Copier le trust anchor système vers le répertoire Unbound
sudo cp /usr/share/dns/root.key /var/lib/unbound/root.key

# Donner les bons droits au user unbound
sudo chown unbound:unbound /var/lib/unbound/root.key

# Redémarrer le service
sudo systemctl restart unbound
```

### 5. Démarrage et validation

```bash
# Vérifier la syntaxe de la configuration
sudo unbound-checkconf

# Vérifier le statut du service
sudo systemctl status unbound
```

**Warnings attendus (sans impact) :**

* `DAEMON_OPTS evaluates to an empty string` → variable d'environnement vide dans le fichier unit systemd, cosmétique
* `subnetcache: prefetch is set but not working` → module subnet cache chargé par défaut mais non utilisé, le prefetch fonctionne normalement sur le cache principal

**Test de résolution :**

```bash
# Première requête : résolution complète depuis la racine (~50-200ms)
dig @127.0.0.1 -p 5335 google.com
# → status: NOERROR ✅

# Deuxième requête : servie depuis le cache (~0ms)
dig @127.0.0.1 -p 5335 google.com
# → Query time: 0 msec ✅
```

![dig](/images/2026-04-01-23-08-53.png)

`NOERROR`, résolution parfaite. Unbound est **opérationnel**.

### 6. Bascule AdGuard Home

Dans l'interface AdGuard Home → **Paramètres** → **Paramètres DNS** :

**Serveurs DNS en amont :**

```sh
127.0.0.1:5335
[/bytel.fr/]192.168.1.254
[/lan/]192.168.1.254
```

* `127.0.0.1:5335` → toutes les requêtes passent par Unbound
* Lignes conditionnelles → les noms locaux de la box Bouygues sont renvoyés vers la box (seule à les connaître)

**Serveurs DNS de repli :** vider entièrement. Si un repli vers Quad9/Cloudflare est configuré, AdGuard basculera dessus dès qu'Unbound met un peu de temps à répondre → retour à des DNS soumis aux blocages ARCOM, ce qui annule l'intérêt d'Unbound.

**Serveurs DNS d'amorçage :** vider entièrement. L'amorçage sert à résoudre les hostnames DoH/DoT des upstreams (ex: `dns.quad9.net`). Or `127.0.0.1:5335` est une IP, rien à résoudre.

**DNS privés inverses :** non nécessaire. AdGuard Home gère le DHCP et connaît déjà les associations IP ↔ hostname via sa table de baux.

**Validation :** cliquer sur **Tester les serveurs en amont** → doit passer au vert, puis **Appliquer**.

---

## 🔄 Maintenance

### Mise à jour automatique des Root Hints

Les adresses des serveurs racine changent rarement (quelques fois par décennie) mais une mise à jour mensuelle est une bonne pratique de sécurité. Ajout d'une tâche cron via `sudo crontab -e` :

```cron
# Mise à jour mensuelle des root hints (1er du mois à 3h du matin)
0 3 1 * * wget -qO /var/lib/unbound/root.hints https://www.internic.net/domain/named.root && systemctl restart unbound
```

* `-q` (quiet) → pas de sortie console, évite de polluer les mails cron
* `&&` → Unbound n'est redémarré que si le téléchargement a réussi

Vérification : `sudo crontab -l`

### Script de Sauvegarde (PowerShell)

Le script de sauvegarde de la config AdGuard a été mis à jour pour inclure la configuration Unbound. Il utilise une boucle sur un tableau de fichiers, facilitant l'ajout de futures configs à sauvegarder.

**Pré-requis :** clé SSH ed25519 dédiée déjà déployée sur le Pi.

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

![script](/images/2026-04-01-23-29-37.png)

---

## 📋 Résumé

* **Service :** Unbound 1.22.0 — résolveur DNS récursif
* **Hébergement :** Raspberry Pi 3B (colocation avec AdGuard Home)
* **Port :** 5335/UDP+TCP (localhost uniquement)
* **Chaîne DNS :** Appareils → AdGuard Home :53 → Unbound :5335 → Serveurs racine
* **Sécurité :** DNSSEC (validation locale), QNAME minimisation, anti-rebinding, 0x20 randomization
* **Privacy :** Aucun DNS tiers dans la chaîne, aucune requête envoyée à Google/Cloudflare/Quad9
* **Maintenance :** Cron mensuel (root hints) + script de sauvegarde PowerShell
* **Supervision :** Via AdGuard Home (stats requêtes) + `sudo systemctl status unbound`

---

## 📚 Références

* Unbound (NLnet Labs) : <https://github.com/NLnetLabs/unbound>
* Root Hints (IANA/InterNIC) : <https://www.internic.net/domain/named.root>
* Root Servers : <https://root-servers.org/>
* QNAME Minimisation (RFC 9156) : <https://www.rfc-editor.org/rfc/rfc9156>
* DNSSEC : <https://www.icann.org/resources/pages/dnssec-what-is-it-why-important-2019-03-05-en>
* Quad9 — réponse aux injonctions françaises : <https://quad9.net/news/press/quad9-faces-new-dns-censorship-legal-challenge-in-france-from-canal/>
