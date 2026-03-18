# Liste de tous les Challenges et Ateliers réalisés pour valider les concepts techniques abordés en cours 💻

Commande utilisée pour extraire la liste du fichier [Resume.md](/RESUME.md)

```ps1
(Get-Content .\RESUME.md | Where-Object { $_ -like "*challenges/*" }) -join "`r`n`r`n" | Set-Clipboard
```

---

[Challenge A102](./Challenge_A102.md) : Exercice de recherche pour achat de périphériques

[Challenge A103](./Challenge_A103.md) : Description de notre Hardware

[Challenge A104](./Challenge_A104.md) : Configuration de 3 Setups selon les besoins.

[Challenge A105](./Challenge_A105.md) : Découverte de la Virtualisation, Virtualbox et VM Windows.

[Challenge A106](./Challenge_A106.md) : Agents invités (Guest addition) pour la virtualisation et VM Linux.

[Challenge A107](./Challenge_A107.md) : Gestionnaire de mots de passe.

[Challenge A108](./Challenge_A108.md) : Création d'un Diagramme Réseau.

[Challenge A109](./Challenge_A109.md) : Calculs d'adresses IP, réseau, broadcast, plage et masque.

[Challenge A201](./Challenge_A201.md) : Test et comparatif de Microsoft / Libreoffice.

[Challenge A202](./Challenge_A202.md) : Mise en réseau de VM.

[Challenge A203](./Challenge_A203.md) : Memtest, Anydest, Teamviewer.

[Atelier A204](./Challenge_A204.md) : Atelier "Mme Michu", dépannage d'une VM simulant un labtop en panne.

[Challenge A206](./Challenge_A206.md) : Test BIOS, et partitions sur USB.

[Challenge A207](./Challenge_A207.md) : QCM certification ITIL.

[Challenge A208](./Challenge_A208.md) : Installation GLPI Agent et test de ticketing.

[Challenge A301](./Challenge_A301.md) : Mise en place d'un réseau et ping sur Packet Tracer.

[Challenge A303](./Challenge_A303.md) : Création d’un plan d’adressage

[Challenge A304](./Challenge_A304.md) : Config de Routeurs et LANs sur Packet Tracer

[Atelier A305](./Challenge_A305.md) : Atelier Packet Tracer, de la création du plan d'adressage, à la config switch, routeurs, routes, DHCP, etc.

[Challenge A306](./Challenge_A306.md) : DNS et SSH dans Packet Tracer

[Challenge A307](./Challenge_A307.md) : Self Hosting, NAT et redirection dans Packet Tracer.

[Atelier A308](./Challenge_A308.md) : Installation de Proxmox sur un serveur OVH, configuration du NAT, installation et configuration du routeur pfSense, et VPN.

[Challenge A309](./Challenge_A309.md) : Pratiquer les VLAN, Inter-VLAN et ACLs

[Challenge A401](./Challenge_A401.md) : Installation de Windows Server 2022 sur VMware.

[Challenge A402](./Challenge_A402.md) : Installation WS2025 sur Proxmox, AD DS Gestion d'utilisateurs.

[Challenge A403](./Challenge_A403.md) : Utilisateurs, Groupes, et GPO Fond d'écran.

[Challenge A404](./Challenge_A404.md) : Création de partages et droits.

[Challenge A405](./Challenge_A405.md) : Mappage de lecteurs, ressources, quotas, filtres, et audit.

[Atelier A406](./Challenge_A406.md) : Atelier de mise en place et gestion de GPO.

[Challenge A408](./Challenge_A408.md) : Enregistrement DNS, IIS et index.html

[Challenge A409](./Challenge_A409.md) : Supression et récupération utilisateur dans l'AD.

[Challenge A410](./Challenge_A410.md) : Test de Windows Deployment Services (WDS) et boot PXE.

[Challenge A411](./Challenge_A411.md) : Installation et test du RDS (Remote Desktop Services).

[Challenge A412](./Challenge_A412.md) : Installation d'une VM sur Hyper-V.

[Challenge A501](./Challenge_A501.md) : Installation d'une VM Linux sur VMware, et jeu Terminus Quest pour apprendre les commandes de base.

[Challenge A502](./Challenge_A502.md) : Jouer à VIM Adventures et VIM Tutor pour se familiariser avec VIM.

[Challenge A503](./Challenge_A503.md) : Création d'utilisateur, droits et dossiers sur Rocky Linux.

[Challenge A504](./Challenge_A504.md) : Compilation d'un Binaire.

[Atelier A505](./Challenge_A505.md) : Atelier mise en place d'une stack LAMP.

[Atelier A506](./Challenge_A506.md) : Atelier mise en place de SAMBA (AD).

[Challenge A507](./Challenge_A507.md) : Installation et test d'Arch Linux.

[Challenge A508](./Challenge_A508.md) : Installation et configuration d'un serveur Asterisk.

[Challenge A509](./Challenge_A509.md) : Atelier Installer et configurer un serveur Asterisk sur proxmox, fonctionnel via app VOIP du smartphone.

[Challenge B101](./Challenge_B101.md) : Installation d'Hyperviseurs Types 1 é 2, imbrication.

[Challenge B102](./Challenge_B102.md) : Installation ESXi (vSphere)

[Challenge B103](./Challenge_B103.md) : Installation de vCenter et le configurer.

[Challenge B201](./Challenge_B201.md) : Installation de TrueNAS (Proxmox), configuration ZFS, SMB et snapshots.

[Challenge B202](./Challenge_B202.md) : Installation Veeam Backup & Replication, configuration, restauration.

[Challenge B204](./Challenge_B204.md) : Installer Proxmox Backup Server, configurer, backup et restore.

[Challenge B301](./Challenge_B301.md) : Installation de Zabbix

[Challenge B302](./Challenge_B302.md) : Installation d'Agents Zabbix pour étendre la supervision à l’ensemble de l’infrastructure.

[Challenge B303](./Challenge_B303.md) : Exploration plus en détail des paramètres, alertes de Zabbix avec les Agents, configuration d'une alerte pour fichiers.

[Challenge B304](./Challenge_B304.md) : Installation de Nagios et de ses agents.

[Challenge B401](./Challenge_B401.md) : Pratiquer sur Scratch en créant un jeu du Nombre Mystère (Plus/Moins).

[Challenge B402](./Challenge_B402.md) : Pratiquer Git & Github

[Challenge B403](./Challenge_B403.md) : Recréer le jeu du Nombre Mystère (Plus/Moins) en script python.

[Challenge B404](./Challenge_B404.md) : Créer des Scripts batch et Powershell d'Administration Système.

[Atelier B405](./Challenge_B405.md) : Atelier Powershell et AD.

[Challenge B406](./Challenge_B406.md) : Jeu du Nombre Mystère Plus ou Moins en PowerShell.

[Challenge B407](./Challenge_B407.md) : une série d'exercices pour créer une suite d'outils Admin en Bash.

[Atelier B408](./Challenge_B408.md) : Atelier Bash - Automatisation de l'administration système.

[Challenge C101](./Challenge_C101.md) : Effectuer la note de cadrage d'un projet.

[Challenge C102](./Challenge_C102.md) : Créer le WBS, matrice RACI et diagramme Gantt.

[Challenge C103](./Challenge_C103.md) : Analyser les risques pour notre projet.

[Challenge C104](./Challenge_C104.md) : Définir un scénario d’incident majeur et les mesures, procédures, PRA-PCA type.

[Challenge C201](./Challenge_C201.md) : Expérimenter les versions gratuites de Google Cloud, AWS et Azure.

[Challenge C202](./Challenge_C202.md) : Proposition d'une stratégie cloud simple, cohérente et réaliste pour une PME.

[Atelier C203](./Challenge_C203.md) : Déploiement Nextcloud

[Challenge C204](./Challenge_C204.md) : Déploiement Openstack

[Challenge C301](./Challenge_C301.md) : Segmentation VLAN & Contrôle d'accès (ACL)

[Challenge C302](./Challenge_C302.md) : Authentification réseau avec RADIUS et LDAP / AD

[Challenge C303](./Challenge_C303.md) : DMZ isolée, serveur web, règles de pare-feu, redirection de port NAT, VPN & Authentification Radius

[Atelier C304](./Challenge_C304.md) : IDP, IPS & SIEM, mise en place d'une chaîne de détection et de supervision complète : Suricata & Wazuh

[Challenge C305](./Challenge_C305.md) : Sécurisation d’un serveur Debian : durcissement du système et restriction stricte des accès SSH

[Demo C305](./Challenge_C305_demo-cmd.md) : Exemple commandes/configurations firewall et durcissement SSH

[Challenge C306](./Challenge_C306.md) : Sécurisation d'un serveur Linux contre les attaques par force brute avec Fail2ban/Crodwsec et Knockd

[Demo C306](./Challenge_C306_demo-cmd.md) : Exemple commandes/configurations PAM, Google Authenticator, last, lastb, fail2ban, knockd et crodwsec

[Challenge C307](./Challenge_C307.md) : Mise en place de HTTPS sur une ou plusieurs apps, de reverse-proxy et load-balancer avec Nginx, Apache et HAProxy. Durcissement, Mitigation DDoS, et rate-limiting

[Challenge C308](./Challenge_C308.md) : Déploiement d'un SSO Keycloak et intégration avec Vault,  intégration d'une deuxième application (Nextcloud), et sécurité couche 7 avec WAF ModSecurity + OWASP CRS

[Challenge C401](./Challenge_C401.md) : Déploiement Docker et découverte

[Challenge C402](./Challenge_C402.md) : Déployer GLPI avec Docker Compose

[Challenge C403](./Challenge_C403.md) : Déployer GLPI sur un cluster avec Portainer

[Challenge C501](./Challenge_C501.md) : Déployer Kali, DVWA et exploitations XSS (Cross-site scripting)

[Challenge C502](./Challenge_C502.md) : Exploitations de failles LFI (Local File Inclusion)

[Challenge C503](./Challenge_C503.md) : Attaque Man in the Middle et scans Nmap

---

[Homelab Adguard](./Homelab_AdGuard.md) : Déployer AdGuard Home, serveur DHCP et DNS sinkhole.

[Homelab Proxmox VE](./Homelab_Proxmox.md) : Déployer Proxmox Virtual Environment, hyperviseur de type I pour héberger des machines virtuelles et containers.

[Homelab Checkmk](./Homelab_Checkmk.md) : Déployer Checkmk (Raw Edition), supervision d'infrastructure.

[Homelab pfSense](./Homelab_pfSense.md) : Déploiement d'un Routeur/Pare-feu pfSense sous Proxmox.

[Homelab Scanopy](./Homelab_Scanopy.md) : Déploiement d'un outil de cartographie réseau distribuée et IPAM dynamique.

---

[TP Packet Tracer](../ressources/TP_CiscoPacketTracer2SN.pdf)
