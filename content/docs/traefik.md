---
title: "Traefik Reverse Proxy"
date: 2025-05-26
weight: 10
categories: ["networking", "docker", "security"]
tags: ["traefik", "docker", "reverse-proxy", "homelab", "access-management", "tls", "cloudflare"]
aliases: ["Traefik Reverse Proxy", "Homelab Traefik Setup"]
draft: false
---

# Traefik Reverse Proxy

## Overview

{{< hint info >}}
This document details the Traefik reverse proxy configuration for homelab services with separate internal and external access paths.
{{< /hint >}}

![Diagram](https://raw.githubusercontent.com/M7mdBinGhaith/homelab/18eea4d899108f13c3865dacda952ae65f7939b3/compose-stacks/traefik/entrypoint.svg)

[Traefik](https://traefik.io/) serves as the central gateway for all web services in the homelab, providing:

- Automatic HTTPS with Cloudflare DNS validation
- Separate entrypoints for internal and external access
- Protected dashboard interface
- Docker service auto-discovery
- Integration with authentication services

## Architecture

{{< hint >}}
The external entrypoint relies on firewall port forwarding from 443 to 444, keeping internal and external traffic separate.
{{< /hint >}}




### Components

- **Traefik Proxy**: Main reverse proxy service
- **Docker Integration**: Service auto-discovery
- **Cloudflare DNS**: Certificate issuance and validation

### Network Design

- **Internal Access**: Standard ports 80/443 for local network
- **External Access**: Ports 81/444 for internet access
- **Security Separation**: Distinct handling of internal vs external entrypoints

## Setup

### Prerequisites

- Docker and Docker Compose
- External Docker network named "proxy" created
- Cloudflare DNS configured for your domain
- Cloudflare API token with Dns zone edit permissions
- Basic understanding of Docker networks and labels

### Installation Steps

#### 1. Directory Structure

```bash
mkdir -p /your/path/traefik/data
touch /your/path/traefik/data/acme.json
chmod 600 /your/path/traefik/data/acme.json
```

#### 2. Cloudflare API Token

```bash
echo "your-cloudflare-api-token" > /your/path/traefik/cf_api_token.txt
```

#### 3. Environment File

```bash
# Generate with: echo $(htpasswd -nb user password)
TRAEFIK_DASHBOARD_CREDENTIALS=user:hashed_password
```

#### 4. Static Configuration

Save this as `/your/path/traefik/data/traefik.yml`:

```yaml
api:
  dashboard: true
  debug: true
entryPoints:
  http:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: https
          scheme: https
  https:
    address: ":443"
  external-http:
    address: ":81"
  external-https:
    address: ":444"
serversTransport:
  insecureSkipVerify: true
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
  file:
    filename: /config.yml
certificatesResolvers:
  cloudflare:
    acme:
      email: example@mail.com
      storage: acme.json
      dnsChallenge:
        provider: cloudflare
        #disablePropagationCheck: true # uncomment this if you have issues pulling certificates through cloudflare
        #delayBeforeCheck: 60s # uncomment along with disablePropagationCheck if needed
        resolvers:
          - "1.1.1.1:53"
          - "1.0.0.1:53"
log:
  level: DEBUG  # Options: ERROR, WARN, INFO, DEBUG
```

#### 5. Dynamic Configuration

Create the file `/your/path/traefik/data/config.yml` with your Authentik middleware and security settings:

```yaml
http:
  middlewares:
    middlewares-authentik: 
      forwardAuth:
        address: "http://server:9000/outpost.goauthentik.io/auth/traefik"
        trustForwardHeader: true
        authResponseHeaders:
          - X-authentik-username
          - X-authentik-groups
          - X-authentik-email
          - X-authentik-name
          - X-authentik-uid
          - X-authentik-jwt
          - X-authentik-meta-jwks
          - X-authentik-meta-outpost
          - X-authentik-meta-provider
          - X-authentik-meta-app
          - X-authentik-meta-version
          - Authorization
    https-redirectscheme:
      redirectScheme:
        scheme: https
        permanent: true

    default-headers:
      headers:
        frameDeny: true
        browserXssFilter: true
        contentTypeNosniff: true
        forceSTSHeader: true
        stsIncludeSubdomains: true
        stsPreload: true
        stsSeconds: 15552000
        customFrameOptionsValue: SAMEORIGIN
        customRequestHeaders:
          X-Forwarded-Proto: https

    default-whitelist:
      ipAllowList:
        sourceRange:
          - "10.0.0.0/8"
          - "192.168.0.0/16"
          - "172.16.0.0/12"

    secured:
      chain:
        middlewares:
          - default-whitelist
          - default-headers
```

#### 6. Docker Compose

Save the following as `docker-compose.yml`:

```yaml
version: "3.8"
services:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - proxy
    ports:
      - 80:80
      - 443:443
      - 81:81
      - 444:444
      # - 443:443/tcp # Uncomment if you want HTTP3
      # - 443:443/udp # Uncomment if you want HTTP3
    environment:
      CF_DNS_API_TOKEN_FILE: /run/secrets/cf_api_token # note using _FILE for docker secrets
      # CF_DNS_API_TOKEN: ${CF_DNS_API_TOKEN} # if using .env
      TRAEFIK_DASHBOARD_CREDENTIALS: ${TRAEFIK_DASHBOARD_CREDENTIALS}
    secrets:
      - cf_api_token
    env_file: .env # use .env
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /your/path/traefik/data/traefik.yml:/traefik.yml:ro
      - /your/path/traefik/data/acme.json:/acme.json
      - /your/path/traefik/data/config.yml:/config.yml:ro
    labels:
      - traefik.enable=true
      - traefik.http.routers.traefik.rule=Host(`traefik-dashboard.local.domain.com`)
      - traefik.http.middlewares.traefik-auth.basicauth.users=${TRAEFIK_DASHBOARD_CREDENTIALS}
      - traefik.http.middlewares.traefik-https-redirect.redirectscheme.scheme=https
      - traefik.http.middlewares.sslheader.headers.customrequestheaders.X-Forwarded-Proto=https
      - traefik.http.routers.traefik.middlewares=traefik-https-redirect
      - traefik.http.routers.traefik-secure.entrypoints=https
      - traefik.http.routers.traefik-secure.rule=Host(`traefik-dashboard.local.domain.com`)
      - traefik.http.routers.traefik-secure.middlewares=traefik-auth
      - traefik.http.routers.traefik-secure.tls=true
      - traefik.http.routers.traefik-secure.tls.certresolver=cloudflare
      - traefik.http.routers.traefik-secure.tls.domains[0].main=local.domain.com
      - traefik.http.routers.traefik-secure.tls.domains[0].sans=*.local.domain.com
      - traefik.http.routers.traefik-secure.tls.domains[1].main=domain.com
      - traefik.http.routers.traefik-secure.tls.domains[1].sans=*.domain.com
      - traefik.http.routers.traefik-secure.service=api@internal
      - traefik.http.middlewares.external-https-redirect.redirectscheme.scheme=https
      - traefik.http.middlewares.external-https-redirect.redirectscheme.port=444
secrets:
  cf_api_token:
    file: /your/path/traefik/cf_api_token.txt
networks:
  proxy:
    external: true
```

#### 7. Deployment

```bash
docker-compose up -d
```

## Configuration

### Entrypoints

{{< hint tip >}}
This setup has distinct entrypoints for internal and external traffic.
{{< /hint >}}

#### Internal Ports

- **http (80)**: Redirects to HTTPS
- **https (443)**: Standard HTTPS for internal network access

#### External Ports

{{< hint tip >}}
Using NAT redirection to map external https port to internal
{{< /hint >}}

- **external-http (81)**: HTTP for external access 
- **external-https (444)**: HTTPS for external access

### Firewall Setup

{{< hint warning >}}
Your firewall should forward external port 443 to internal port 444.
{{< /hint >}}

#### Benefits

- Segregated traffic sources
- Different security rules for internal vs. external
- Selective exposure of services

### Certificate Management

#### Cloudflare Integration

- DNS-01 challenge for domain validation
- Wildcard certificate support
- Automated renewal process

## Service Integration

### Basic Service

To connect a service to Traefik, add these labels:

```yaml
services:
  example-service:
    # ...
    labels:
      - traefik.enable=true
      - traefik.http.routers.myservice.rule=Host(`service.local.domain.com`)
      - traefik.http.routers.myservice.entrypoints=https
      - traefik.http.routers.myservice.tls=true
      - traefik.http.services.myservice.loadbalancer.server.port=8080
```

### External Access

For a service that should be accessible externally:

```yaml
services:
  external-service:
    # ...
    labels:
      # Internal access
      - traefik.enable=true
      - traefik.http.routers.myservice.rule=Host(`service.local.domain.com`)
      - traefik.http.routers.myservice.entrypoints=https
      - traefik.http.routers.myservice.tls=true
      
      # External access
      - traefik.http.routers.myservice-external.rule=Host(`service.domain.com`)
      - traefik.http.routers.myservice-external.entrypoints=external-https
      - traefik.http.routers.myservice-external.tls=true
      
      # Common configuration
      - traefik.http.services.myservice.loadbalancer.server.port=8080
```

### Authentication Integration

#### Authentik SSO

{{< hint tip >}}
Your configuration already includes the Authentik forward authentication middleware.
{{< /hint >}}

A sample to protect a service with Authentik authentication :

```yaml
services:
  protected-app:
    # ...
    labels:
      - traefik.enable=true
      - traefik.http.routers.protected-app.rule=Host(`app.local.domain.com`)
      - traefik.http.routers.protected-app.middlewares=middlewares-authentik@file
      - traefik.http.routers.protected-app.entrypoints=https
      - traefik.http.routers.protected-app.tls=true
```