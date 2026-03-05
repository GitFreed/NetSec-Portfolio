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

### 🖥️ Environnement technique

---

### 🔎 Points de vigilance

---

## Déploiement d'un SSO Keycloak et intégration avec Vault
