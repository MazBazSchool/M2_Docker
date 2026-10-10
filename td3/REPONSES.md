# TD3 — Réponses B à C

> J'ai fait les tests sur Docker 29.4.0 avec OrbStack.

## B1

```bash
cp .env.example .env
docker compose config
docker compose up -d
docker compose logs api
```

Avec `docker compose config`, j'ai confirmé les trois services, le réseau `td3_default`, le volume `td3_postgres-data` et `POSTGRES_PASSWORD=change-me` pris depuis mon `.env` local. Je versionne seulement `.env.example`.

## B2

Au premier démarrage, j'ai eu trois `ECONNREFUSED` vers Postgres (`192.168.156.2:5432`), puis mon API a été relancée grâce à `restart: on-failure` et a fini par répondre. Avec `depends_on`, j'impose seulement l'ordre de création, pas la disponibilité réelle. C'est donc un contournement ; la vraie solution est d'attendre les healthchecks au TD4.

## B3

```bash
curl http://localhost:3000/
curl http://localhost:3000/
docker compose down
docker compose up -d
curl http://localhost:3000/
docker compose down -v
docker compose up -d
curl http://localhost:3000/
```

Après `down` puis `up`, ma réponse est passée de `hitsInRedis: 1, visitsInPostgres: 3` à `hitsInRedis: 1, visitsInPostgres: 4` : j'ai gardé ma donnée Postgres avec le volume, mais Redis a recommencé. Après `down -v` puis `up`, mes deux valeurs sont revenues à 1. Pour conserver aussi Redis, j'ajouterais `redis-data:/data` et `redis-server --appendonly yes`.

## B4

```bash
docker compose exec api getent hosts db
docker compose up -d --force-recreate db
docker compose exec api getent hosts db
```

Avant ma recréation, `getent hosts db` me donnait `192.168.156.2`. Après, mon API a redémarré pendant quelques secondes puis a repris ; je garde le nom `db` grâce au DNS Compose. Dans ma config, je dois donc toujours utiliser le nom, jamais une IP de conteneur.

## C1

```bash
docker compose up -d --build
docker compose watch
# Dans un autre terminal : modifier app/src/server.js puis
curl http://localhost:3000/health
```

Avec mon fichier d'override, je passe bien la cible à `dev` : `docker compose config` m'affiche les deux règles `develop.watch`. Après avoir ajouté temporairement `mode: "watch"` à `/health`, j'ai eu `{"status":"UP","mode":"watch"}` sans rebuild manuel. Si je modifie `package.json`, je dois rebuild pour installer les nouvelles dépendances. Ma base est accessible sur `${DB_PORT:-5432}` pour mon client SQL local.

## C2

```bash
docker compose -f compose.yaml up -d --build
```

Si j'oublie `--build`, Docker peut réutiliser mon image construite avec la cible `dev`, même si ma configuration demande `prod`. Avec `--build`, je reconstruis bien l'image de production. Avec un seul fichier, je ne publie pas le port de la base, car il vient de mon override de développement.
