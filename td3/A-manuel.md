# TD3 — Partie A, lancement manuel

> J'ai fait les essais avec ces noms et ce mot de passe, pour garder la même config entre les trois conteneurs.

## Premier essai et problème réseau

```bash
docker build --target prod -t td3-api:local ./app
docker run -d --name db -e POSTGRES_DB=visites -e POSTGRES_USER=visites_user -e POSTGRES_PASSWORD=change-me -v postgres-data:/var/lib/postgresql postgres:18-alpine
docker run -d --name cache redis:8-alpine
docker run -d --name api -p 3000:3000 -e DB_HOST=db -e DB_PORT=5432 -e DB_NAME=visites -e DB_USER=visites_user -e DB_PASSWORD=change-me -e REDIS_URL=redis://cache:6379 td3-api:local
docker logs api
```

Mon API a terminé avec le code 1. J'ai eu exactement `Error: getaddrinfo ENOTFOUND td3-manual-db` dans les logs : sur le réseau `bridge` par défaut, je ne peux pas résoudre les noms automatiquement.

## Correction avec un réseau dédié

```bash
docker rm -f api db cache
docker network create visites-net
docker run -d --name db --network visites-net -e POSTGRES_DB=visites -e POSTGRES_USER=visites_user -e POSTGRES_PASSWORD=change-me -v postgres-data:/var/lib/postgresql postgres:18-alpine
docker logs -f db
docker run -d --name cache --network visites-net redis:8-alpine
docker run -d --name api --network visites-net -p 3000:3000 -e DB_HOST=db -e DB_PORT=5432 -e DB_NAME=visites -e DB_USER=visites_user -e DB_PASSWORD=change-me -e REDIS_URL=redis://cache:6379 td3-api:local
curl http://localhost:3000/
```

Avec mon réseau utilisateur `td3-manual-net`, j'ai un DNS Docker : mon API a répondu trois fois, avec `hitsInRedis` et `visitsInPostgres` à 1, 2 puis 3.

## Persistance

```bash
curl http://localhost:3000/
curl http://localhost:3000/
docker rm -f api db cache
# Recréer les trois conteneurs avec les mêmes commandes et le même volume.
```

Après ma recréation, j'ai eu `hitsInRedis: 1` et `visitsInPostgres: 4`. Mon compteur Postgres a donc survécu grâce au volume nommé, alors que Redis a été remis à zéro car je ne lui avais pas mis de volume.
