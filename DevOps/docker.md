# Docker — Référence complète des concepts et commandes

## 1. Images vs Conteneurs

**Image** : snapshot en lecture seule, composé de couches (layers) empilées. Construite à partir d'un Dockerfile.
**Conteneur** : instance en cours d'exécution d'une image, avec une couche writable propre par-dessus.

```bash
docker images                    # liste les images locales
docker image ls -a               # inclut les images intermédiaires
docker image rm <image>          # supprime une image
docker rmi $(docker images -q)   # supprime toutes les images
docker image prune               # supprime les images non utilisées (dangling)
docker image prune -a            # supprime aussi les images non référencées par un conteneur
docker image inspect <image>     # métadonnées détaillées (couches, config, env)
docker history <image>           # historique des couches et leur taille
```

---

## 2. `docker build` — construire une image

```bash
docker build -t nom:tag .
```

| Argument | Rôle |
|---|---|
| `-t, --tag` | Nom et tag de l'image (`monapp:1.0`) |
| `-f, --file` | Chemin du Dockerfile si différent de `./Dockerfile` |
| `--no-cache` | Ignore le cache de build, tout reconstruire |
| `--build-arg KEY=VAL` | Passe une variable au Dockerfile (`ARG`) |
| `--target` | Cible un stage précis dans un build multi-stage |
| `--platform` | Plateforme cible (`linux/amd64`, `linux/arm64`) |
| `--progress` | Format de sortie (`plain`, `auto`, `tty`) |
| `--pull` | Force le pull de la dernière version de l'image de base |
| `-q, --quiet` | Affiche uniquement l'ID de l'image finale |

---

## 3. `docker run` — créer et démarrer un conteneur

```bash
docker run -d -p 8080:80 -v data:/app/data --name monapp --env-file .env monapp:1.0
```

| Argument | Rôle |
|---|---|
| `-d, --detach` | Exécute en arrière-plan |
| `-it` | Mode interactif + pseudo-TTY (shell, debug) |
| `--name` | Nom explicite du conteneur |
| `-p, --publish host:container` | Mappe un port hôte → conteneur |
| `-P` | Publie tous les ports `EXPOSE` sur des ports aléatoires |
| `-v, --volume host:container` | Bind mount ou volume nommé |
| `--mount` | Syntaxe verbeuse équivalente à `-v`, plus explicite |
| `-e, --env KEY=VAL` | Variable d'environnement |
| `--env-file` | Charge les variables depuis un fichier `.env` |
| `--rm` | Supprime le conteneur automatiquement à l'arrêt |
| `--network` | Rattache à un réseau Docker spécifique |
| `--restart` | Politique de redémarrage (`no`, `always`, `on-failure`, `unless-stopped`) |
| `-w, --workdir` | Répertoire de travail dans le conteneur |
| `-u, --user` | UID/utilisateur d'exécution (sécurité, éviter root) |
| `--memory`, `--cpus` | Limites de ressources |
| `--entrypoint` | Écrase l'`ENTRYPOINT` de l'image |
| `-v /var/run/docker.sock:/var/run/docker.sock` | Docker-in-Docker (accès au daemon hôte) |

---

## 4. Gestion des conteneurs

```bash
docker ps                    # conteneurs actifs
docker ps -a                 # tous les conteneurs, y compris arrêtés
docker start/stop/restart <id>
docker rm <id>                # supprime un conteneur arrêté
docker rm -f <id>             # force la suppression (même actif)
docker logs -f <id>           # suit les logs en direct
docker logs --tail 100 <id>   # dernières 100 lignes
docker exec -it <id> sh       # shell interactif dans un conteneur en cours
docker exec <id> <cmd>        # exécute une commande ponctuelle
docker inspect <id>           # config JSON complète (réseau, mounts, env)
docker stats                  # utilisation CPU/mémoire en temps réel
docker cp <id>:/path ./local  # copie fichier conteneur → hôte (et inversement)
docker top <id>               # processus en cours dans le conteneur
```

---

## 5. Volumes — persistance des données

Un conteneur est éphémère : sa couche writable disparaît à sa suppression, sauf données persistées via un volume.

- **Volume nommé** : géré par Docker (`/var/lib/docker/volumes/...`), portable, recommandé pour bases de données.
- **Bind mount** : chemin hôte explicite, utile en dev pour le hot-reload de code.
- **tmpfs** : stockage en mémoire uniquement, non persistant, utile pour données sensibles temporaires.

```bash
docker volume create data
docker volume ls
docker volume inspect data
docker volume rm data
docker volume prune            # supprime les volumes non utilisés

docker run -v data:/app/data ...        # volume nommé
docker run -v $(pwd):/app ...           # bind mount
docker run --tmpfs /app/tmp ...         # tmpfs
```

---

## 6. Réseaux

Docker crée un réseau bridge par défaut ; chaque conteneur y obtient une IP interne. Sur un réseau **utilisateur** (non défaut), les conteneurs se résolvent par leur nom.

```bash
docker network ls
docker network create monreseau
docker network create --driver bridge --subnet 172.20.0.0/16 monreseau
docker network inspect monreseau
docker network connect monreseau <id>
docker network rm monreseau
```

| Driver | Usage |
|---|---|
| `bridge` | Défaut, un seul hôte |
| `host` | Partage directement le réseau de l'hôte, pas d'isolation |
| `none` | Aucun réseau |
| `overlay` | Multi-hôtes (Swarm) |

---

## 7. Dockerfile — instructions essentielles

| Instruction | Rôle |
|---|---|
| `FROM` | Image de base, point de départ du build |
| `WORKDIR` | Définit/crée le répertoire de travail |
| `COPY` | Copie fichiers hôte → image (préférer à `ADD` sauf besoin d'extraction/URL) |
| `ADD` | Comme `COPY`, mais gère aussi les archives et URLs |
| `RUN` | Exécute une commande au moment du build (crée une couche) |
| `ENV` | Variable d'environnement persistante dans l'image |
| `ARG` | Variable disponible uniquement au build (via `--build-arg`) |
| `EXPOSE` | Documente le port utilisé (n'ouvre rien réellement) |
| `CMD` | Commande par défaut, surchargeable au `docker run` |
| `ENTRYPOINT` | Exécutable fixe, `CMD` devient ses arguments par défaut |
| `USER` | Change l'utilisateur d'exécution (sécurité) |
| `VOLUME` | Déclare un point de montage anonyme |
| `HEALTHCHECK` | Commande de vérification de santé périodique |
| `LABEL` | Métadonnées clé-valeur |
| `.dockerignore` | Exclut fichiers du contexte de build (réduit taille/temps) |

**Build multi-stage** — pattern clé pour des images légères :

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN go build -o app .

FROM gcr.io/distroless/static
COPY --from=build /src/app /app
ENTRYPOINT ["/app"]
```

---

## 8. Docker Compose — orchestration multi-conteneurs

```bash
docker compose up            # démarre tous les services
docker compose up -d         # en arrière-plan
docker compose up --build    # force le rebuild des images
docker compose down          # arrête et supprime conteneurs/réseaux
docker compose down -v       # supprime aussi les volumes
docker compose ps
docker compose logs -f <service>
docker compose exec <service> sh
docker compose build
docker compose config        # valide et affiche la config résolue
```

```yaml
services:
  web:
    build: .
    ports:
      - "8080:80"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    depends_on:
      - db
    volumes:
      - .:/app          # bind mount pour le dev
  db:
    image: postgres:16
    volumes:
      - dbdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: secret
volumes:
  dbdata:
```

Points clés : `depends_on` gère l'ordre de démarrage mais pas l'attente de "readiness" (utiliser `healthcheck` pour ça) ; les services communiquent entre eux via leur nom comme hostname sur le réseau créé automatiquement par Compose.

---

## 9. Registre et distribution

```bash
docker login
docker pull nginx:1.27
docker push mondockerhub/monapp:1.0
docker tag monapp:1.0 mondockerhub/monapp:1.0
docker save -o monapp.tar monapp:1.0     # export vers fichier tar
docker load -i monapp.tar                 # import depuis un tar
```

---

## 10. Bonnes pratiques et pièges fréquents

- **Ordre des couches** : copier `package.json`/`requirements.txt` et installer les dépendances *avant* de copier le code source, pour maximiser le cache de build.
- **Ne pas tourner en root** : ajouter une instruction `USER` dédiée.
- **Images de base légères** : `alpine`, `slim`, ou `distroless` pour réduire la surface d'attaque et la taille.
- **Pinner les versions** : éviter `latest`, préférer `node:20.11-slim` pour la reproductibilité.
- **`.dockerignore`** : exclure `node_modules`, `.git`, fichiers de build, pour un contexte de build léger et rapide.
- **Une seule couche pour install + cleanup** : `RUN apt-get update && apt-get install -y x && apt-get clean` sur une seule ligne, car supprimer un fichier dans une couche ultérieure ne réduit pas la taille de l'image.
- **`docker system prune -a --volumes`** : nettoyage complet quand le disque se remplit (images, conteneurs, réseaux, volumes non utilisés).
- **Secrets** : ne jamais mettre de secrets en `ENV`/`ARG` en clair dans le Dockerfile (visibles dans l'historique) — utiliser `docker secret`, un gestionnaire de secrets externe, ou `--mount=type=secret` en BuildKit.

---

## 11. Commandes de diagnostic rapide

```bash
docker version                 # versions client/serveur
docker info                    # config globale du daemon
docker system df               # espace disque utilisé par images/conteneurs/volumes
docker events                  # flux d'événements en temps réel du daemon
```
