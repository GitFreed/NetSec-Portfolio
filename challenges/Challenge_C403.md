# Challenge C403 11/03/2026

## 🧑‍🏫 Pitch de l’exercice : 🐝 Déployer GLPI sur un cluster avec Portainer 🏗️

![swarm](/images/2026-03-11-11-25-54.png)

Challenge : <https://github.com/O-clock-Aldebaran/SC04E03-Deployer-GLPI-sur-Docker-Swarm-GitFreed>

[Cours C403.](/RESUME.md#-c403-orchestration-avec-docker-swarm-et-portainer)

> 📚 **Ressources** :
>
> - Docker Swarm Docs : <https://docs.docker.com/engine/swarm/>
> - Docker Swarm Cheatsheet : <https://cheatography.com/gauravpandey44/cheat-sheets/docker-swarm/>
> - Install Portainer CE with Docker Swarm : <https://docs.portainer.io/start/install-ce/server/swarm/linux>

---

## 🗂 Contexte

Hier soir, vous avez déployé GLPI avec Docker Compose sur une seule machine. C'est bien, mais pas suffisant pour une infrastructure de production.

Votre responsable veut maintenant **haute disponibilité** : si un serveur tombe, l'application doit continuer à fonctionner. Pour ça, on va passer à **Docker Swarm** — le mode cluster intégré à Docker — et déployer GLPI en plusieurs **replicas** gérés via **Portainer**.

> 💡 **Docker Compose vs Docker Swarm**
>
> - Compose → un seul hôte, idéal pour le développement
> - Swarm → plusieurs hôtes, orchestration, haute disponibilité
> - La bonne nouvelle : la syntaxe reste très proche, on réutilise le `compose.yaml` que vous avez créer lors de votre challenge !

---

## 🎯 Objectifs

À la fin de cet exercice, vous aurez :

- Initialisé un cluster Docker Swarm
- Déployé une stack via l'interface Portainer
- Configuré GLPI pour tourner en **2 ou 3 replicas**
- Observé le comportement du load balancer Ingress de Swarm
- Compris pourquoi multiplier les replicas d'une BDD pose problème

---

## 📋 Contraintes & Règles du jeu

> ⚠️ **Important**
>
> ✓ Repartir du `compose.yaml` corrigé ce matin comme base  
> ✓ Utiliser **Portainer** pour déployer la stack (pas la CLI dans un premier temps)  
> ✓ Tester l'accès à GLPI depuis le navigateur avant de passer à l'étape suivante  
> ✗ Ne pas chercher à tout faire en CLI — Portainer est là pour ça  

---

## 🔍 Commandes utiles

| Action | Commande |
| --- | --- |
| Voir les services de la stack | `docker service ls` |
| Voir les replicas d'un service | `docker service ps glpi-swarm_glpi` |
| Logs d'un service | `docker service logs -f glpi-swarm_glpi` |
| Scaler un service | `docker service scale glpi-swarm_glpi=5` |
| Mettre à jour la stack | Portainer → Stack → Editor → Update |
| Supprimer la stack | `docker stack rm glpi-swarm` |
| Quitter Swarm | `docker swarm leave --force` |
| Lister les nœuds | `docker node ls` |

---

## Étape 1 — Initialiser Docker Swarm 🐝

```sh
docker swarm init --advertise-addr 10.0.0.30
```

![swarmanager](/images/2026-03-11-10-52-04.png)

![workerjoin](/images/2026-03-11-10-52-58.png)

```sh
docker node ls
```

![node](/images/2026-03-11-10-54-37.png)

```sh
docker node promote/demote xxxxx
```

![manager](/images/2026-03-11-10-59-56.png)

![demote](/images/2026-03-11-11-03-05.png)

---

## Étape 2 — Déploiement Portainer 🏗️

```sh
curl -L https://downloads.portainer.io/ce-lts/portainer-agent-stack.yml -o portainer-agent-stack.yml

docker stack deploy -c portainer-agent-stack.yml portainer

docker service ls
```

On peut voir le nombre de replicas (les 3 agents) et les ports

![replicas](/images/2026-03-11-11-23-31.png)

On peut se connecter sur l'interface web Portainer du Leader via le port **9443** en **https**

<https://10.0.0.30:9443/>

Si *Your Portainer instance timed out for security purposes* il faut

```sh
docker stack rm portainer
docker stack deploy -c portainer-agent-stack.yml portainer
```

![portainer](/images/2026-03-11-11-46-55.png)

On retrouve notre environnement connecté avec tous les détails

![environment](/images/2026-03-11-12-05-39.png)

Son Dashboard

![dash](/images/2026-03-11-12-06-48.png)

Et le détail du Cluster dans le Swarm

![cluster](/images/2026-03-11-12-07-26.png)

Si on veut mettre en pause un agent : `sudo docker node update --availability pause docker3`

![pause](/images/2026-03-11-13-31-26.png)

Il ne sera pas réutilisé pour créer de nouveaux containers

Créer des répliques :

```sh
docker service create nginx

docker service create --replicas 10 nginx
```

![replicas](/images/2026-03-11-13-36-11.png)

Pour que l'agent redevienne actif : `docker node update --availability active docker3`

Pour vider un noeud par exemple le 2 : `docker node update --availability drain docker2`

Tout est passé du 2 au 3 en instantané.

![drain](/images/2026-03-11-13-47-30.png)

Pour supprimer toutes les replicas sauf une : `docker service update --replicas 1 <ID>`

Pour créer un service avec 3 répliques, avec nom + choix du port :

```sh
docker service create --name web2 --replicas 3 nginx && publish 80:80 nginx
```

Pour s'assurer qu'un service Docker Swarm s'exécute uniquement sur des nœuds workers (et non sur les nœuds managers) : `--constraint node.role==worker`

Pour rolling update en version 1.25 les images nginx :

```sh
docker service update --image nginx:1.25 \
--update-delay 10s \
--update-parallelism 2 \
web
```

Ajoute 3 images up en 1.25 et laisse 3 en latest pour pouvoir rollback

![update](/images/2026-03-11-14-23-55.png)

Pour rollback : `docker service update --rollback web`

Pour supprimer tout un service : `docker service rm <ID>`

On peut également agir sur les services directement via l'interface web : nombre de Répliques, Updates, Rollback, Delete etc

![services](/images/2026-03-11-14-30-18.png)

Pour backup la config il faut sauvegarder le `/var/lib/docker/swarm/` et le `/var/lib/docker/volumes`, ou directement `/var/lib/docker/`

---

## Étape 3 — Adapter le `compose.yaml` pour Swarm

Le format est presque identique à Docker Compose, mais Swarm utilise la clé **`deploy`** pour configurer les replicas et la politique de redémarrage.

Voici les modifications à apporter à votre fichier :

- restart: n'existe pas en mode Swarm → remplacé par deploy.restart_policy
- deploy replicas pour le nombre d'instances GLPI, 1 seul pour la BDD

### Ajout de la configuration pour Swarm

```yaml
services:
  # ==========================================================
  # 🗄️ ÉTAPE 1 : Amorçage de la base de données
  # (À déployer en premier. Attendre ~30 secondes après le déploiement)
  # ==========================================================
  db:
    image: mariadb:10.11
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - glpi-net
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager

  # ==========================================================
  # 🖥️ ÉTAPE 2 : Déploiement Applicatif en Haute Disponibilité
  # (Décommenter ce bloc et faire "Update the stack" une fois la DB prête)
  # ==========================================================
  # glpi:
  #   image: glpi/glpi:latest
  #   ports:
        #- target: 80
        #     published: 8080
        #     protocol: tcp
        #     mode: host
  #     # Etape 3 : Effacer target et repasser en port 80
  #     #- "8080:80"
  #   environment:
  #     TIMEZONE: 'Europe/Paris'
  #     MARIADB_HOST: db
  #     MARIADB_DATABASE: ${MYSQL_DATABASE}
  #     MARIADB_USER: ${MYSQL_USER}
  #     MARIADB_PASSWORD: ${MYSQL_PASSWORD}
  #   volumes:
  #     - glpi_data:/var/www/html
  #     - glpi_config:/var/glpi
  #   networks:
  #     - glpi-net
  #   deploy:
  #     replicas: 1 # 3 instances à l'étape 3
  #     placement:
  #       constraints:
  #         - node.role == manager

  # 🛠️ Service Bonus : Adminer (À décommenter également à l'étape 2)
  # adminer:
  #   image: adminer:latest
  #   ports:
  #     - "8081:8080"
  #   networks:
  #     - glpi-net
  #   deploy:
  #     replicas: 1

# ==========================================================
# 💾 RÉSEAU ET STOCKAGE (Déclarations globales)
# ==========================================================
volumes:
  db_data:
  glpi_data:
  glpi_config:

networks:
  glpi-net:
    driver: overlay
```

---

## Étape 4 — Déployer la stack via Portainer

Dans Portainer :

1. **Stacks → Add stack**
2. Donnez un nom au stack (ex: `glpi-swarm`)
3. Coller le `compose.yaml` adapté dans l'éditeur (avec seulement MadiaDB, pas GLPI ni Adminer)
4. Renseigner les variables du `.env` ou l'upload directement.
5. Cliquer sur **Deploy the stack**
6. Attendre 30s puis éditer le Stack avec la partie GLPI active, 1 seule replica, ports en mode Host, activer Adminer aussii
7. Update
8. Se connecter à GLPI lancer l'installation, créer la DB
9. Re-éditer le Stack avec la partie GLPI active, 3 replicas, ports en mode 8080:80
10. Update

![stack](/images/2026-03-11-15-51-28.png)

![stackOK](/images/2026-03-11-15-52-44.png)

Suivez le déploiement dans **Containers** et attendez que tous les replicas soient `running`.

![containers](/images/2026-03-11-16-04-46.png)

---

## Étape 5 — Vérifier le déploiement

```bash
# Lister les services de la stack
docker service ls

# Voir les replicas du service GLPI
docker service ps glpi-swarm_glpi
```

![service](/images/2026-03-11-16-02-40.png)

![slpiswarm](/images/2026-03-11-16-09-32.png)

Accéder à GLPI depuis le navigateur sur `http://10.0.0.30:8080/` — l'installation initiale ne doit se faire qu'**une seule fois** grâce aux volumes.

![glpi](/images/2026-03-11-17-47-59.png)

---

## Étape 6 — Observer le load balancer

Docker Swarm intègre un **load balancer en mode Ingress** : chaque requête est automatiquement routée vers l'un des replicas disponibles.

Ouvrez un terminal et observez sur quel replica atterrissent vos requêtes :

```bash
# Voir les logs de chaque replica en temps réel
docker service logs -f glpi-swarm_glpi
```

Naviguez dans GLPI et rechargez plusieurs fois la page : vous pouvez voir les requêtes distribuées entre les replicas dans les logs.

![load](/images/2026-03-11-20-03-06.png)

---

## Étape 7 — Simuler une panne

Trouvez l'ID d'un des conteneurs GLPI :

```bash
docker ps | grep glpi
```

Supprimez-le brutalement :

```bash
docker rm -f <container_id>
```

Observez : Swarm doit **automatiquement recréer** un nouveau replica pour maintenir le nombre demandé. Vérifiez avec :

```bash
docker service ps glpi-swarm_glpi
```

GLPI doit rester accessible pendant toute l'opération ✅

Sur l'interface web Portainer on voit le container disparaitre et un nouveau prendre sa place.

![swarm](/images/2026-03-11-20-08-38.png)

![containers](/images/2026-03-11-20-09-41.png)

## ⭐ Bonus — Résoudre le problème de la BDD

> 🚨 **Bonus complexe** — réservé aux plus avancés !

Le problème de fond : **un service stateful (base de données) n'est pas fait pour être scalé horizontalement** sans mécanisme de réplication.

### Pistes de solution

- **Option A — Galera Cluster (réplication multi-master MariaDB)**

Utiliser l'image `bitnami/mariadb-galera` qui intègre la réplication synchrone entre les nœuds :

```yaml
db:
  image: bitnami/mariadb-galera
  environment:
    MARIADB_GALERA_CLUSTER_NAME: glpi_cluster
    MARIADB_GALERA_CLUSTER_ADDRESS: gcomm://db
    MARIADB_ROOT_PASSWORD: rootpass
    MARIADB_DATABASE: glpi
    MARIADB_USER: glpi
    MARIADB_PASSWORD: glpipass
  deploy:
    replicas: 3
```

- **Option B — Contraindre la BDD à un seul nœud (solution simple)**

Forcer tous les replicas GLPI à utiliser **une seule instance** de BDD en fixant le service `db` sur un nœud précis :

```yaml
db:
  deploy:
    replicas: 1
    placement:
      constraints:
        - node.role == manager
```

- **Option C — Externaliser la BDD (solution pro)**

Ne pas mettre la BDD dans Swarm du tout, et utiliser une BDD externe (RDS, Managed Database...) accessible par tous les replicas GLPI.
