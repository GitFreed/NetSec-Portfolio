# Challenge C403 10/03/2026

## 🧑‍🏫 Pitch de l’exercice : 🐝 Déployer Docker Swarm

Challenge : <.>

[Cours C403.](/RESUME.md#️-c403)

> 📚 **Ressources** :
>
> - Docker Swarm Docs : <https://docs.docker.com/engine/swarm/>
> - Docker Swarm Cheatsheet : <https://cheatography.com/gauravpandey44/cheat-sheets/docker-swarm/>

---

docker swarm init --advertise-addr <IP-DU-MANAGER>

![swarmanager](/images/2026-03-11-10-52-04.png)

![workerjoin](/images/2026-03-11-10-52-58.png)

docker node ls

![node](/images/2026-03-11-10-54-37.png)

docker node promote/demomte xxxxx

![manager](/images/2026-03-11-10-59-56.png)

![demote](/images/2026-03-11-11-03-05.png)