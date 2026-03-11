# Challenge C403 11/03/2026

## 🧑‍🏫 Pitch de l’exercice : 🐝 Déployer Docker Swarm & Portainer 🏗️

![swarm](/images/2026-03-11-11-25-54.png)

Challenge : <.>

[Cours C403.](/RESUME.md#-c403-docker-swarm)

> 📚 **Ressources** :
>
> - Docker Swarm Docs : <https://docs.docker.com/engine/swarm/>
> - Docker Swarm Cheatsheet : <https://cheatography.com/gauravpandey44/cheat-sheets/docker-swarm/>
> - Install Portainer CE with Docker Swarm : <https://docs.portainer.io/start/install-ce/server/swarm/linux>

---

## Swarm 🐝

```sh
docker swarm init --advertise-addr <IP-DU-MANAGER>
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

## Portainer 🏗️

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
