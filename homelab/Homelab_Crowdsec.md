# 🛡️ LAB : Déploiement CrowdSec — Sécurité Collaborative sur pfSense

```txt
   ▄▄▄                           █   ▄▄▄▄               
 ▄▀   ▀  ▄ ▄▄   ▄▄▄  ▄     ▄  ▄▄▄█  █▀   ▀  ▄▄▄    ▄▄▄  
 █       █▀  ▀ █▀ ▀█ ▀▄ ▄ ▄▀ █▀ ▀█  ▀█▄▄▄  █▀  █  █▀  ▀ 
 █       █     █   █  █▄█▄█  █   █      ▀█ █▀▀▀▀  █     
  ▀▄▄▄▀  █     ▀█▄█▀   █ █   ▀█▄██  ▀▄▄▄█▀ ▀█▄▄▀  ▀█▄▄▀ 
```

![Type](https://img.shields.io/badge/Environnement-Machine%20Virtuelle-005085?style=flat-square&logo=virtualbox&logoColor=white)
![OS](https://img.shields.io/badge/OS-pfSense%20CE%202.8.1-A81D33?style=flat-square&logo=pfsense&logoColor=white)
![Service](https://img.shields.io/badge/Service-CrowdSec%201.7.6-00B32C?style=flat-square)
![Role](https://img.shields.io/badge/Rôle-IDS%20%2F%20IPS%20Collaboratif-FF6B35?style=flat-square)
![Network](https://img.shields.io/badge/Réseau-LAN-purple?style=flat-square&logo=web&logoColor=white)

**Rôle :** Administrateur d'Infrastructures Sécurisées

**Mission :** Déployer CrowdSec, un moteur de sécurité open-source et collaboratif, directement sur le pare-feu pfSense. CrowdSec analyse les logs du système (firewall, SSH, interface web) pour détecter les comportements malveillants (scan de ports, brute-force, probing HTTP) et appliquer des décisions de remédiation (ban d'IP) au niveau du **Packet Filter**. Sa force réside dans son modèle communautaire : les IP malveillantes détectées par l'ensemble des utilisateurs CrowdSec sont partagées via une API centrale (CAPI), alimentant des **blocklists collaboratives** appliquées automatiquement sur l'infrastructure.

> - [Documentation Officielle CrowdSec](https://docs.crowdsec.net/)
> - [GitHub pfSense Package](https://github.com/crowdsecurity/pfSense-pkg-crowdsec)
> - [Console CrowdSec](https://app.crowdsec.net)

---

## L'intérêt technique 🎯

1. **Détection Comportementale (Log-Driven) :** Contrairement à un IDS réseau classique (Suricata) qui inspecte les paquets en transit, CrowdSec analyse les **logs applicatifs** pour détecter des patterns d'attaque. L'approche est complémentaire : Suricata voit les payloads, CrowdSec voit les comportements (tentatives répétées, scans distribués).
2. **Sécurité Collaborative (CTI Communautaire) :** Chaque instance CrowdSec partage les IP attaquantes détectées avec la communauté. En retour, l'infrastructure bénéficie de blocklists enrichies par des milliers d'autres utilisateurs — un modèle de **Threat Intelligence participative**.
3. **Remédiation au Packet Filter :** Le bouncer `pfsense-firewall` injecte directement les IP bannies dans les tables du pare-feu pfSense (`crowdsec_blacklists` / `crowdsec6_blacklists`), bloquant le trafic malveillant à la couche réseau avant qu'il n'atteigne les services.
4. **Architecture Flexible (Small / Medium / Large) :** CrowdSec peut fonctionner en mode autonome sur pfSense ou être distribué sur plusieurs machines avec un LAPI centralisé, permettant une montée en charge progressive de la sécurité.

---

## 🛠️ Architecture du Lab

- **Environnement :** Serveur Proxmox VE (HP ProDesk 600 G4)
- **Plateforme :** VM pfSense CE 2.8.1 (FreeBSD 15.0-CURRENT)
- **IP pfSense :** `192.168.1.251` (interface WAN sur `vmbr0`)
- **Mode d'installation :** **Large** (Security Engine + Log Processor + LAPI + Bouncer — autonome)
- **Stockage :** ZFS persistant (pas de RAM disk sur `/var`)

### Composants CrowdSec installés

| Composant | Rôle | Port/Fichier |
| --- | --- | --- |
| **Security Engine** (`crowdsec`) | Analyse les logs, évalue les scénarios, prend les décisions | `/var/log/crowdsec/` |
| **Log Processor** | Parse les fichiers de logs et alimente les buckets de détection | `filter.log`, `auth.log`, `system.log`, `nginx.log` |
| **Local API (LAPI)** | API centrale locale, stocke les décisions et sert les bouncers | TCP `8080` (localhost) |
| **Firewall Bouncer** (`pfsense-firewall`) | Applique les bans dans les tables PF de pfSense | `crowdsec_blacklists` |

### Scénarios de détection actifs par défaut

- `crowdsecurity/ssh-bf` — Brute-force SSH
- `crowdsecurity/pfsense-gui-bf` — Brute-force interface web pfSense
- `crowdsecurity/http-probing` — Probing de vulnérabilités HTTP
- `crowdsecurity/portscan` — Scan de ports

---

> Documentation pfSense : <https://docs.crowdsec.net/docs/getting_started/install_crowdsec_pfsense/>

---

## Prérequis

Avant l'installation, vérifier les points suivants sur la VM pfSense :

**1. Version FreeBSD :**

```csh
freebsd-version
# Attendu : 14.x (pfSense CE 2.7.x) ou 15.x (pfSense CE 2.8.x)
```

**2. Stockage persistant pour `/var` :**

Le LAPI CrowdSec stocke sa base de données et les tables GeoIP dans `/var/db`. Il est impératif que `/var` ne soit **pas** monté en RAM disk (tmpfs).

```csh
mount | grep /var
# ✅ OK si on voit "zfs" ou "ufs" pour /var, /var/db, /var/log
# ❌ KO si on voit "tmpfs" pour /var/db (désactiver le RAM disk dans System > Advanced > Miscellaneous)
# ℹ️ tmpfs sur /var/run est normal et sans impact
```

**3. Accès SSH :**

SSH est désactivé par défaut sur pfSense. L'activer depuis l'interface web :

➡️ **System > Advanced > Admin Access** → cocher **Enable Secure Shell** → **Save**

```bash
ssh admin@192.168.1.251
# Au menu pfSense, taper 8 pour accéder au Shell
```

---

## 1️⃣ Installation du Paquet CrowdSec

Depuis le shell pfSense (option 8), exécuter le script d'installation officiel :

```csh
fetch https://raw.githubusercontent.com/crowdsecurity/pfSense-pkg-crowdsec/refs/heads/main/install-crowdsec.sh
sh install-crowdsec.sh
```

Le script télécharge et installe automatiquement les dépendances dans l'ordre :

1. `abseil` (bibliothèque C++)
2. `re2` (moteur de regex)
3. `crowdsec-firewall-bouncer` (composant de remédiation)
4. `crowdsec` (Security Engine)
5. `pfSense-pkg-crowdsec` (intégration interface web)

Chaque paquet doit afficher `Extracting... done`. Confirmer avec `y` si un prompt de confirmation apparaît.

> ⚠️ **Ne pas lancer les services manuellement** (`service crowdsec start` etc.) — pfSense gère le cycle de vie des services après la configuration via l'interface web.
>
> 💡 En cas d'erreur de compatibilité FreeBSD, relancer avec une version spécifique :
>
> ```csh
> sh install-crowdsec.sh --release v0.1.6-1.7.6-34
> ```

![install](/images/2026-03-25-01-55-39.png)

---

## 2️⃣ Configuration via l'Interface Web

➡️ Ouvrir l'interface pfSense : `https://192.168.1.251`

➡️ Aller dans **Services > CrowdSec**

➡️ Vérifier que les trois composants sont activés :

| Option | État | Commentaire |
| --- | --- | --- |
| **Remediation Component** | ✅ Activé | Applique les bans dans le Packet Filter |
| **Log Processor** | ✅ Activé | Analyse les logs pfSense |
| **Local API** | ✅ Activé | Mode Large — LAPI autonome sur pfSense |

➡️ Cliquer **Save**

C'est la configuration par défaut (mode **Large**). Les services démarrent automatiquement.

![config](/images/2026-03-25-01-57-54.png)

---

## 3️⃣ Ajout de l'Acquisition SSH (`system.log`)

Par défaut, CrowdSec sur pfSense n'acquiert que `filter.log` et `nginx.log`. Les logs SSH sur FreeBSD/pfSense sont écrits dans `/var/log/system.log` et `/var/log/auth.log` — il faut les ajouter manuellement.

```csh
cat > /usr/local/etc/crowdsec/acquis.d/ssh.yaml << 'EOF'
filenames:
  - /var/log/system.log
  - /var/log/auth.log
force_inotify: true
poll_without_inotify: true
labels:
  type: syslog
EOF
```

Redémarrer le service pour prendre en compte la nouvelle source :

```csh
service crowdsec.sh restart
```

---

## 4️⃣ Vérification des Services

### Interface web

➡️ **Status > Services** — Les deux services doivent apparaître en vert :

- `crowdsec` (Security Engine)
- `crowdsec-firewall-bouncer` (Remediation)

![service](/images/2026-03-25-01-58-33.png)

### Ligne de commande

> 💡 Sur pfSense (csh), le binaire `cscli` n'est pas dans le `PATH` par défaut. Utiliser le chemin complet `/usr/local/bin/cscli`.

```csh
# Machines enregistrées (le log processor local)
/usr/local/bin/cscli machines list

# Bouncers enregistrés (le firewall bouncer)
/usr/local/bin/cscli bouncers list

# Collections installées (scénarios de détection)
/usr/local/bin/cscli collections list

# Métriques d'acquisition et de parsing
/usr/local/bin/cscli metrics
```

La commande `cscli metrics` doit montrer l'acquisition de :

- `file:/var/log/filter.log` — Logs du pare-feu (règles PF)
- `file:/var/log/auth.log` — Logs d'authentification
- `file:/var/log/system.log` — Logs système (SSH)
- `file:/var/log/nginx.log` — Logs de l'interface web

![collections](/images/2026-03-25-02-07-30.png)

### Vérification des blocklists actives

```csh
# Décisions actives (IP bannies — locales + communautaires)
/usr/local/bin/cscli decisions list -a

# Tables PF du Packet Filter
pfctl -T show -t crowdsec_blacklists      # IPv4
pfctl -T show -t crowdsec6_blacklists     # IPv6
```

---

## 5️⃣ Inscription à la Console CrowdSec (Optionnel — Recommandé)

La [Console CrowdSec](https://app.crowdsec.net) offre un dashboard centralisé gratuit pour visualiser les alertes, les décisions, et gérer les blocklists communautaires.

**1. Créer un compte** sur [app.crowdsec.net](https://app.crowdsec.net)

**2. Récupérer la clé d'enrollment :**

➡️ Menu gauche → **Security Engines** → **Engines** → la clé est affichée en bas de page (bouton **Add Security Engine**). Format : `clxxxxxxxxxxxxxxxxxx`.

**3. Enregistrer l'instance pfSense :**

```csh
/usr/local/bin/cscli console enroll --name "pfSense-Homelab" VOTRE_CLE_ICI
```

**4. Accepter l'enrollment** dans la console web → le moteur apparaît avec un bouton **"Accept enroll"**.

**5. Redémarrer le service :**

```csh
service crowdsec.sh restart
```

![enroll](/images/2026-03-25-02-08-51.png)

![portailweb](/images/2026-03-25-03-16-37.png)

---

## 6️⃣ Test de Fonctionnement

### Test manuel (bouncer uniquement)

Injecter une décision de ban sur une IP fictive pour vérifier la chaîne de remédiation :

```csh
# Bannir une IP fictive pendant 5 minutes
/usr/local/bin/cscli decisions add -t ban -d 5m -i 203.0.113.42

# Vérifier la décision
/usr/local/bin/cscli decisions list

# Vérifier la table PF
pfctl -T show -t crowdsec_blacklists

# Supprimer la décision
/usr/local/bin/cscli decisions delete -i 203.0.113.42
```

### Test de détection (scan de ports depuis Kali)

Depuis la VM Kali en DMZ (`10.0.0.30`) :

```bash
nmap -sS -T4 192.168.1.251
```

Puis vérifier sur pfSense :

```csh
/usr/local/bin/cscli alerts list
/usr/local/bin/cscli metrics
```

> ⚠️ **Comportement attendu :** Les IP privées (RFC 1918) sont **whitelistées par défaut** depuis CrowdSec 1.6.3. L'alerte est générée mais l'IP n'est pas bannie. Cela évite de se bloquer soi-même sur le réseau local. Les IP publiques malveillantes sont bannies normalement.
>
> ℹ️ **Note :** pfSense intègre nativement **sshguard** (Login Protection) qui réagit plus vite que CrowdSec sur les brute-force SSH locaux. Les deux systèmes fonctionnent en parallèle — sshguard pour la réactivité locale, CrowdSec pour la dimension communautaire et la gestion centralisée.

---

## ⚙️ Commandes Utiles (Aide-Mémoire)

```csh
# --- Gestion des services ---
service crowdsec.sh start|stop|restart
service crowdsec_firewall.sh start|stop|restart

# --- Consultation ---
/usr/local/bin/cscli alerts list                    # Alertes actives
/usr/local/bin/cscli decisions list -a              # Toutes les décisions (locales + CAPI)
/usr/local/bin/cscli metrics                        # Métriques d'acquisition et parsing
/usr/local/bin/cscli machines list                  # Log processors enregistrés
/usr/local/bin/cscli bouncers list                  # Bouncers enregistrés
/usr/local/bin/cscli collections list               # Scénarios installés

# --- Actions manuelles ---
/usr/local/bin/cscli decisions add -t ban -d 4h -i <IP>     # Bannir une IP
/usr/local/bin/cscli decisions delete -i <IP>                # Débannir une IP

# --- Tables Packet Filter ---
pfctl -T show -t crowdsec_blacklists                # IPs IPv4 bannies
pfctl -T show -t crowdsec6_blacklists               # IPs IPv6 bannies

# --- Hub (ajout de scénarios) ---
/usr/local/bin/cscli collections install <nom>      # Installer une collection
/usr/local/bin/cscli hub update                     # Mettre à jour le hub
```

---

## 🔐 Points de Sécurité

- **Défense en profondeur :** CrowdSec s'ajoute comme couche de sécurité supplémentaire, en complément du filtrage par règles de pfSense (couche 3/4) et de sshguard (protection anti brute-force native). C'est l'application du principe de **défense en profondeur** recommandé par l'ANSSI.
- **Whitelisting des réseaux privés :** Par défaut, les IP privées ne sont pas bannies par les décisions locales. Ce comportement est géré par le parseur `crowdsecurity/whitelists`. Pour le modifier (non recommandé en homelab) : `/usr/local/bin/cscli parsers remove crowdsecurity/whitelists`.
- **Sauvegarde :** La configuration CrowdSec **n'est pas incluse** dans les sauvegardes pfSense standard. Penser à sauvegarder séparément `/usr/local/etc/crowdsec/` et `/var/db/crowdsec/`.
- **Mises à jour :** Les collections et parseurs du Hub sont mis à jour automatiquement via un cron job quotidien. Les mises à jour majeures de pfSense peuvent nécessiter une réinstallation du paquet CrowdSec.

---

## 🎯 Validation de l'Installation

- **Acquisition des logs :** Les fichiers `filter.log`, `auth.log`, `system.log` et `nginx.log` apparaissent dans `cscli metrics` avec des lignes parsées.
- **Blocklists CAPI :** `cscli decisions list -a` affiche plusieurs milliers d'IP bannies (origine `CAPI`) pour des raisons `ssh:bruteforce`, `generic:scan`, etc.
- **Bouncer actif :** `cscli bouncers list` montre `pfsense-firewall` avec un statut valide.
- **Tables PF alimentées :** `pfctl -T show -t crowdsec_blacklists` retourne des IP.
- **Console (si enrollé) :** L'instance apparaît sur [app.crowdsec.net](https://app.crowdsec.net) avec les métriques remontées.

---

## 📐 Architecture CrowdSec dans l'Infrastructure Homelab

```txt
                        ┌──────────────────────────────────────────┐
                        │          CAPI (CrowdSec Cloud)           │
                        │   Blocklists communautaires partagées    │
                        └──────────────────┬───────────────────────┘
                                           │ HTTPS (pull/push)
                                           │
┌──────────────────────────────────────────────────────────────────────────────┐
│  pfSense VM (192.168.1.251) — Mode Large                                     │
│                                                                              │
│  ┌─────────────────┐    ┌──────────────────┐    ┌────────────────────────┐   │
│  │  Log Processor  │──▶│  Security Engine  │──▶│  Firewall Bouncer      │   │
│  │                 │    │     (LAPI)       │    │  (pfsense-firewall)    │   │
│  │ filter.log      │    │                  │    │                        │   │
│  │ auth.log        │    │  Scénarios :     │    │  ┌──────────────────┐  │   │
│  │ system.log      │    │  · ssh-bf        │    │  │ Packet Filter    │  │   │
│  │ nginx.log       │    │  · portscan      │    │  │ crowdsec_        │  │   │
│  │                 │    │  · gui-bf        │    │  │  blacklists      │  │   │
│  │                 │    │  · http-probing  │    │  └──────────────────┘  │   │
│  └─────────────────┘    └──────────────────┘    └────────────────────────┘   │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│  WAN (vmbr0) : 192.168.1.251          │  LAN (vmbr2) : 10.0.0.1              │
│  ← Trafic entrant analysé             │  → Services protégés (DMZ)           │
└──────────────────────────────────────────────────────────────────────────────┘
```

![crowdsec](/images/2026-02-11-22-23-17.png)
