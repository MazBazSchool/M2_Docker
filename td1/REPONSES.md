# TD1 — Réponses

> J'ai fait les tests sur Docker 29.4.0 avec OrbStack.

## Partie A

### A1

```bash
docker run hello-world
docker run hello-world
```

Au premier lancement, j'ai vu Docker télécharger l'image. Au deuxième, il l'a réutilisée depuis le cache local. Dans les deux cas, j'ai créé un nouveau conteneur qui affiche le message puis termine.

### A2

```bash
docker run -d --name web1 -p 8080:80 nginx:1.29-alpine
docker run -d --name web2 -p 8081:80 nginx:1.29-alpine
```

J'ai deux conteneurs avec leur propre réseau, donc chacun peut écouter sur son port interne `80`. Le conflit concerne seulement le port de ma machine : `8080` est associé à `web1`, `8081` à `web2`. Si je mets `-p 8080:80` pour les deux, Docker refuse le second lancement car le port hôte est déjà utilisé.

### A3

```bash
docker logs web1
docker logs -f web1
```

Après plusieurs visites de `http://localhost:8080`, j'ai retrouvé les logs Nginx. Je les vois parce que le processus Nginx écrit sur sa sortie standard, que je récupère avec `docker logs`.

### A4

```bash
docker exec -it web1 sh
echo 'Mazigh' > /usr/share/nginx/html/index.html
exit
docker rm -f web1
docker run -d --name web1 -p 8080:80 nginx:1.29-alpine
```

Mon prénom apparaît tant que ce conteneur existe. Après l'avoir supprimé puis recréé, j'ai retrouvé la page par défaut : ma modification était dans la couche écrivable du conteneur supprimé. Je peux utiliser `docker exec` pour diagnostiquer, mais pas pour déployer une modification durable ; je dois la mettre dans une image ou dans un volume.

## Partie B

### B1

```bash
docker run --rm -e PRENOM=Mazigh alpine printenv PRENOM
docker run --rm alpine printenv PRENOM
echo $PRENOM
```

Avec `-e`, j'obtiens `Mazigh` avec `printenv`. Sans `-e`, je n'obtiens rien et la commande termine en erreur car je n'ai pas la variable dans le conteneur. Je ne l'ai pas non plus sur mon poste sauf si je l'ai définie moi-même. Je la passe dans l'environnement du processus du conteneur.

### B2

```bash
docker run --rm -it alpine sh
apk add curl
curl --version
exit
docker run --rm -it alpine sh
curl --version
```

Dans le premier conteneur, j'ai installé `curl` et il fonctionne. Après `exit`, `--rm` détruit ce conteneur. Dans le nouveau conteneur, je n'ai donc plus `curl`. Mon installation a modifié le conteneur, pas l'image `alpine`. Pour la rendre durable, je devrais l'ajouter dans un Dockerfile.

## Partie C

### C1

```bash
docker pull node:24
docker pull node:24-slim
docker pull node:24-alpine
docker image ls node
docker run --rm node:24 sh -c 'ls /usr/bin | wc -l'
docker run --rm node:24 which gcc git curl
```

| Image | Taille | Commandes / outils |
|---|---:|---|
| `node:24` | 1,14 Go | 664 commandes dans `/usr/bin` ; `gcc`, `git` et `curl` présents. |
| `node:24-slim` | 249 Mo | 273 commandes dans `/usr/bin` ; `gcc`, `git` et `curl` absents. |
| `node:24-alpine` | 166 Mo | 143 commandes dans `/usr/bin` ; `gcc`, `git` et `curl` absents. |

J'ai bien Node 24 dans les trois images. La plus grosse contient surtout des paquets système, bibliothèques et outils de développement en plus. Je n'en ai pas besoin pour faire tourner une API déjà construite, mais ils peuvent être utiles au build ou au dépannage.

### C2

```bash
docker image history node:24-alpine
```

J'ai trouvé 9 couches dans l'historique. La plus lourde faisait 152 Mo et correspondait à l'installation de Node ; la couche Alpine de base faisait 8,66 Mo.

### C3

```bash
docker image inspect nginx:1.29-alpine
```

Dans les métadonnées, j'ai trouvé `Cmd` qui lance Nginx en premier plan, généralement `nginx -g 'daemon off;'`. J'ai aussi `ExposedPorts` à `80/tcp`. C'est cohérent avec A2 : `-p 8080:80` relie le port 8080 de mon poste au port 80 annoncé dans le conteneur.

## Partie D

### D1

```bash
docker run -d alpine
docker ps -a
```

J'ai vu le conteneur dans `docker ps -a` avec un état arrêté, mais pas dans `docker ps`. L'image Alpine lance un shell par défaut ; sans terminal interactif ni commande à garder en vie, le processus se termine immédiatement. Je dois donner une commande longue, par exemple `alpine sleep 1000`, ou lancer un shell avec `-it`.

### D2

```bash
docker run -d -p 9082:8080 nginx:1.29-alpine
```

Je vois que le conteneur reste démarré mais que Nginx n'écoute pas sur son port interne `8080` : il écoute sur `80`. Je dois publier le bon port : `-p 9082:80`.

### D3

```bash
docker run -d --name dormeur alpine sleep 1000
time docker stop dormeur
docker ps -a --filter name=dormeur
```

Sans `--init`, j'ai attendu 10,223 s et mon conteneur a fini en 137. J'ai vu Docker envoyer `SIGTERM`, attendre puis envoyer `SIGKILL`. Avec `--init`, j'ai attendu 0,141 s et j'ai eu le code 143 : le petit init a relayé `SIGTERM` à `sleep`.

### D4

```bash
docker run --name gourmand --memory 50m node:24-alpine \
  node -e "const a=[]; while(true) a.push(new Array(1e6).fill(1))"
docker inspect gourmand
```

J'ai obtenu `ExitCode=137`, `OOMKilled=true` et une limite relevée à `52428800` octets. Le noyau applique la limite du cgroup puis son OOM killer termine le processus qui dépasse la mémoire disponible.

## Partie E

```bash
docker system df
docker container prune
docker system df
```

Avant les TD, j'avais 61,27 Go d'images avec `docker system df`. Après mes tests et mon nettoyage ciblé, j'avais 63,13 Go (j'ai gardé les images de base téléchargées et le cache BuildKit). J'ai supprimé uniquement mes conteneurs TD nommés `td1-*` au lieu de lancer `docker container prune`, car ma machine contient aussi des conteneurs arrêtés d'autres projets. J'ai aussi supprimé mes images TD taguées à la fin.
