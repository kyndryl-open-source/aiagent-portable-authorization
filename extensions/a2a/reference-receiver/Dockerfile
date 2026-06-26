#
# Copyright contributors to the aiagent-portable-authorization project.
# SPDX-License-Identifier: MIT
#
# Local reference image for the A2A HTTP receiver. Production users should
# harden the base image, pin by digest, and apply their standard image-scanning
# and provenance controls.

FROM node:20-alpine

WORKDIR /app

COPY extensions/a2a/package.json extensions/a2a/package-lock.json ./extensions/a2a/
COPY extensions/a2a/sdk ./extensions/a2a/sdk
COPY extensions/a2a/receiver-middleware ./extensions/a2a/receiver-middleware
COPY extensions/a2a/reference-receiver ./extensions/a2a/reference-receiver
COPY extensions/a2a/reference-presenter ./extensions/a2a/reference-presenter

WORKDIR /app/extensions/a2a
RUN npm ci --omit=dev

EXPOSE 3300

CMD ["npm", "run", "receiver"]
