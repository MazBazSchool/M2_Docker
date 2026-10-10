# TD2 — Mesures et réponses

> J'ai fait les builds sur Docker 29.4.0 avec OrbStack. Pour les builds « à froid », j'ai utilisé `--no-cache` ; les images de base étaient déjà téléchargées.

| Version | Taille de l'image | Build à froid | Rebuild après une ligne de code | `.env` dans l'image ? | Utilisateur |
|---|---:|---:|---:|---|---|
| v1 | 1,18 Go | 3,818 s | Non mesuré séparément | Oui, `.env` apparaît dans `ls -la /app` | `root` |
| v2 | 1,18 Go | 2,652 s | 1,369 s | Oui | `root` |
| v3 | 1,18 Go | 3,077 s | Non mesuré séparément | Non, `.env` absent de `/app` | `root` |
| v4 | 169 Mo | 3,025 s | 1,491 s | Non, l'image ne contient que `dist/` et `node_modules/` | `node` |

## Commandes prévues

```bash
docker build --no-cache -t td2:v1 -f Dockerfile.v1 .
docker build -t td2:v4 .
docker image ls td2
docker run --rm td2:v4 id
docker run --rm -p 3000:3000 -e MESSAGE='Bonjour' -e APP_VERSION=1.0.0 td2:v4
```

J'ai aussi lancé deux conteneurs de `td2:v4` sur les ports 3001 et 3002. J'ai eu `Bonjour TD2` et `Deuxieme API`, avec deux hostnames différents. Le même tag d'image a donc bien supporté mes deux configurations.

## Q1

Avec ma version naïve, `.env` est copié par `COPY . .` et se retrouve dans l'image. C'est grave parce que je peux partager, envoyer dans un registry ou inspecter une image. Je ne dois pas stocker une clé d'API dans une image.

## Q2

Pour utiliser le cache, je copie d'abord `package.json` et `package-lock.json`, puis je fais `npm ci`. Si je modifie seulement `src/`, Docker réutilise la couche des dépendances. Si je modifie `package.json` ou le lockfile, Docker doit reconstruire cette couche car mes dépendances peuvent changer.

## Q3

J'exclus `.env` pour mes secrets, `node_modules` pour ne pas envoyer les modules installés sur mon poste, `dist` car je le construis dans l'image, `.git` car je n'ai pas besoin de l'historique, et les fichiers macOS/logs. Mon contexte Docker est donc plus petit et je n'envoie pas de fichiers inutiles.

## Q4

Dans mon étape de build, j'ai Node complet, npm, TypeScript, les dépendances de développement et les sources TypeScript. Dans l'image finale, je garde seulement Node Alpine, les dépendances de production et `dist`. Je n'ai plus TypeScript, les types `@types/*` ni les fichiers `src/` dans l'image finale.

## Q5

Pour moi, `MESSAGE` et `APP_VERSION` sont de la configuration, pas du code. En les passant avec `-e`, j'utilise exactement la même image dans plusieurs environnements et je peux modifier un message sans reconstruire, taguer ni distribuer une nouvelle image.
