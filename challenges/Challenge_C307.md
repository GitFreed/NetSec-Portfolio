# Challenge C307 04/03/2026

## 🧑‍🏫 Pitch de l’exercice : 🛡️ Mise en place de HTTPS sur une ou plusieurs apps, de reverse-proxy et load-balancer avec Nginx, Apache et HAProxy. Durcissement, Mitigation DDoS, et rate-limiting

[Cours C307.](/RESUME.md#-c307-reverse-proxy-load-balancer-https--anti-ddos)

---

### 🎯 Contexte

Vous êtes le responsable réseau d'une entreprise dont les services web gagnent en visibilité.
Vos serveurs applicatifs ne doivent en aucun cas être exposés directement sur Internet. Votre mission est de concevoir, déployer et sécuriser la "frontière" de votre réseau (l'Edge Network). Vous devez mettre en place un point de passage unique et robuste chargé de trois missions critiques : chiffrer les communications de bout en bout (Terminaison TLS), répartir intelligemment la charge de trafic vers une grappe de serveurs internes (Load-Balancing), et rejeter massivement les attaques par inondation (Anti-DDoS) avant même qu'elles n'atteignent la couche système.

### 🖥️ Environnement technique

* **Infrastructure Virtuelle :** Proxmox VE avec un réseau ponté isolé (Bridge `vmbr2` sur le subnet `10.0.0.0/24`).
* **Nœud Frontal (Edge/Proxy) :** Un conteneur LXC dédié faisant office de bouclier et de routeur applicatif (avec Apache, Nginx et HAProxy).
* **Nœuds Back-end (Cœur de réseau) :** Un cluster de 3 conteneurs LXC internes exécutant les serveurs web finaux (Apache).
* **Sécurité Périmétrique :** Durcissement de la pile TCP du noyau (`sysctl`), pare-feu de couche 3/4 (`iptables`), et moteur de limitation de flux en mémoire (*stick-tables* HAProxy).

### 🔎 Points de vigilance

* **Le Single Point of Failure (SPOF) :** Dans cette maquette, si le nœud Proxy tombe, le routage s'arrête et tous les sites sont inaccessibles. En production réelle, il est impératif de doubler ce frontal avec un mécanisme de haute disponibilité réseau (comme VRRP avec Keepalived) et une IP virtuelle flottante.
* **La perte de l'IP source (Traçabilité) :** À cause du Reverse Proxy (qui agit comme un intermédiaire), les serveurs internes ne voient que l'adresse IP du Proxy, et non celle du client final. Il est vital de s'assurer que l'en-tête HTTP `X-Forwarded-For` est bien injecté par le proxy pour préserver l'analyse des flux dans les logs internes.
* **Le "Friendly Fire" lors du filtrage :** En mettant en place des seuils de blocage stricts (Rate-Limiting), le risque d'auto-bannissement est réel. Il faut systématiquement exclure (Whitelist) les plages IP d'administration, les locaux de l'entreprise ou les sondes de monitoring internes pour éviter de se couper soi-même du réseau.
* **Le dimensionnement du chiffrement :** Le *SSL Offloading* (déchiffrer tout le trafic entrant au niveau du proxy) centralise la gestion des certificats, mais consomme énormément de ressources CPU. Ce point d'entrée doit être dimensionné en conséquence pour encaisser les pics de connexions simultanées sans créer de latence sur le réseau.

---

## 🛠️ Reverse-proxy, Load-balancer & HTTPS

### Architecture du lab

```txt
                    [Client]
                    Lubuntu — 10.0.0.50
                       │
          ┌────────────┴────────────────────────────────┐
          │           vmbr2 — LAN A (10.0.0.0/24)       │
          └──┬──────────┬──────────┬──────────┬─────────┘
             │          │          │          │
         10.0.0.30  10.0.0.20  10.0.0.21  10.0.0.22
             │          │          │          │
          [proxy]    [web1]     [web2]     [web3]
          Nginx      Apache     Apache     Apache
          HAProxy    Site A     Site B     Site C
```

| Machine | Type | IP | Rôle |
| --------- | ------ | ---- | ------ |
| web1 | LXC | 10.0.0.20 | Serveur Apache — Site A 🔵 |
| web2 | LXC | 10.0.0.21 | Serveur Apache — Site B 🟢 |
| web3 | LXC | 10.0.0.22 | Serveur Apache — Site C 🔴 |
| proxy | LXC | 10.0.0.30 | Reverse Proxy + Load Balancer |
| Lubuntu | VM | 10.0.0.50 | Client pour tester |

---

### Créer l'infra Proxmox 🏗️

Créer les 4 conteneurs LXC et vérification de la connectivité sur chaque

```bash
ping -c 2 10.0.0.1    # Gateway (pfSense)
ping -c 2 8.8.8.8     # Internet
ping -c 2 google.com  # DNS
```

### Sites web Apache 🌐

#### Installer Apache sur web1, web2 et web3

Sur **chaque** conteneur :

```bash
apt update && apt install -y apache2
systemctl status apache2
```

#### Créer un site distinct sur chaque serveur

**Sur web1** (10.0.0.20) :

```bash
cat > /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>Web1</title></head>
<body style="background-color: #3498db; color: white;
  text-align: center; padding-top: 50px; font-family: Arial;">
    <h1>🔵 Serveur WEB1</h1>
    <p>IP : 10.0.0.20 — Site A</p>
</body>
</html>
EOF
```

**Sur web2** (10.0.0.21) — couleur verte :

```bash
cat > /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>Web2</title></head>
<body style="background-color: #2ecc71; color: white;
  text-align: center; padding-top: 50px; font-family: Arial;">
    <h1>🟢 Serveur WEB2</h1>
    <p>IP : 10.0.0.21 — Site B</p>
</body>
</html>
EOF
```

**Sur web3** (10.0.0.22) — couleur rouge :

```bash
cat > /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>Web3</title></head>
<body style="background-color: #e74c3c; color: white;
  text-align: center; padding-top: 50px; font-family: Arial;">
    <h1>🔴 Serveur WEB3</h1>
    <p>IP : 10.0.0.22 — Site C</p>
</body>
</html>
EOF
```

*Les couleurs différentes permettront de voir quel serveur répond pendant le load-balancing* 💡

#### Vérification

Depuis l'hôte, ouvrir un navigateur :

* `http://10.0.0.20` → Page bleue (WEB1)
* `http://10.0.0.21` → Page verte (WEB2)
* `http://10.0.0.22` → Page rouge (WEB3)

✅ Tout est en HTTP (port 80, non chiffré) pour l'instant

### HTTPS sur Apache (certificat auto-signé) 🔒

#### Générer le certificat sur web1

```bash
# Activer le module SSL
a2enmod ssl

# Créer le dossier
mkdir -p /etc/apache2/ssl

# Générer clé privée + certificat auto-signé
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/apache2/ssl/web1.key \
  -out /etc/apache2/ssl/web1.crt \
  -subj "/C=FR/ST=France/L=Paris/O=Lab/CN=web1.lab.local"
```

![cert](/images/2026-03-04-17-06-29.png)

#### Configurer le VirtualHost HTTPS

```bash
cat > /etc/apache2/sites-available/web1-ssl.conf << 'EOF'
<VirtualHost *:443>
    ServerName web1.lab.local
    DocumentRoot /var/www/html

    SSLEngine on
    SSLCertificateFile /etc/apache2/ssl/web1.crt
    SSLCertificateKeyFile /etc/apache2/ssl/web1.key
</VirtualHost>
EOF
```

```bash
a2ensite web1-ssl.conf
systemctl restart apache2
```

#### Vérification Https

Depuis l'hôte : `https://10.0.0.20`

→ **Avertissement de sécurité** (normal, certificat auto-signé)

→ Accepter → Page bleue en HTTPS ✅

🔒 Le chiffrement fonctionne, c'est juste la **confiance** du navigateur qui manque (pas de CA reconnue)

![warn](/images/2026-03-04-17-09-28.png)

#### (Optionnel) Rediriger HTTP → HTTPS

```bash
cat > /etc/apache2/sites-available/000-default.conf << 'EOF'
<VirtualHost *:80>
    ServerName web1.lab.local
    Redirect permanent / https://10.0.0.20/
</VirtualHost>
EOF

systemctl restart apache2
```

`http://10.0.0.20` redirige maintenant automatiquement vers `https://10.0.0.20`

### Reverse-proxy Nginx 🟢

#### Installer Nginx sur le conteneur proxy

```bash
apt update && apt install -y nginx
```

#### Générer le certificat

```bash
mkdir -p /etc/nginx/ssl

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/proxy.key \
  -out /etc/nginx/ssl/proxy.crt \
  -subj "/C=FR/ST=France/L=Paris/O=Lab/CN=proxy.lab.local"
```

#### Configurer le reverse-proxy et HTTPS 🔐

```bash
cat > /etc/nginx/sites-available/reverse-proxy.conf << 'EOF'
server {
    listen 80;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name _;

    ssl_certificate /etc/nginx/ssl/proxy.crt;
    ssl_certificate_key /etc/nginx/ssl/proxy.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        return 200 '<!DOCTYPE html>
<html><head><title>Proxy HTTPS</title></head>
<body style="background-color: #2c3e50; color: white;
  text-align: center; padding-top: 50px; font-family: Arial;">
    <h1>🔒 Reverse Proxy Nginx (HTTPS)</h1>
    <p><a href="/web1/" style="color: #3498db;">Web1</a>
     | <a href="/web2/" style="color: #2ecc71;">Web2</a>
     | <a href="/web3/" style="color: #e74c3c;">Web3</a></p>
</body></html>';
        add_header Content-Type text/html;
    }

    location /web1/ {
        proxy_pass http://10.0.0.20/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /web2/ {
        proxy_pass http://10.0.0.21/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /web3/ {
        proxy_pass http://10.0.0.22/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
```

#### Activer et tester

```bash
rm -f /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/reverse-proxy.conf /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

#### Vérification nginx

Depuis l'hôte  :

* `http://10.0.0.30` → Redirige vers `https://10.0.0.30`
* `https://10.0.0.30` → Page d'accueil avec le cadenas et les 3 liens 🔒
* `https://10.0.0.30/web1/` → Page bleue, en HTTPS

💡 Le certificat n'est que sur le proxy. Les 3 back-ends restent en HTTP simple (sauf web1 qui redirige)

### Reverse-proxy Apache (comparaison) 🔴

#### Installer Apache sur le proxy

On le fait tourner sur le port **8443** pour ne pas entrer en conflit avec Nginx :

```bash
apt install -y apache2
a2enmod proxy proxy_http ssl rewrite headers
```

#### Configurer Apache comme reverse-proxy

```bash
cat > /etc/apache2/sites-available/reverse-proxy-ssl.conf << 'EOF'
Listen 8443

<VirtualHost *:8443>
    ServerName proxy.lab.local

    SSLEngine on
    SSLCertificateFile /etc/nginx/ssl/proxy.crt
    SSLCertificateKeyFile /etc/nginx/ssl/proxy.key

    ProxyPreserveHost On

    ProxyPass /web1/ http://10.0.0.20/
    ProxyPassReverse /web1/ http://10.0.0.20/

    ProxyPass /web2/ http://10.0.0.21/
    ProxyPassReverse /web2/ http://10.0.0.21/

    ProxyPass /web3/ http://10.0.0.22/
    ProxyPassReverse /web3/ http://10.0.0.22/
</VirtualHost>
EOF
```

💡 nginx écoutant le même port, il y a conflit avec apache, il faut donc le désactiver. En production, on utiliserait **l'un ou l'autre**, pas les deux. L'objectif ici est de **comparer**

```sh
systemctl stop nginx
systemctl disable nginx
```

Finir la configuration et restart apache

```bash
a2dissite 000-default.conf
a2ensite reverse-proxy-ssl.conf
sed -i 's/Listen 80/Listen 8080/' /etc/apache2/ports.conf
systemctl restart apache2
```

#### Vérification apache

Depuis l'hôte :

* `https://10.0.0.30:8443/web1/` → Page bleue via Apache
* `https://10.0.0.30:8443/web2/` → Page verte via Apache
* `https://10.0.0.30:8443/web3/` → Page rouge via Apache

### Load-balancer HAProxy ⚖️

#### Préparer : libérer les ports

```bash
systemctl stop nginx apache2
systemctl disable nginx apache2
```

#### Installer HAProxy

```bash
apt install -y haproxy
```

#### Préparer le certificat

HAProxy a besoin d'un **seul fichier** `.pem` :

```bash
mkdir -p /etc/haproxy/ssl
cat /etc/nginx/ssl/proxy.crt /etc/nginx/ssl/proxy.key \
  > /etc/haproxy/ssl/proxy.pem
chmod 600 /etc/haproxy/ssl/proxy.pem
```

#### Configurer HAProxy

```bash
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
```

```bash
cat > /etc/haproxy/haproxy.cfg << 'EOF'
global
    log /dev/log local0
    log /dev/log local1 notice
    maxconn 2000
    daemon
    ssl-default-bind-ciphers HIGH:!aNULL:!MD5
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    option  forwardfor
    timeout connect 5000ms
    timeout client  50000ms
    timeout server  50000ms
    errorfile 503 /etc/haproxy/errors/503.http

# === FRONTEND : ce que voit le client ===
frontend http_front
    bind *:80
    http-request redirect scheme https code 301

frontend https_front
    bind *:443 ssl crt /etc/haproxy/ssl/proxy.pem
    default_backend web_servers

    stats enable
    stats uri /stats
    stats auth admin:admin
    stats refresh 5s

# === BACKEND : les serveurs réels ===
backend web_servers
    balance roundrobin
    option httpchk GET /
    http-check expect status 200

    server web1 10.0.0.20:80 check inter 5s fall 3 rise 2
    server web2 10.0.0.21:80 check inter 5s fall 3 rise 2
    server web3 10.0.0.22:80 check inter 5s fall 3 rise 2
EOF
```

#### Démarrer HAProxy

```bash
haproxy -c -f /etc/haproxy/haproxy.cfg   # vérifier la syntaxe
systemctl restart haproxy
systemctl enable haproxy
```

### Tests de validation ✅

#### Test 1 : Load-balancing (Round Robin)

Ouvrir `https://10.0.0.30` — rafraîchir plusieurs fois (F5) :

🔵 WEB1 → 🟢 WEB2 → 🔴 WEB3 → 🔵 WEB1 → ...

⚠️ Si toujours le même serveur → le navigateur utilise le cache. Tester avec `curl` :

```bash
curl -k https://10.0.0.30
curl -k https://10.0.0.30
curl -k https://10.0.0.30
```

#### Test 2 : Page de stats HAProxy

Ouvrir `https://10.0.admin0.30/stats` — login : **admin** / **admin**

On y voit :

* L'état de chaque serveur (vert = UP, rouge = DOWN)
* Le nombre de requêtes par serveur
* Les temps de réponse
* Les connexions actives

![stats](/images/2026-03-04-18-29-12.png)

#### Test 3 : Haute disponibilité

Simuler une panne — sur web2 :

```bash
systemctl stop apache2
```

Attendre ~15 secondes, puis rafraîchir `https://10.0.0.30`

→ Les requêtes ne vont plus que vers **web1** et **web3**

→ Dans `/stats` : web2 est en rouge (DOWN) 🔴

![stats](/images/2026-03-04-18-30-48.png)

Relancer web2 :

```bash
systemctl start apache2
```

→ Après ~10 secondes, web2 repasse en vert et reçoit à nouveau des requêtes ✅

#### Test 4 : Vérifier le chiffrement

Cliquer sur le **cadenas** dans le navigateur → Détails du certificat :

* Émetteur : Lab
* Objet : proxy.lab.local
* Protocole : TLS 1.2 ou 1.3

![cert](/images/2026-03-04-18-46-04.png)

---

### Récap' de la pratique 📝

```txt
Étape 1-2 : Site web basique en HTTP
                    │
Étape 3   : HTTPS avec certificat auto-signé
                    │
Étape 4-5 : Reverse Proxy Nginx + terminaison TLS
                    │
Étape 6   : Reverse Proxy Apache (comparaison)
                    │
Étape 7   : Load Balancer HAProxy
                    │
Étape 8   : Tests (round robin, stats, haute dispo)
```

| Concept | Ce qu'on a fait |
| --------- | ---------------- |
| **HTTPS / TLS** | Certificat auto-signé sur Apache |
| **Reverse-proxy** | Nginx et Apache devant 3 back-ends |
| **Terminaison TLS** | HTTPS sur le proxy, HTTP vers les back-ends |
| **Load-balancing** | Round Robin avec HAProxy |
| **Health checks** | Détection auto des pannes serveur |
| **Haute dispo** | Exclusion / réintégration automatique |
| **Monitoring** | Page de stats HAProxy |

---

## 🔥 Anti-DDoS & Rate-limiting

```txt
[Win11 10.0.0.10]
        │  HTTPS
        ▼
[proxy 10.0.0.30]  ← HAProxy (round-robin, health checks)
        │  HTTP
        ├─► web1 10.0.0.20
        ├─► web2 10.0.0.21
        └─► web3 10.0.0.22
```

### Rate-limiting avec HAProxy (stick-table) ⚖️

HAProxy est déjà en place : on ajoute le rate-limiting dans la config existante.

```bash
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak2

cat > /etc/haproxy/haproxy.cfg << 'EOF'
global
    log /dev/log local0
    log /dev/log local1 notice
    maxconn 2000
    daemon
    ssl-default-bind-ciphers HIGH:!aNULL:!MD5
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    option  forwardfor
    timeout connect 5000ms
    timeout client  50000ms
    timeout server  50000ms
    errorfile 503 /etc/haproxy/errors/503.http

# === FRONTEND HTTP → HTTPS ===
frontend http_front
    bind *:80
    http-request redirect scheme https code 301

# === FRONTEND HTTPS avec rate-limiting ===
frontend https_front
    bind *:443 ssl crt /etc/haproxy/ssl/proxy.pem

    # Stick-table : compte requêtes/min et connexions simultanées par IP
    stick-table type ip size 100k expire 60s store http_req_rate(60s),conn_cur

    http-request track-sc0 src
    tcp-request connection track-sc0 src

    # Bloquer si > 100 requêtes/minute par IP → 429
    http-request deny deny_status 429 if { sc_http_req_rate(0) gt 100 }

    # Bloquer si > 20 connexions simultanées par IP → 429
    tcp-request connection reject if { sc_conn_cur(0) gt 20 }

    default_backend web_servers

    stats enable
    stats uri /stats
    stats auth admin:admin
    stats refresh 5s

# === BACKEND ===
backend web_servers
    balance roundrobin
    option httpchk GET /
    http-check expect status 200

    server web1 10.0.0.20:80 check inter 5s fall 3 rise 2
    server web2 10.0.0.21:80 check inter 5s fall 3 rise 2
    server web3 10.0.0.22:80 check inter 5s fall 3 rise 2
EOF
```

```bash
haproxy -c -f /etc/haproxy/haproxy.cfg
systemctl restart haproxy
```

#### Étape 2 — Tests 🧪

#### Installer les outils

```bash
apt install -y apache2-utils siege
```

#### Test avec HAProxy rate-limiting

La commande suivante demande 200 requêtes au total (-n 200), mais avec une simultanéité de 50 (-c 50). Dès la première milliseconde, l'outil ab (ApacheBench) tente d'ouvrir 50 sessions TCP en même temps. Il envoie une rafale de paquets SYN vers le port 443.

```bash
# 200 requêtes, 50 simultanées — dépasse la limite de 100/min
ab -n 200 -c 50 -k https://10.0.0.30/
```

Observer dans la sortie :

```sh
Non-2xx responses:   XX     ← réponses 429 (bloquées)
Failed requests:     XX
```

![failed](/images/2026-03-04-19-14-06.png)

### Durcissement OS & Nginx

On peut aller plus loin en durcissant le système et en ajoutant du rate-limiting au niveau de Nginx (en frontal ou en complément de HAProxy).

#### Durcissement OS : SYN cookies 🛡️

On protège d'abord la couche TCP contre le SYN flood.

```bash
# Vérifier l'état actuel
sysctl net.ipv4.tcp_syncookies

# Activer les SYN cookies + durcir la file SYN
sysctl -w net.ipv4.tcp_syncookies=1
sysctl -w net.ipv4.tcp_max_syn_backlog=4096
sysctl -w net.ipv4.tcp_synack_retries=2
```

Rendre permanent :

```bash
cat >> /etc/sysctl.conf << 'EOF'
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_synack_retries = 2
EOF

sysctl -p
```

#### Limiter les SYN avec iptables 🔥

```bash
# Max 10 nouvelles connexions SYN/seconde par IP (burst 20)
iptables -A INPUT -p tcp --syn \
    -m hashlimit \
    --hashlimit-name syn_limit \
    --hashlimit 10/sec \
    --hashlimit-burst 20 \
    --hashlimit-mode srcip \
    -j ACCEPT

# Rejeter tout SYN au-delà
iptables -A INPUT -p tcp --syn -j DROP

# Vérifier les règles
iptables -L INPUT -v --line-numbers
```

> 💡 Ces deux étapes (sysctl + iptables) se font sur **tous les serveurs exposés**, pas seulement le proxy.

#### Rate-limiting avec Nginx (comparaison) 🟢

On relance Nginx en frontal **à la place de HAProxy** pour comparer l'approche.

```bash
systemctl stop haproxy

# Définir les zones de comptage dans nginx.conf
sed -i '/http {/a \    limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;\n    limit_req_zone $binary_remote_addr zone=login:1m rate=1r/s;' \
    /etc/nginx/nginx.conf
```

Modifier le vhost :

```bash
cat > /etc/nginx/sites-available/reverse-proxy.conf << 'EOF'
server {
    listen 80;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name _;

    ssl_certificate     /etc/nginx/ssl/proxy.crt;
    ssl_certificate_key /etc/nginx/ssl/proxy.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    # Limite générale : 10 req/s, burst 20
    location / {
        limit_req zone=general burst=20 nodelay;
        limit_req_status 429;
        proxy_pass http://10.0.0.20/;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Limite stricte sur une route "sensible" : 1 req/s, burst 5
    location /admin {
        limit_req zone=login burst=5;
        limit_req_status 429;
        return 200 'Admin simulé';
        add_header Content-Type text/plain;
    }
}
EOF
```

```bash
nginx -t && systemctl restart nginx
```

#### Test avec Nginx rate-limiting

```bash
systemctl stop haproxy && systemctl start nginx

# 500 requêtes, 50 simultanées — dépasse 10 req/s
ab -n 500 -c 50 https://10.0.0.30/
```

Voir en direct :

```bash
tail -f /var/log/nginx/access.log | grep " 429 "
```

#### Test : Siege (plus réaliste)

```bash
siege -c 100 -t 20s https://10.0.0.30/
```

```txt
Availability:   ~65 %    ← ~35% bloquées = rate-limiting actif
Failed transactions: XXX
```

---

### Récap de ce qu'on a mis en place

```txt
[OS - proxy]         SYN cookies + iptables hashlimit
                          ↓ bloque SYN flood
[HAProxy]            stick-table : 100 req/min, 20 conn simultanées
                          ↓ bloque HTTP flood par IP
[Nginx]              limit_req_zone : 10 req/s général, 1 req/s /admin
                          ↓ bloque les rafales
```

| Mécanisme | Commande de vérification |
| ----------- | -------------------------- |
| SYN cookies actifs | `sysctl net.ipv4.tcp_syncookies` |
| Règles iptables | `iptables -L INPUT -v` |
| Connexions actives | `ss -s` |
| Logs 429 Nginx | `tail -f /var/log/nginx/access.log` |
| Stats HAProxy | `https://10.0.0.30/stats` |

---

**Pour résumer ce double challenge, nous avons mis en place une infrastructure web à la fois hautement disponible et sécurisée en profondeur :**

* **Une architecture de Load-Balancing et de Terminaison TLS (Couche 7) :** Nous avons déployé un Reverse Proxy frontal (HAProxy/Nginx/Apache) agissant comme point d'entrée unique. Il centralise le chiffrement HTTPS, puis distribue le trafic en clair (HTTP) vers trois serveurs back-end de manière équitable (Round-Robin), garantissant que la chute d'un nœud ne coupe pas le service.

* **Une protection réseau anti-flood (Couches 3 et 4) :** Nous avons durci la pile TCP du pare-feu (SYN cookies) et appliqué des règles `iptables` pour "Drop" les rafales de requêtes de connexion réseau illégitimes (SYN flood) au plus bas niveau possible, avant qu'elles n'impactent la couche système.

* **Un contrôle des flux et Rate-Limiting (Couches 4 et 7) :** Nous avons configuré le proxy pour limiter le nombre de sockets TCP ouverts simultanément par une même adresse IP, et plafonné le volume de requêtes HTTP par seconde. Cela protège la bande passante et la mémoire contre les attaques par déni de service (DDoS) ciblées sur l'applicatif.
