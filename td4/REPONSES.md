# TD4 — Réponses

> J'ai fait les tests sur Docker 29.4.0 avec OrbStack. Le port 8000 était déjà pris par un autre projet, donc j'ai mis `API_PORT=8010` dans mon `.env` local.

## A1 — Démarrage fiable

```bash
cp .env.example .env
docker compose down -v
docker compose up -d --build
docker compose logs api
docker compose ps
```

J'ai mis un healthcheck sur chaque service : `pg_isready` pour Postgres, `redis-cli ping` pour Redis et `wget` pour l'API. J'ai `wget` dans l'image Alpine via BusyBox. Compose m'a affiché `db Healthy`, `cache Healthy`, puis seulement le démarrage de mon API ; je n'ai donc plus besoin de `restart: on-failure`.

Dans `pg_isready -U $${POSTGRES_USER}`, je mets un double dollar pour empêcher Compose de remplacer la variable sur mon poste. Mon conteneur reçoit donc `${POSTGRES_USER}` et son shell l'interprète avec sa propre valeur.

## A2 — Trois fichiers Compose

```bash
docker compose -f compose.yaml -f compose.prod.yaml config
```

Dans `compose.yaml`, je garde seulement la base commune : pas de port publié ni de mot de passe. Mon override automatique sert au développement et `compose.prod.yaml` ajoute ma config de production. Je ne publie pas le port de la base en prod : seule mon API doit être accessible depuis l'extérieur ; je garde Postgres sur le réseau interne Docker.

## A3 — Production et persistance

```bash
docker compose -f compose.yaml -f compose.prod.yaml up -d --build
curl http://localhost:8000/
curl http://localhost:8000/
docker compose -f compose.yaml -f compose.prod.yaml down
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

Après `down` puis `up`, j'ai eu `hitsInRedis: 1, visitsInPostgres: 2` : j'ai gardé la visite précédente dans Postgres et Redis a recommencé. Je ne peux pas joindre ma base depuis ma machine car je ne publie aucun port, ce qui est voulu en production.

## A4 — Arrêt propre

```bash
time docker compose -f compose.yaml -f compose.prod.yaml stop api
docker compose -f compose.yaml -f compose.prod.yaml ps -a
```

L'arrêt a pris 0,304 s et mon conteneur est sorti avec le code `0`. Mon application traite `SIGTERM`, ferme le serveur HTTP et les connexions PostgreSQL/Redis. Après un `docker stop`, `restart: unless-stopped` laisse volontairement mon conteneur arrêté ; en cas de crash, Docker le redémarre.

## A5 — Bonus : secret Compose

Pour ne plus mettre mon mot de passe dans les variables d'environnement en prod, je créerais `secrets/db_password.txt` hors Git puis :

```yaml
secrets:
  db_password:
    file: ./secrets/db_password.txt

services:
  api:
    secrets: [db_password]
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password
  db:
    secrets: [db_password]
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
```

Avant, je vois `DB_PASSWORD` dans l'environnement du conteneur avec `docker inspect`. Avec cette version, je ne retrouve plus la valeur dans l'environnement : mon application lit le fichier monté dans `/run/secrets`.

## B1 — Secret de build

```bash
# Démonstration à ne pas conserver dans le Dockerfile final :
docker build --build-arg TOKEN="un-faux-jeton" -t td4:arg ./app
docker history --no-trunc td4:arg

# Version sûre, avec une instruction temporaire BuildKit :
docker build --secret id=token,src=token.txt -t td4:secret ./app
docker history --no-trunc td4:secret
```

Avec ma variante `ARG`, j'ai eu l'avertissement BuildKit `SecretsUsedInArgOrEnv`. Dans `docker history --no-trunc td4:arg-demo`, je retrouvais le jeton factice dans `ARG TOKEN=...` et les lignes `RUN |1 TOKEN=...`. Quand je fais le même grep sur mon image avec secret, je ne trouve rien : `RUN --mount=type=secret` ne laisse pas le jeton dans une couche. Je ne garde pas cette démonstration dans mon Dockerfile final car mon API n'a pas besoin de jeton pour être construite.

## B2 — Durcissement

```bash
docker compose -f compose.yaml -f compose.prod.yaml exec api sh -c 'echo test > /app/test'
docker compose -f compose.yaml -f compose.prod.yaml exec api grep CapEff /proc/1/status
docker inspect "$(docker compose -f compose.yaml -f compose.prod.yaml ps -q api)"
```

Quand j'écris dans `/app`, j'ai `Read-only file system`, mais je peux écrire dans `/tmp`. J'ai `CapEff` à `0000000000000000`. Avec l'inspection, je confirme `ReadonlyRootfs=true`, `CapDrop=["ALL"]`, `Memory=268435456` et un tmpfs `/tmp`. Si je dois garder un fichier, je le mets dans un volume ou dans un stockage externe.

## B3 — Scan bloquant

```bash
alias trivy='docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache aquasec/trivy:0.74.0'
trivy image --severity CRITICAL --exit-code 1 td4-api ; echo "code de sortie : $?"
# Test temporaire : npm install lodash@4.17.4, rebuild puis scan ; supprimer ensuite la dépendance.
```

Mon scan de `td4-api` a trouvé 0 vulnérabilité CRITICAL et s'est terminé avec le code 0. En ajoutant temporairement `lodash@4.17.4`, j'ai trouvé `CVE-2019-10744` avec Trivy (CRITICAL, correctif `4.17.12`) et le scan a terminé avec le code 1. J'ai ensuite remis mes fichiers `package.json` et `package-lock.json` comme avant.

## B4 — Pipeline

Dans ma pipeline à la racine, je suis l'ordre `build → scan → test → publish`. Je passe l'image entre les jobs dans `image.tar`, puis je la charge avec `docker load`. Mon job `test` crée son réseau et ses conteneurs sans monter le dépôt. Je limite `publish` à la branche par défaut et aux étapes précédentes vertes, donc ma pipeline rouge ne peut pas publier.

J'ai vérifié que le YAML passe sans erreur, et j'ai fait l'équivalent de build/scan/test en local. Je n'ai pas lancé la pipeline GitLab elle-même car il faudrait que je pousse sur un dépôt GitLab avec un registry et des identifiants CI.

## B5 — Déploiement depuis le registry

```bash
docker login <registry>
API_IMAGE=<registry>/<chemin>/visites-api:<commit> docker compose -f compose.yaml -f compose.prod.yaml up -d --no-build
```

Avec `--no-build`, je lance exactement l'image validée par la CI et pas une image locale reconstruite. Avec un tag qui contient le commit, je peux savoir quelle image je lance ; `latest` peut changer au fil du temps. Je n'ai pas exécuté cette commande faute d'accès au registry, mais j'ai prévu la variable `API_IMAGE` dans `compose.prod.yaml`.
