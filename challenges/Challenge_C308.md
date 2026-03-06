# Challenge C308 05/03/2026

## 🧑‍🏫 Pitch de l’exercice : Architecture Zero Trust : Routage d'Identité (SSO) & Sécurité de la Couche 7 (WAF) 🛡️

[Cours C308.](/RESUME.md#️-c308-sso-iam--waf-identités--sécurité-web)

---

**Pitch :** Unifier et sécuriser les accès de l'infrastructure en déployant un portail d'authentification centralisé (Identity Provider), et intégrer un pare-feu applicatif (WAF) pour analyser la charge utile des trames HTTP afin de bloquer les attaques complexes à la volée.

### 🎯 Contexte d'introduction

En tant qu'architecte réseau et sécurité, vous constatez que l'infrastructure héberge des services de plus en plus critiques, tels qu'un coffre-fort de secrets (Vault) et un espace de stockage (Nextcloud).

Historiquement, chaque application gère sa propre base de données d'utilisateurs. Cela multiplie les flux d'authentification sur le réseau, crée des failles de sécurité potentielles et rend l'audit des accès impossible. De plus, les pare-feux classiques de couche 3 et 4 (iptables, limites TCP) que nous avons déployés en périphérie sont totalement aveugles face aux attaques cachées à l'intérieur même du trafic web légitime (comme les injections SQL ou le Cross-Site Scripting).

### 🚀 Objectifs de l'intervention

Pour verrouiller ce périmètre applicatif, nous allons agir sur deux axes majeurs de la couche 7 :

1. **La centralisation des flux d'accès (SSO avec Keycloak) :** Nous allons déployer Keycloak pour agir comme notre unique "routeur d'identité". Désormais, Vault et Nextcloud ne valideront plus aucun mot de passe. Ils redirigeront les flux d'authentification vers Keycloak via des protocoles standardisés (OIDC/SAML). Si l'accès est validé, Keycloak renvoie un jeton cryptographique (Token) au service. Le réseau ne transporte plus de multiples mots de passe, mais uniquement des jetons d'accès sécurisés.

2. **L'inspection applicative profonde (WAF avec ModSecurity + OWASP) :**
Nous allons greffer un module d'inspection (ModSecurity) sur notre point d'entrée. Contrairement à un pare-feu classique qui lit les en-têtes IP/TCP, le WAF va ouvrir la charge utile (le *payload*) de chaque paquet HTTP. En s'appuyant sur les règles mondiales de l'OWASP (Core Rule Set), il va analyser le comportement des requêtes en temps réel pour "Drop" celles contenant du code malveillant, avant même qu'elles n'atteignent nos serveurs web internes.

---

[Déploiement d'un SSO Keycloak et intégration avec Vault](#-déploiement-dun-sso-keycloak-et-intégration-avec-vault)

[Intégration d'une deuxième application (Nextcloud) dans le SSO Keycloak](#-intégration-dune-deuxième-application-nextcloud-dans-le-sso-keycloak)

[Intégration d'un module d'inspection (ModSecurity) sur notre point d'entrée](#-intégration-dun-module-dinspection-modsecurity-sur-notre-point-dentrée)

---

### 🖥️ Environnement technique

```txt
        [Client navigateur]
         Lubuntu — 10.0.0.50
                │
   ┌────────────┴─────────────────────────────────────────┐
   │           vmbr2 — LAN A (10.0.0.0/24)                │
   └──┬──────────────┬──────────────┬──────────────┬──────┘
      │              │              │              │
  10.0.0.60      10.0.0.62     10.0.0.63      10.0.0.64
      │              │              │              │
  [keycloak]      [vault]       [web-app]     [nextcloud]
  Keycloak 26    Vault 1.x     (optionnel)   Nextcloud 30
  port 8080      port 8200                   port 80
```

| Machine | Type | IP | Rôle |
| --------- | ------ | ---- | ------ |
| Keycloak | LXC | 10.0.0.60 | Identity Provider |
| Vault | LXC | 10.0.0.62 | HashiCorp Vault |
| Nextcloud | LXC | 10.0.0.64 | Nextcloud |
| Lubuntu | VM | 10.0.0.50 | Client navigateur pour les tests |

---

## 🔐 Déploiement d'un SSO Keycloak et intégration avec Vault

## Étape 1 — Installer Keycloak

### Prérequis : Java 17

Keycloak nécessite OpenJDK 17 pour fonctionner.

Sur le conteneur **keycloak** (10.0.0.60) :

```bash
apt update && apt install -y openjdk-21-jdk
java -version
# → openjdk version "21.x.x"
```

### Télécharger et installer Keycloak

```bash
# Télécharger l'archive Keycloak 26.5.4
wget https://github.com/keycloak/keycloak/releases/download/26.5.4/keycloak-26.5.4.tar.gz

# Extraire l'archive
tar -xvzf keycloak-26.5.4.tar.gz

# Déplacer dans /opt
mv keycloak-26.5.4 /opt/keycloak
```

### Créer le compte administrateur initial

```bash
# Créer un utilisateur admin (le mot de passe sera demandé interactivement)
/opt/keycloak/bin/kc.sh build
/opt/keycloak/bin/kc.sh bootstrap-admin user --optimized
```

> ⚠️ À ne faire qu'une seule fois, avant le premier démarrage.
> Notez le login/mot de passe saisi — c'est le compte **admin global**.

### Démarrer Keycloak (mode développement)

```bash
# Démarrage en mode dev (HTTP, pas de cert TLS requis)
/opt/keycloak/bin/kc.sh start-dev --http-port=8080
```

| Option | Signification |
|--------|---------------|
| `start-dev` | Mode développement (HTTP, logs verbeux) |
| `--http-port=8080` | Port d'écoute (défaut : 8080) |

> Pour la **production**, utiliser `start` avec TLS configuré.

### Démarrer Keycloak en arrière-plan

```bash
# Lancer en arrière-plan et rediriger les logs
nohup /opt/keycloak/bin/kc.sh start-dev --http-port=8080 \
    > /var/log/keycloak.log 2>&1 &

# Vérifier que le processus tourne
ps aux | grep keycloak

# Suivre les logs de démarrage (attendre "Keycloak is running")
tail -f /var/log/keycloak.log
```

### Vérifier que Keycloak répond

```bash
apt install -y curl
curl -s http://10.0.0.60:8080/realms/master | python3 -m json.tool | head -5
# → {"realm": "master", "public_key": "...", ...}
```

→ ✅ Keycloak est démarré et accessible

![OK](/images/2026-03-05-18-51-09.png)

## Étape 2 — Configurer Keycloak : Realm et Client

### Accéder à la console d'administration

Depuis le client, ouvrir un navigateur :

<http://10.0.0.60:8080/admin>

Se connecter avec le compte admin créé à l'étape précédente.

![log](/images/2026-03-05-18-53-09.png)

### Créer un nouveau Realm

Un **Realm** est un espace d'identité isolé (équivalent d'un tenant).

1. Cliquer sur le menu déroulant **"master"** en haut à gauche
2. Cliquer sur **"Create realm"**
3. Renseigner :

| Champ | Valeur |
|-------|--------|
| Realm name | `demo` |
| Enabled | ON |

4. Cliquer sur **"Create"**

→ Vous êtes maintenant dans le realm `demo`

### Créer un Client OIDC (pour Vault)

Les **Clients** sont les applications qui délèguent l'authentification à Keycloak.

Dans le realm `demo` :

1. **Clients** → **Create client**
2. Onglet **"General Settings"** :

| Champ | Valeur |
|-------|--------|
| Client type | OpenID Connect |
| Client ID | `vault` |
| Name | `vault` |

3. Cliquer **"Next"**

### Paramètres du flux d'authentification

Onglet **"Capability config"** :

| Option | Valeur |
|--------|--------|
| Client authentication | ON (confidential) |
| Standard flow | ON |
| Direct access grants | ON (optionnel, pour tests) |

Cliquer **"Next"**

### URLs de redirection

Onglet **"Login settings"** :

| Champ | Valeur |
|-------|--------|
| Root URL | `http://10.0.0.62:8200` |
| Valid redirect URIs | `http://10.0.0.62:8200/ui/vault/auth/oidc/oidc/callback` |
| Valid redirect URIs | `http://10.0.0.62:8200/oidc/callback` |
| Web origins | `http://10.0.0.62:8200` |

Cliquer **"Save"**

--

### Récupérer le Client Secret

1. Aller dans **Clients** → `vault` → onglet **"Credentials"**
2. Copier la valeur du **Client secret**

```sh
# Exemple :
Client ID     : vault
Client Secret : IGqNuGf9AfyqkEfvcX07wUSDSnjvlqZw
```

> 💡 Ce secret sera utilisé pour configurer l'auth method OIDC dans Vault.

### Créer un Mapper pour les groupes (claims JWT)

Pour transmettre les groupes Keycloak dans le token JWT :

1. **Clients** → `vault` → onglet **"Client scopes"**
2. Cliquer sur `vault-dedicated`
3. **"Add mapper"** → **"By configuration"** → **"Group Membership"**
4. Configurer :

| Champ | Valeur |
|-------|--------|
| Name | `groups` |
| Token Claim Name | `groups` |
| Full group path | OFF |
| Add to ID token | ON |
| Add to access token | ON |

Cliquer **"Save"**

![realm](/images/2026-03-05-19-01-24.png)

---

## Étape 3 — Créer des utilisateurs et groupes dans Keycloak

### Créer un groupe

Dans le realm `demo` :

1. **Groups** → **"Create group"**
2. Name : `vault-readers`
3. Cliquer **"Create"**

### Créer un utilisateur

1. **Users** → **"Create new user"**
2. Renseigner :

| Champ | Valeur |
|-------|--------|
| Username | `alice` |
| Email | `alice@demo.local` |
| First name | `Alice` |
| Last name | `Demo` |
| Email verified | ON |

3. Cliquer **"Create"**

### Définir le mot de passe de l'utilisateur

1. Aller dans **Users** → `alice` → onglet **"Credentials"**
2. Cliquer **"Set password"**
3. Renseigner :

| Champ | Valeur |
|-------|--------|
| Password | `Alice123!` |
| Temporary | OFF |

4. Cliquer **"Save"** puis **"Save password"**

### Ajouter l'utilisateur au groupe

1. Aller dans **Users** → `alice` → onglet **"Groups"**
2. Cliquer **"Join Group"**
3. Sélectionner `vault-readers`
4. Cliquer **"Join"**

→ ✅ Alice appartient au groupe `vault-readers`

![alice](/images/2026-03-05-19-05-08.png)

### Vérifier le token OIDC (optionnel)

On peut tester la génération d'un token directement via l'API :

```bash
curl -s -X POST \
  "http://10.0.0.60:8080/realms/demo/protocol/openid-connect/token" \
  -d "client_id=vault" \
  -d "client_secret=IGqNuGf9AfyqkEfvcX07wUSDSnjvlqZw" \
  -d "username=alice" \
  -d "password=Alice123!" \
  -d "grant_type=password" | python3 -m json.tool
```

→ Repérer le champ `access_token` et le décoder sur [jwt.io](https://jwt.io)

→ Vérifier que le claim `groups` contient `vault-readers`

![jwt](/images/2026-03-05-19-09-03.png)

---

## Étape 4 — Installer HashiCorp Vault

### Ajouter le dépôt officiel HashiCorp

Sur le conteneur **vault** (10.0.0.62) :

```bash
# Importer la clé GPG HashiCorp
apt install -y gpg curl
wget -O - https://apt.releases.hashicorp.com/gpg | \
    gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Ajouter le dépôt APT
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
    https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
    tee /etc/apt/sources.list.d/hashicorp.list

# Ajotuer le dépot sur un Container
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com bookworm main" > /etc/apt/sources.list.d/hashicorp.list

# Installer Vault
apt update && apt install -y vault

# Vérifier la version
vault version
# → Vault v1.x.x
```

### Démarrer Vault en mode développement

```bash
# Démarrer Vault sur toutes les interfaces, port 8200
vault server -dev -dev-listen-address="0.0.0.0:8200"
```

> ⚠️ Le mode `-dev` est **uniquement pour le lab** :
>
> - Stockage en mémoire (les données sont perdues à l'arrêt)
> - TLS désactivé
> - Root token affiché dans les logs

![dev](/images/2026-03-05-19-13-56.png)

### Récupérer et exporter le Root Token

Au démarrage, Vault affiche :

```txt
Root Token: hvs.so5iJlRBh1RIxN4LssQxzAz3
```

Arrêter Vault (Ctrl+C) et le relancer en arrière plan

```bash
# Démarrer Vault en arrière plan avec nohup
nohup vault server -dev -dev-listen-address="0.0.0.0:8200"
```

```bash
# Exporter les variables d'environnement
export VAULT_ADDR="http://127.0.0.1:8200"
export VAULT_TOKEN="hvs.so5iJlRBh1RIxN4LssQxzAz3"

# Vérifier que Vault est accessible
vault status
```

| Champ | Valeur attendue |
|-------|----------------|
| Initialized | true |
| Sealed | false |
| Version | 1.x.x |

### Démarrer Vault en arrière-plan (lab)

```bash
# Dans un screen ou tmux
screen -S vault
vault server -dev -dev-listen-address="0.0.0.0:8200"
# Ctrl+A puis D pour détacher le screen
```

Ou avec nohup :

```bash
nohup vault server -dev -dev-listen-address="0.0.0.0:8200" \
    > /var/log/vault.log 2>&1 &

# Le token se trouve dans les logs :
cat /tmp/vault.log | grep "Root Token"
```

![vault](/images/2026-03-05-19-39-59.png)

---

## Étape 5 — Configurer l'authentification OIDC dans Vault

### Activer la méthode d'authentification OIDC

```bash
export VAULT_ADDR="http://127.0.0.1:8200"
export VAULT_TOKEN="hvs.3Utr3fE8P9MihpSAqcOAUc0Y"

vault auth enable oidc
# → Success! Enabled oidc auth method at: oidc/
```

### Configurer l'OIDC avec Keycloak comme provider

```bash
vault write auth/oidc/config \
    oidc_discovery_url="http://10.0.0.60:8080/realms/demo" \
    oidc_client_id="vault" \
    oidc_client_secret="IGqNuGf9AfyqkEfvcX07wUSDSnjvlqZw" \
    default_role="reader"
```

| Paramètre | Signification |
|-----------|---------------|
| `oidc_discovery_url` | URL du realm Keycloak (expose le `.well-known`) |
| `oidc_client_id` | Client ID déclaré dans Keycloak |
| `oidc_client_secret` | Secret récupéré dans Keycloak |
| `default_role` | Rôle Vault appliqué par défaut |

### Vérifier la découverte OIDC

```bash
# Keycloak expose automatiquement ses métadonnées OIDC
curl -s "http://10.0.0.60:8080/realms/demo/.well-known/openid-configuration" \
    | python3 -m json.tool | grep -E "issuer|authorization|token|jwks"
```

→ Vault utilise cette URL pour valider les tokens JWT émis par Keycloak.

### Créer un rôle Vault lié aux groupes Keycloak

```bash
vault write auth/oidc/role/reader \
    bound_audiences="vault" \
    allowed_redirect_uris="http://10.0.0.62:8200/ui/vault/auth/oidc/oidc/callback" \
    allowed_redirect_uris="http://10.0.0.62:8200/oidc/callback" \
    user_claim="sub" \
    groups_claim="groups" \
    token_policies="reader-policy" \
    token_ttl="1h"
```

| Paramètre | Signification |
|-----------|---------------|
| `bound_audiences` | Le token doit être destiné au client `vault` |
| `user_claim` | Champ du JWT utilisé comme identifiant user |
| `groups_claim` | Champ du JWT contenant les groupes |
| `token_policies` | Politique Vault appliquée après login |
| `token_ttl` | Durée de vie du token Vault |

### Créer une politique Vault (reader-policy)

```bash
vault policy write reader-policy - << 'EOF'
# Autoriser la lecture de secrets dans le chemin demo/
path "secret/data/demo/*" {
  capabilities = ["read", "list"]
}

# Autoriser la liste à la racine
path "secret/metadata/demo/*" {
  capabilities = ["list"]
}
EOF
```

---

## Étape 6 — Créer des secrets dans Vault

### Écrire un secret de test

```bash
# Activer le moteur de secrets KV v2
vault secrets enable -path=secret kv-v2

# Créer un secret
vault kv put secret/demo/config \
    db_host="10.0.0.100" \
    db_user="appuser" \
    db_password="SuperSecret42!"

# Vérifier
vault kv get secret/demo/config
```

![secret](/images/2026-03-05-19-51-20.png)

### Mapper le groupe Keycloak → politique Vault

```bash
# Créer un groupe externe dans Vault
vault write identity/group \
    name="vault-readers-group" \
    type="external" \
    policies="reader-policy"

# Récupérer l'ID du groupe
vault read identity/group/name/vault-readers-group
GROUP_ID=$(vault read -field=id identity/group/name/vault-readers-group)

# Récupérer l'accessor de la méthode OIDC
OIDC_ACCESSOR=$(vault auth list -format=json | \
    python3 -c "import sys,json; d=json.load(sys.stdin); \
    print(d['oidc/']['accessor'])")

# Créer l'alias du groupe (lien entre groupe Keycloak et groupe Vault)
vault write identity/group-alias \
    name="vault-readers" \
    mount_accessor="${OIDC_ACCESSOR}" \
    canonical_id="${GROUP_ID}"
```

→ Les membres du groupe `vault-readers` dans Keycloak auront la politique `reader-policy` dans Vault.

---

## Étape 7 — Tester l'authentification OIDC

### Via l'interface Web de Vault

Depuis l'hôte, ouvrir un navigateur :

<http://10.0.0.62:8200/ui>

1. Sélectionner la méthode d'authentification **OIDC**
2. Cliquer sur **"Sign in with OIDC Provider"**
3. Être redirigé vers la page de login Keycloak
4. Se connecter avec `alice` / `Alice123!`
5. Être redirigé vers Vault avec un token valide

![demo](/images/2026-03-05-19-53-57.png)

![vault](/images/2026-03-05-19-58-13.png)

### Via la CLI Vault

```bash
# Depuis la machine vault ou tout client avec vault installé
vault login -method=oidc -path=oidc role=reader
```

→ Un lien s'ouvre dans le navigateur → se connecter avec Keycloak → le token est injecté dans la CLI

### Tester l'accès au secret après login

```bash
# Après authentification OIDC (token stocké dans ~/.vault-token)
vault kv get secret/demo/config
```

→ ✅ Alice peut lire le secret

```bash
# Tenter d'écrire → doit échouer (politique reader uniquement)
vault kv put secret/demo/config db_host="attacker"
# → Error: 1 error occurred: * permission denied
```

→ ✅ L'accès en écriture est bien refusé

### Vérifier les informations du token

```bash
vault token lookup
```

```txt
Key                  Value
---                  -----
display_name         oidc-alice
policies             [default reader-policy]
ttl                  59m50s
meta                 map[role:reader username:alice]
```

→ On voit que le token est lié à l'utilisateur `alice` et à la politique `reader-policy`

---

## Étape 8 — Vérification end-to-end

### Récapitulatif du flux OIDC

```txt
Alice (navigateur)
    │
    ├─1─► Vault UI : "Login with OIDC"
    │
    ├─2─► Keycloak : page de login
    │
    ├─3─► Alice saisit ses credentials
    │
    ├─4─► Keycloak émet un token JWT signé
    │       (claims: sub=alice, groups=[vault-readers])
    │
    ├─5─► Vault valide le JWT (via JWKS de Keycloak)
    │       → mappe le groupe "vault-readers" → policy "reader-policy"
    │
    └─6─► Vault émet un token interne
              → Alice peut lire secret/demo/*
```

--

### Tableau de vérification

| Test | Résultat attendu |
|------|-----------------|
| Démarrage Keycloak | `curl http://10.0.0.60:8080/realms/master` → JSON |
| Démarrage Vault | `vault status` → `Sealed: false` |
| Discovery OIDC | `curl .../well-known/openid-configuration` → JSON |
| Login Alice via UI | Redirection Keycloak → Vault → token valide |
| Lecture secret | `vault kv get secret/demo/config` → données |
| Écriture secret | `vault kv put ...` → `permission denied` |

---

## Dépannage

--

### Keycloak ne démarre pas

```bash
# Vérifier les logs
tail -f /var/log/keycloak.log

# Vérifier que le port 8080 est libre
ss -tlnp | grep 8080

# Vérifier Java
java -version
```

--

### Vault ne valide pas les tokens Keycloak

```bash
# Vérifier que l'URL de découverte est accessible depuis Vault
curl -s "http://10.0.0.60:8080/realms/demo/.well-known/openid-configuration"

# Vérifier la config OIDC de Vault
vault read auth/oidc/config

# Tester manuellement un token JWT
vault write auth/oidc/oidc/auth_url \
    redirect_uri="http://10.0.0.62:8200/oidc/callback" \
    state="test" nonce="test" role="reader"
```

--

### Les groupes ne sont pas transmis dans le JWT

Vérifier que le mapper `groups` est bien configuré dans Keycloak :

1. **Clients** → `vault` → **Client scopes** → `vault-dedicated`
2. Vérifier la présence du mapper **Group Membership** avec `Token Claim Name = groups`
3. Vérifier que `Full group path = OFF` (sinon le claim contient `/vault-readers` au lieu de `vault-readers`)

```bash
# Décoder un token pour vérifier les claims
curl -s -X POST \
  "http://10.0.0.60:8080/realms/demo/protocol/openid-connect/token" \
  -d "client_id=vault&client_secret=...&username=alice&password=Alice123!&grant_type=password" \
  | python3 -c "
import sys,json,base64
d=json.load(sys.stdin)
token=d['access_token']
payload=token.split('.')[1]
payload += '=' * (4 - len(payload) % 4)
print(json.dumps(json.loads(base64.b64decode(payload)), indent=2))
"
```

→ Vérifier la présence de `"groups": ["vault-readers"]`

--

### Erreur "redirect_uri mismatch"

```bash
# Vérifier que les URIs de redirection dans Vault correspondent exactement
# à celles déclarées dans Keycloak (Client → Valid redirect URIs)

vault read auth/oidc/role/reader
# → allowed_redirect_uris doit correspondre exactement
```

---

## Récap' de la pratique

```txt
Étape 1 : Keycloak installé (Java + tar.gz)
              │
Étape 2 : Realm "demo" + Client OIDC "vault" configurés
              │
Étape 3 : Utilisateur "alice" + groupe "vault-readers" créés
              │
Étape 4 : Vault installé et démarré (mode dev)
              │
Étape 5 : Auth method OIDC configuré → Keycloak comme provider
              │
Étape 6 : Secrets créés + politiques et groupes mappés
              │
Étape 7 : Login Alice via OIDC → accès au secret validé
```

| Concept | Ce qu'on a fait |
| --------- | ---------------- |
| **Keycloak Realm** | Espace d'identité isolé `demo` |
| **Client OIDC** | Application `vault` déclarée dans Keycloak |
| **Claims JWT** | Groupes transmis dans le token (`groups_claim`) |
| **Vault OIDC auth** | Validation des tokens JWT signés par Keycloak |
| **Group mapping** | Groupe Keycloak → politique Vault automatique |
| **Least privilege** | Politique `reader` : lecture seule sur `secret/demo/*` |
| **Audit trail** | Vault et Keycloak journalisent chaque accès |

---
---
---

## 🔐 Intégration d'une deuxième application (Nextcloud) dans le SSO Keycloak

💡 Keycloak est déjà opérationnel (démo précédente). On ajoute Nextcloud comme **deuxième Service Provider** dans le même realm `demo`.

## Étape 1 — Déclarer un nouveau Client Nextcloud dans Keycloak

### Accéder à la console Keycloak

Depuis le client :

<http://10.0.0.60:8080/admin>

Se connecter avec le compte admin, sélectionner le realm **`demo`**.

### Créer le client OIDC pour Nextcloud

Dans le realm `demo` :

1. **Clients** → **Create client**
2. Onglet **"General Settings"** :

| Champ | Valeur |
|-------|--------|
| Client type | OpenID Connect |
| Client ID | `nextcloud` |
| Name | `Nextcloud` |

3. Cliquer **"Next"**

### Paramètres du flux d'authentification

Onglet **"Capability config"** :

| Option | Valeur |
|--------|--------|
| Client authentication | ON (confidential) |
| Standard flow | ON |
| Direct access grants | OFF |

Cliquer **"Next"**

### URLs de redirection Nextcloud

Onglet **"Login settings"** :

| Champ | Valeur |
|-------|--------|
| Root URL | `https://10.0.0.64` |
| Valid redirect URIs | `https://10.0.0.64/index.php/apps/user_oidc/code` |
| Valid redirect URIs | `https://10.0.0.64/*` |
| Web origins | `https://10.0.0.64` |

Cliquer **"Save"**

### Récupérer le Client Secret

1. **Clients** → `nextcloud` → onglet **"Credentials"**
2. Copier la valeur du **Client secret**

```sh
# Exemple :
Client ID     : nextcloud
Client Secret : ZAIpAgdWsnb8mHdd11bF4w2Xd8AwA5Gv
```

> 💡 Ce secret sera utilisé pour configurer l'app OIDC dans Nextcloud.

### Ajouter le mapper de groupes pour Nextcloud

Le client `nextcloud` a besoin que les groupes soient transmis dans le token JWT.

1. **Clients** → `nextcloud` → onglet **"Client scopes"**
2. Cliquer sur `nextcloud-dedicated`
3. **"Add mapper"** → **"By configuration"** → **"Group Membership"**
4. Configurer :

| Champ | Valeur |
|-------|--------|
| Name | `groups` |
| Token Claim Name | `groups` |
| Full group path | OFF |
| Add to ID token | ON |
| Add to access token | ON |

Cliquer **"Save"**

### Créer un groupe dédié Nextcloud

Dans le realm `demo` :

1. **Groups** → **"Create group"**
2. Name : `nextcloud-users`
3. Cliquer **"Create"**

### Ajouter Alice au groupe Nextcloud

1. **Users** → `alice` → onglet **"Groups"**
2. Cliquer **"Join Group"**
3. Sélectionner `nextcloud-users`
4. Cliquer **"Join"**

→ Alice appartient maintenant aux groupes `vault-readers` **et** `nextcloud-users`

---

## Étape 2 — Installer Nextcloud sur 10.0.0.64

### Préparer le système (LXC Debian/Ubuntu)

Sur le conteneur **nextcloud** (10.0.0.64) :

```bash
apt update && apt upgrade -y

# Installer Apache, PHP et les extensions nécessaires à Nextcloud
apt install -y apache2 \
    php php-cli php-common \
    php-curl php-gd php-xml php-mbstring \
    php-zip php-intl php-bcmath \
    php-gmp php-imagick php-apcu \
    libapache2-mod-php \
    mariadb-server \
    php-mysql \
    unzip wget curl bzip2

rm /var/www/html/index.html
```

### Configurer MariaDB

```bash
# Démarrer MariaDB
systemctl enable --now mariadb

# Sécuriser l'installation
mysql_secure_installation
# → Répondre Y à toutes les questions, définir un mot de passe root
```

```bash
# Créer la base de données Nextcloud
mysql -u root -p << 'EOF'
CREATE DATABASE IF NOT EXISTS nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER IF NOT EXISTS 'ncuser'@'localhost' IDENTIFIED BY 'NcDbPass42!';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'ncuser'@'localhost';
FLUSH PRIVILEGES;
EOF
```

### Télécharger et installer Nextcloud

```bash
# Télécharger la dernière version de Nextcloud
wget https://download.nextcloud.com/server/releases/latest.tar.bz2

# Extraire dans le répertoire web
tar -xjf latest.tar.bz2 -C /var/www/html/

# Ajuster les permissions
chown -R www-data:www-data /var/www/html/nextcloud
chmod -R 750 /var/www/html/nextcloud
```

### Générer un certificat auto-signé

Nextcloud **impose HTTPS** pour utiliser OpenID Connect.

```bash
# Générer un certificat auto-signé
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/nextcloud.key \
    -out /etc/ssl/certs/nextcloud.crt \
    -subj "/CN=10.0.0.64/O=Lab/C=FR"
```

### Configurer Apache pour Nextcloud (HTTPS)

```bash
# Activer les modules Apache nécessaires
a2enmod rewrite headers env dir mime ssl

# Créer le VirtualHost HTTPS
cat > /etc/apache2/sites-available/nextcloud.conf << 'EOF'
<VirtualHost *:80>
    ServerName 10.0.0.64
    Redirect permanent / https://10.0.0.64/
</VirtualHost>

<VirtualHost *:443>
    DocumentRoot /var/www/html/nextcloud
    ServerName 10.0.0.64

    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/nextcloud.crt
    SSLCertificateKeyFile /etc/ssl/private/nextcloud.key

    <Directory /var/www/html/nextcloud/>
        Options +FollowSymlinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
    CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
</VirtualHost>
EOF

# Activer le site et désactiver le site par défaut
a2ensite nextcloud.conf
a2dissite 000-default.conf
systemctl restart apache2
```

### Finaliser l'installation via CLI (occ)

```bash
cd /var/www/html/nextcloud

sudo -u www-data php occ maintenance:install \
    --database="mysql" \
    --database-name="nextcloud" \
    --database-user="ncuser" \
    --database-pass="NcDbPass42!" \
    --admin-user="admin" \
    --admin-pass="Admin123!"
```

```txt
# → Nextcloud was successfully installed
```

### Ajouter l'IP dans les trusted domains

```bash
sudo -u www-data php occ config:system:set trusted_domains 1 \
    --value="10.0.0.64"

# Forcer HTTPS dans la config Nextcloud
sudo -u www-data php occ config:system:set overwrite.cli.url \
    --value="https://10.0.0.64"
sudo -u www-data php occ config:system:set overwriteprotocol \
    --value="https"

# Autoriser les requêtes vers des serveurs locaux (Keycloak sur LAN)
# Nextcloud 28+ bloque par défaut les IPs privées (protection SSRF)
sudo -u www-data php occ config:system:set allow_local_remote_servers \
    --value=true --type=boolean
```

> ✅ Nextcloud est accessible sur `https://10.0.0.64`
>
> ⚠️ Le navigateur affichera un avertissement de certificat auto-signé — cliquer **"Avancer quand même"**.

![nexcloud](/images/2026-03-06-00-20-47.png)

---

## Étape 3 — Installer et configurer l'app OIDC dans Nextcloud

### Installer l'application `user_oidc`

Nextcloud dispose d'une application officielle pour l'authentification OIDC.

```bash
cd /var/www/html/nextcloud

# Installer l'app via occ
sudo -u www-data php occ app:install user_oidc

# Vérifier l'installation
sudo -u www-data php occ app:list | grep user_oidc
# → user_oidc: 8.x.x
```

### Activer l'application

```bash
sudo -u www-data php occ app:enable user_oidc
# → user_oidc enabled
```

### Configurer le provider OIDC (Keycloak)

```bash
sudo -u www-data php occ user_oidc:provider keycloak \
    --clientid="nextcloud" \
    --clientsecret="ZAIpAgdWsnb8mHdd11bF4w2Xd8AwA5Gv" \
    --discoveryuri="http://10.0.0.60:8080/realms/demo/.well-known/openid-configuration" \
    --unique-uid=0 \
    --check-bearer=0 \
    --mapping-display-name="name" \
    --mapping-email="email" \
    --mapping-groups="groups" \
    --group-provisioning=1
```

| Paramètre | Signification |
| ----------- | --------------- |
| `--clientid` | Client ID déclaré dans Keycloak |
| `--clientsecret` | Secret récupéré dans Keycloak |
| `--discoveryuri` | URL d'autodiscovery du realm Keycloak |
| `--unique-uid=0` | Utilise le claim `preferred_username` comme UID (pas le `sub`) |
| `--mapping-groups` | Claim JWT contenant les groupes |
| `--group-provisioning=1` | Crée automatiquement les groupes dans Nextcloud |

> ⚠️ En v8.5, le sous-commande `add` n'existe plus — l'identifiant (`keycloak`) est un argument positionnel direct.

### Vérifier la configuration OIDC

```bash
# Afficher le provider "keycloak"
sudo -u www-data php occ user_oidc:provider keycloak
```

```sh
identifier: keycloak
clientid: nextcloud
discoveryuri: http://10.0.0.60:8080/realms/demo/.well-known/openid-configuration
```

> ⚠️ En v8.5, `user_oidc:provider list` n'existe pas — passer l'identifiant directement pour afficher un provider.

--

### Désactiver l'écran de login par mot de passe (optionnel)

Pour forcer l'authentification uniquement via OIDC :

```bash
sudo -u www-data php occ config:app:set user_oidc allow_multiple_user_backends \
    --value="0"
```

> ⚠️ À ne faire qu'**après** avoir validé le SSO — sinon plus d'accès si Keycloak est down.

---

## Étape 4 — Tester l'authentification SSO

### Accéder à Nextcloud avec le SSO

Depuisle client, ouvrir un navigateur :

<https://10.0.0.64>

1. La page de login Nextcloud affiche un bouton **"Se connecter avec keycloak"**
2. Cliquer sur ce bouton
3. Être redirigé vers la page de login Keycloak (`http://10.0.0.60:8080/...`)
4. Se connecter avec `alice` / `Alice123!`
5. Être redirigé vers Nextcloud, connecté en tant qu'Alice

![demo](/images/2026-03-06-00-24-20.png)

![alice](/images/2026-03-06-00-28-40.png)

### Vérifier la session Alice dans Nextcloud

Depuis le Client (connecté en tant qu'Alice) :

1. Cliquer sur l'avatar en haut à droite → **"Paramètres personnels"**
2. Vérifier :

| Champ | Valeur attendue |
| ------- | ---------------- |
| Nom d'utilisateur | `alice` (ou le `sub` JWT) |
| Nom affiché | `Alice Demo` |
| Email | `alice@demo.local` |
| Backend | `user_oidc` |

### Vérifier les groupes dans Nextcloud

```bash
# Depuis nextcloud (10.0.0.64)
sudo -u www-data php occ user:list

sudo -u www-data php occ user:info <ID alice>
```

```sh
user_id: alice
display_name: Alice Demo
email: alice@demo.local
groups:
  - nextcloud-users
```

→ ✅ Alice appartient au groupe `nextcloud-users` (hérité de Keycloak)

![alice](/images/2026-03-06-00-33-35.png)

### Tester la déconnexion SSO

1. Dans Nextcloud, se déconnecter
2. Être redirigé vers la page Keycloak
3. Ouvrir maintenant `http://10.0.0.62:8200/ui` (Vault)
4. Cliquer **"Sign in with OIDC"**
5. Keycloak **ne redemande pas** les credentials (session SSO active)
6. Vault est accessible directement

→ ✅ Le SSO fonctionne entre les deux applications (Vault **et** Nextcloud)

---

## Étape 5 — Vérification end-to-end (multi-applications)

### Flux SSO complet — deux applications

```txt
Alice (navigateur)
    │
    ├─1─► Nextcloud : "Se connecter avec keycloak"
    │
    ├─2─► Keycloak : page de login (première fois)
    │
    ├─3─► Alice saisit ses credentials (une seule fois)
    │
    ├─4─► Keycloak émet un token JWT signé
    │       (claims: sub=alice, groups=[vault-readers, nextcloud-users])
    │
    ├─5─► Nextcloud valide le JWT → Alice connectée
    │       → groupe "nextcloud-users" synchronisé
    │
    └─6─► Plus tard : Alice accède à Vault (http://10.0.0.62:8200)
              → Keycloak réutilise la session existante
              → Pas de nouveau login requis ✅
```

### Tableau de vérification

| Test | Résultat attendu |
| ------ | ----------------- |
| Keycloak opérationnel | `curl http://10.0.0.60:8080/realms/demo` → JSON |
| Nextcloud accessible | `curl https://10.0.0.64` → page HTML |
| Discovery OIDC Nextcloud | `occ user_oidc:provider keycloak` → config affichée |
| Login Alice via SSO | Bouton Keycloak → redirection → connectée |
| Groupes synchronisés | `occ user:info alice` → `nextcloud-users` |
| SSO entre apps | Vault accessible sans re-login après Nextcloud |

---

## Dépannage

### Nextcloud ne trouve pas le provider OIDC

```bash
# Vérifier que l'URL de découverte est accessible depuis nextcloud
curl -s "http://10.0.0.60:8080/realms/demo/.well-known/openid-configuration" \
    | python3 -m json.tool | grep -E "issuer|authorization"

# Vérifier la config de l'app
sudo -u www-data php occ user_oidc:provider keycloak
```

### Erreur "redirect_uri mismatch"

```bash
# Vérifier que l'URI de callback dans Keycloak correspond exactement
# Keycloak → Clients → nextcloud → Valid redirect URIs :
# → https://10.0.0.64/index.php/apps/user_oidc/code
```

> Nextcloud envoie `/index.php/apps/user_oidc/code` (avec `index.php`) dans la requête OIDC.
> La valeur configurée dans Keycloak doit correspondre exactement.

### L'utilisateur n'est pas créé après le premier login

```bash
# Vérifier les logs Nextcloud
tail -f /var/www/html/nextcloud/data/nextcloud.log | python3 -m json.tool

# Vérifier que l'app est bien activée
sudo -u www-data php occ app:list | grep user_oidc

# Vérifier les claims du token JWT (adapter le secret)
curl -s -X POST \
  "http://10.0.0.60:8080/realms/demo/protocol/openid-connect/token" \
  -d "client_id=nextcloud" \
  -d "client_secret=NcDeFg9876543210AbC" \
  -d "username=alice" \
  -d "password=Alice123!" \
  -d "grant_type=password" \
  | python3 -c "
import sys,json,base64
d=json.load(sys.stdin)
token=d['access_token']
payload=token.split('.')[1]
payload += '=' * (4 - len(payload) % 4)
print(json.dumps(json.loads(base64.b64decode(payload)), indent=2))
"
```

→ Vérifier la présence de `"email"`, `"name"` et `"groups"` dans le payload

### Les groupes ne se synchronisent pas dans Nextcloud

Vérifier la configuration du mapper dans Keycloak :

1. **Clients** → `nextcloud` → **Client scopes** → `nextcloud-dedicated`
2. Vérifier la présence du mapper **Group Membership** avec `Token Claim Name = groups`
3. Vérifier que `Full group path = OFF`

```bash
# Vérifier la configuration du provider (mapping groupes)
sudo -u www-data php occ user_oidc:provider keycloak
# → Vérifier que group-provisioning=1 et mapping-groups=groups
```

---

## Récap' de la pratique

```txt
Démo 2 (acquis) :
    Keycloak → realm "demo" + client "vault" + Alice + vault-readers
              │
Démo 3 (nouveau) :
    Étape 1 : Client OIDC "nextcloud" ajouté dans Keycloak
              │    + groupe "nextcloud-users" + Alice membre
              │
    Étape 2 : Nextcloud installé (Apache + PHP + MariaDB)
              │
    Étape 3 : App user_oidc configurée → Keycloak comme provider
              │
    Étape 4 : SSO validé → Alice connectée via Keycloak
              │
    Étape 5 : SSO cross-apps → Vault + Nextcloud, un seul login
```

| Concept | Ce qu'on a fait |
| --------- |---------------- |
| **Multi-SP** | Deux Service Providers (Vault + Nextcloud) sur un seul IdP |
| **Client OIDC** | Un client Keycloak par application |
| **app user_oidc** | Module Nextcloud officiel pour l'auth OIDC |
| **Provisioning JIT** | Utilisateur créé dans Nextcloud au premier login |
| **Group sync** | Groupes Keycloak propagés dans Nextcloud |
| **SSO réel** | Un seul login Keycloak pour accéder aux deux apps |
| **Least privilege** | Groupes distincts par application (`vault-readers` / `nextcloud-users`) |

---
---
---

## 🧱 Intégration d'un module d'inspection (ModSecurity) sur notre point d'entrée

## Architecture

```txt
  [attacker]          [waf-proxy]            [web]
  curl / nikto  →→→  Nginx + ModSec  →→→  Apache
  10.0.0.83          10.0.0.80              10.0.0.81
```

| Machine | Type | IP | Rôle |
|---------|------|----|------|
| web | LXC Debian 12 | 10.0.0.81 | Apache — backend |
| waf-proxy | LXC Debian 12 | 10.0.0.80 | Nginx + ModSecurity + OWASP CRS |

Les attaques sont lancées **depuis waf-proxy lui-même** (pas besoin d'une 3e VM).

---

## Étape 1 — Préparer le backend (web)

### Créer le conteneur web

Dans Proxmox → **Créer CT** :

| Champ | Valeur |
|-------|--------|
| CT ID | 181 |
| Hostname | web |
| Template | debian-12 |
| Disque | 4 Go |
| CPU | 1 |
| RAM | 256 Mo |
| Bridge | vmbr2 |
| IPv4 | 10.0.0.81/24 |
| GW | 10.0.0.1 |

### Installer Apache

```bash
apt update && apt install -y apache2 curl
echo "<h1>Web backend OK</h1>" > /var/www/html/index.html
systemctl enable --now apache2
```

Vérification rapide :

```bash
curl http://10.0.0.81
# → <h1>Web backend OK</h1>
```

---

## Étape 2 — Préparer le proxy WAF

### Créer le conteneur waf-proxy

| Champ | Valeur |
|-------|--------|
| CT ID | 180 |
| Hostname | waf-proxy |
| Template | debian-12 |
| Disque | 4 Go |
| CPU | 1 |
| RAM | 512 Mo |
| Bridge | vmbr2 |
| IPv4 | 10.0.0.80/24 |
| GW | 10.0.0.1 |

### Installer Nginx + ModSecurity + CRS

Sur **waf-proxy** :

```bash
apt update && apt install -y \
    nginx \
    libnginx-mod-http-modsecurity \
    modsecurity-crs curl
```

> `modsecurity-crs` installe le paquet Debian officiel qui place les règles dans `/usr/share/modsecurity-crs/`

### Vérifier les fichiers installés

```bash
ls /usr/lib/nginx/modules/ | grep modsecurity
# → ngx_http_modsecurity_module.so

ls /usr/share/modsecurity-crs/rules/ | head -5
# → règles CRS prêtes à l'emploi

ls /etc/modsecurity/crs/
# → crs-setup.conf
# → REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
# → RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf
```

![rules](/images/2026-03-06-00-42-49.png)

---

## Étape 3 — Configurer ModSecurity

### Créer la config de base

Le paquet Debian ne fournit pas de `modsecurity.conf-recommended`. On le crée avec le minimum requis :

```bash
cat > /etc/modsecurity/modsecurity.conf << 'EOF'
SecRuleEngine DetectionOnly
SecRequestBodyAccess On
SecResponseBodyAccess Off
SecAuditEngine RelevantOnly
SecAuditLog /var/log/modsec_audit.log
SecAuditLogParts ABIJDEFHZ
SecAuditLogType Serial
EOF

# Créer le fichier d'audit log (requis avant le démarrage)
touch /var/log/modsec_audit.log
```

```bash
# Vérifier le mode (DetectionOnly par défaut)
grep SecRuleEngine /etc/modsecurity/modsecurity.conf
# → SecRuleEngine DetectionOnly
```

### Créer le fichier de règles Nginx

```bash
cat > /etc/nginx/modsecurity.conf << 'EOF'
Include /etc/modsecurity/modsecurity.conf
Include /etc/modsecurity/crs/crs-setup.conf
Include /usr/share/modsecurity-crs/rules/*.conf
EOF
```

---

## Étape 4 — Configurer Nginx en reverse proxy

### Créer le VirtualHost

```bash
cat > /etc/nginx/sites-available/waf.conf << 'EOF'
server {
    listen 80;
    server_name _;

    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsecurity.conf;

    location / {
        proxy_pass http://10.0.0.81/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF

ln -s /etc/nginx/sites-available/waf.conf /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default
```

### Démarrer

```bash
nginx -t && systemctl reload nginx
```

Test — accès normal via le proxy :

```bash
curl http://10.0.0.80
# → <h1>Web backend OK</h1>  ✅
```

---

## Étape 5 — Observer les attaques (mode DetectionOnly)

### Ouvrir les logs en temps réel

**Terminal 1 :**

```bash
tail -f /var/log/modsec_audit.log
```

> Les détections ModSecurity vont dans l'audit log, pas dans le error.log nginx.

**Terminal 2 (pour envoyer les attaques) :**

### Test : Injection SQL

```bash
# Les guillemets simples doivent être encodés en %27
curl "http://10.0.0.80/?id=1%27%20OR%20%271%27%3D%271"
# Equivalent à : http://10.0.0.80/?id=1' OR '1'='1
```

→ La requête **passe** (DetectionOnly) mais l'audit log affiche :

```log
[id "949110"] [msg "Inbound Anomaly Score Exceeded (Total Score: 33)"]
[id "942100"] [msg "SQL Injection Attack Detected via libinjection"]
[severity "CRITICAL"] [tag "OWASP_CRS/3.3.7"]
```

![attack](/images/2026-03-06-00-57-00.png)

> Le CRS utilise l'**anomaly scoring** : chaque règle ajoute des points (SQLi CRITICAL = +5).
> Le score total (ici 33) est comparé au seuil (5 par défaut) — dépassé → alerte.

### Test : XSS

```bash
# Les balises < > encodées en %3C %3E
curl "http://10.0.0.80/?q=%3Cscript%3Ealert(1)%3C%2Fscript%3E"
# Equivalent à : http://10.0.0.80/?q=<script>alert(1)</script>
```

→ Log :

```log
[id "941100"] [msg "XSS Attack Detected via libinjection"]
```

![attack](/images/2026-03-06-00-55-22.png)

### Test : Path Traversal

```bash
curl "http://10.0.0.80/?file=../../../../etc/passwd"
```

→ Log :

```log
[id "930100"] [msg "Path Traversal Attack"]
```

![attack](/images/2026-03-06-00-58-24.png)![](/images/2026-03-06-00-58-24.png)

---

## Étape 6 — Bloquer les attaques (mode Prevention)

### Passer en mode Prevention

```bash
sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' \
    /etc/modsecurity/modsecurity.conf

nginx -t && systemctl reload nginx
```

### Vérifier que le trafic légitime passe toujours

```bash
curl http://10.0.0.80/
# → 200 OK ✅

curl "http://10.0.0.80/?nom=Alice&age=25"
# → 200 OK ✅
```

### Les attaques sont maintenant bloquées

```bash
curl -v "http://10.0.0.80/?id=1%27%20OR%20%271%27%3D%271"
# → HTTP/1.1 403 Forbidden  🚫

curl -v "http://10.0.0.80/?q=%3Cscript%3Ealert(1)%3C%2Fscript%3E"
# → HTTP/1.1 403 Forbidden  🚫

curl -v "http://10.0.0.80/?file=../../../../etc/passwd"
# → HTTP/1.1 403 Forbidden  🚫
```

![403](/images/2026-03-06-01-00-11.png)

### Tableau récap'

| Attaque | Sans WAF | DetectionOnly | Prevention |
| -------- | --------- | -------------- | ----------- |
| SQL Injection | passe | passe + log | 403 bloqué |
| XSS | passe | passe + log | 403 bloqué |
| Path Traversal | passe | passe + log | 403 bloqué |
| Requête normale | passe | passe | passe |

---

## Récap' des étapes

```text
web (10.0.0.81)         Apache backend simple
waf-proxy (10.0.0.80)   Nginx + ModSecurity + CRS (paquet apt)
                        │
                        ├─ DetectionOnly → log les attaques sans bloquer
                        └─ Prevention   → HTTP 403 sur toute requête suspecte
```
