FROM node:20-alpine
WORKDIR /app
COPY --chown=node:node portauth-engine/package.json portauth-engine/package-lock.json ./
RUN npm ci --omit=dev
COPY --chown=node:node portauth-engine/src/ ./src/
COPY --chown=node:node portauth-db/migrations/ /portauth-db/migrations/
EXPOSE 3200
USER node
CMD ["node", "src/index.js"]
