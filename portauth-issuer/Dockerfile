FROM node:20-alpine
WORKDIR /app
COPY --chown=node:node portauth-issuer/package.json portauth-issuer/package-lock.json ./
RUN npm ci --omit=dev
COPY --chown=node:node portauth-issuer/src/ ./src/
COPY --chown=node:node portauth-db/migrations/ /portauth-db/migrations/
EXPOSE 3100
USER node
CMD ["node", "src/index.js"]
