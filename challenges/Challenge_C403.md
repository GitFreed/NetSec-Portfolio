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
