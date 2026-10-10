# Installation complète et compilation : TypeScript n'est utile qu'ici.
FROM node:24 AS build
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY tsconfig.json ./
COPY src ./src
RUN npm run build

# Dépendances strictement nécessaires à l'exécution.
FROM node:24-alpine AS production-dependencies
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

# Image finale légère, sans le compilateur TypeScript.
FROM node:24-alpine
WORKDIR /app
ENV NODE_ENV=production

COPY --from=production-dependencies --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist

USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
