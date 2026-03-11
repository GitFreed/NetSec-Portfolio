# Challenge C403 11/03/2026

## 🧑‍🏫 Pitch de l’exercice : 🐝 Déployer GLPI sur un cluster avec Portainer 🏗️

![swarm](/images/2026-03-11-11-25-54.png)

Challenge : <https://github.com/O-clock-Aldebaran/SC04E03-Deployer-GLPI-sur-Docker-Swarm-GitFreed>

[Cours C403.](/RESUME.md#-c403-docker-swarm--portainer)

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
|---|---|
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

---

## Étape 4 — Déployer la stack via Portainer
