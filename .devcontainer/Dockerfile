# FROM mcr.microsoft.com/devcontainers/javascript-node:18-bullseye
FROM node:23-alpine

RUN apk update && \
    apk add --no-cache git bash openssh-client
    
RUN npm init --yes && \
    npm install -g zenn-cli@latest

# VOLUME [ "/data" ]
