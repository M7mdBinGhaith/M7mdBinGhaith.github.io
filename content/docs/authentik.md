---
title: "Authentik Behind Traefik Reverse Proxy"
date: 2025-05-26
weight: 20
categories: ["security", "docker", "authentication"]
tags: ["authentik", "traefik", "docker", "homelab", "access-management", "sso", "reverse-proxy"]
aliases: ["Authentik Traefik Setup", "Local Authentik Deployment"]
draft: false
---

# Authentik Behind Traefik Reverse Proxy

[Traefik Reverse Proxy Setup]({{< ref "traefik" >}}) - Required foundation for this Authentik deployment
## Overview

{{< hint info >}}
This guide walks through setting up Authentik SSO behind a Traefik reverse proxy.
{{< /hint >}}

![Diagram](https://raw.githubusercontent.com/M7mdBinGhaith/homelab/18eea4d899108f13c3865dacda952ae65f7939b3/compose-stacks/authentik/auth.svg)

[Authentik](https://goauthentik.io/) is an open-source Identity Provider that provides secure authentication, authorization, and user management. This setup uses Docker Compose to deploy Authentik with PostgreSQL and Redis, all behind Traefik for secure access.




## Architecture



The deployment consists of:

- **Traefik**: Reverse proxy handling TLS termination and routing
- **PostgreSQL**: Database backend for Authentik
- **Redis**: Cache and message broker
- **Authentik Server**: Main application component
- **Authentik Worker**: Background task processor


## Prerequisites

- Docker and Docker Compose installed
- Traefik already configured as a reverse proxy
- External Docker network named "proxy" created
- NFS share mounted (or modify the volume paths as needed)
- `.env` file with required variables

## Installation

### 1. Create the .env File

Create a `.env` file in the same directory as your docker-compose.yml:

```bash
# Required variables
PG_PASS=your_secure_password
PG_USER=authentik
PG_DB=authentik

# Authentik-specific settings
AUTHENTIK_SECRET_KEY=your_very_secure_random_key
AUTHENTIK_ERROR_REPORTING__ENABLED=false
AUTHENTIK_PORT_HTTP=9000
AUTHENTIK_PORT_HTTPS=9443

# Optional: Email settings
AUTHENTIK_EMAIL__HOST=smtp.example.com
AUTHENTIK_EMAIL__PORT=587
AUTHENTIK_EMAIL__USERNAME=mail_user
AUTHENTIK_EMAIL__PASSWORD=mail_password
AUTHENTIK_EMAIL__USE_TLS=true
AUTHENTIK_EMAIL__FROM=authentik@example.com
```

### 2. Create Directory Structure

Ensure your NFS mount points exist:

```bash
sudo mkdir -p /nfs/apps/authentik/{media,certs,custom-templates}
```

### 3. Deploy with Docker Compose

Save the following as `docker-compose.yml`:

```yaml
services:
  postgresql:
    image: docker.io/library/postgres:12-alpine
    restart: unless-stopped
    healthcheck:
      test:
        - CMD-SHELL
        - pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 5s
    volumes:
      - database:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: ${PG_PASS:?database password required}
      POSTGRES_USER: ${PG_USER:-authentik}
      POSTGRES_DB: ${PG_DB:-authentik}
    env_file:
      - .env
    networks:
      proxy: null
  redis:
    image: docker.io/library/redis:alpine
    command: --save 60 1 --loglevel warning
    restart: unless-stopped
    healthcheck:
      test:
        - CMD-SHELL
        - redis-cli ping | grep PONG
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 3s
    volumes:
      - redis:/data
    networks:
      proxy: null
  server:
    image: ${AUTHENTIK_IMAGE:-ghcr.io/goauthentik/server}:latest
    container_name: authentik_server
    restart: unless-stopped
    command: server
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
    volumes:
      - /nfs/apps/authentik/media:/media
      - /nfs/apps/authentik/custom-templates:/templates
    env_file:
      - .env
    depends_on:
      - postgresql
      - redis
    networks:
      proxy: null
    labels:
      - traefik.enable=true
      - traefik.http.routers.authentik.entrypoints=http
      - traefik.http.routers.authentik.rule=Host(`authentik.local.yourdomain.com`)
      - traefik.http.middlewares.authentik-https-redirect.redirectscheme.scheme=https
      - traefik.http.routers.authentik.middlewares=authentik-https-redirect
      - traefik.http.routers.authentik-secure.entrypoints=https
      - traefik.http.routers.authentik-secure.rule=Host(`authentik.local.yourdomain.com`)
      - traefik.http.routers.authentik-secure.tls=true
      - traefik.http.routers.authentik-secure.service=authentik
      - traefik.http.services.authentik.loadbalancer.server.scheme=https
      - traefik.http.services.authentik.loadbalancer.server.port=9443
      - traefik.docker.network=proxy
  worker:
    image: ${AUTHENTIK_IMAGE:-ghcr.io/goauthentik/server}:latest
    restart: unless-stopped
    command: worker
    environment:
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /nfs/apps/authentik/media:/media
      - /nfs/apps/authentik/certs:/certs
      - /nfs/apps/authentik/custom-templates:/templates
    env_file:
      - .env
    depends_on:
      - postgresql
      - redis
    networks:
      proxy: null
networks:
  proxy:
    external: true
volumes:
  database:
    driver: local
  redis:
    driver: local
```

### 4. Start the Stack

```bash
docker-compose up -d
```

## Traefik Configuration Explained


The Traefik labels in the compose file configure:

1. **HTTP to HTTPS Redirection**: All HTTP traffic is redirected to HTTPS
2. **Host Rule**: Traffic to `authentik.local.yourdomain.com` is routed to Authentik
3. **TLS**: HTTPS is enabled for secure connections
4. **Backend Service**: Authentik server runs on port 9443 with HTTPS

## Initial Setup

1. Once deployed, access Authentik at `https://authentik.local.yourdomain.com`
2. You'll be prompted to create an admin user on first launch
3. Follow the on-screen instructions to complete initial configuration



## Integrating with Other Services

Authentik can be configured as an identity provider for:

- Web applications via OIDC/OAuth2
- LDAP-compatible services
- SAML applications
- Proxied applications via forward authentication

