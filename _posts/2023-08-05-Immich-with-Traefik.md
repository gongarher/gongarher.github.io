---
layout: post
title:  "Immich over Traefik on Raspberry Pi - Lightweight & minimal"
author: Gonzalo García Hernantes
categories: [ photos, docker, traefik ]
image: assets/images/immich.png
---

This project was born with the necessity of owning a lightweight self-hosted media service that accomplishes the functionality of Google Drive. After trying to setup multiple containers such as nextcloud, filestash or photoprism, I come up with this solution that combines [Immich](https://immich.app/) and [Traefik](https://doc.traefik.io/traefik/) due to its low-resources. Take into account that I have to **focus on the resoure's efficiency** if I want to keep my Raspberry Pi 4 1Gb running smoothly.

In this post, I will explain all the steps I followed to get everything up & running.


## Architecture

I decided to deploy the minimun docker container in order to get Immich up & running: 

+ Immich server
+ Immich microservices
+ Immich web
+ redis
+ database (postgres)
+ Immich-proxy (then replaced with Traefik container)

I'm going to deploy my stack via docker compose so I can keep my configuration, taking the [docker compose example shown in the official documentation](https://github.com/immich-app/immich/blob/v1.72.1/docker/docker-compose.yml) as my starting point.

## Docker compose initial configuration

Firstly, as I want to get the minimal immich service working, I need to remove **immich-machine-learning** and **immich-typesense** services and dependencies from our docker compose. 

To increase security, I'm going to deploy my immich stack in their own separate network. Here is my docker compose file:


#### docker-compose.yaml

```yaml
version: "3.8"
services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    command: [ "start.sh", "immich" ]
    networks:
      - immich
    volumes:
      - /path/to/upload:/usr/src/app/upload
    env_file:
      - ./services/immich/.env
    depends_on:
      - redis
      - database
    restart: unless-stopped

  immich-microservices:
    container_name: immich_microservices
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    command: [ "start.sh", "microservices" ]
    networks:
      - immich
    volumes:
      - /path/to/upload:/usr/src/app/upload
    env_file:
      - ./services/immich/.env
    depends_on:
      - redis
      - database
    restart: unless-stopped

  immich-web:
    container_name: immich_web
    image: ghcr.io/immich-app/immich-web:${IMMICH_VERSION:-release}
    networks:
      - immich
    env_file:
      - ./services/immich/.env
    restart: unless-stopped

  redis:
    container_name: immich_redis
    image: redis:6.2-alpine@sha256:70a7a5b641117670beae0d80658430853896b5ef269ccf00d1827427e3263fa3
    networks:
      - immich
    restart: unless-stopped

  database:
    container_name: immich_postgres
    image: postgres:14-alpine@sha256:28407a9961e76f2d285dc6991e8e48893503cc3836a4755bbc2d40bcc272a441
    networks:
      - immich
    env_file:
      - ./services/immich/.env
    volumes:
      - ./path/to/immich/postgres:/var/lib/postgresql/data
    restart: unless-stopped

  immich-proxy:
    container_name: immich_proxy
    image: ghcr.io/immich-app/immich-proxy:${IMMICH_VERSION:-release}
    networks:
      - immich
    env_file:
      - ./services/immich/.env
    ports:
      - 2283:8080
    depends_on:
      - immich-server
      - immich-web
    restart: unless-stopped

networks:  
  immich:
    external: true
```

Once generated my docker-compose file, I must customize **/path/to/upload** with the folder where I want to store my media library. Then, add some environment variables to our env file (./services/immich/.env). I'm using a separate env file in order to keep it isolated from other projects and not use .env default file.

#### Env file
```bash
# Postgres
DB_HOSTNAME=immich_postgres
DB_USERNAME=postgres
POSTGRES_USER=postgres
DB_PASSWORD="<Your db password>"
POSTGRES_PASSWORD="<Your db password>"
DB_DATABASE_NAME=immich
POSTGRES_DB=immich

# Redis
REDIS_HOSTNAME=immich_redis

# Typesense
TYPESENSE_ENABLED=false

# Proxy
IMMICH_WEB_URL=http://immich-web:3000
IMMICH_SERVER_URL=http://immich-server:3001

```
Note: As some postgres vars are being used from multiple containers, I duplicated some of those varse in order to get all my containers working with the same envfile.

Now its time to deploy immich services and see how they work! Once all of them are up & running, go to **http://SERVER_LOCAL_IP:2283** in order to create your admin user and customize your immich instance :).

That's the way of setting up the minimal & basic configuration. Now I'm going to explain how to integrate immich with traefik in order to safely expose my service to the internet.

## Replacing immich proxy with Traefik

As the Immich High-Level Diagram describes, the immich access is set via its web and server containers, so I'm going to expose those via traefik, replacing immich-proxy (with traefik).

![walking]({{ site.baseurl }}/assets/images/immich-architecture.png)

Apart from removing my immich-proxy container, I have to integrate the immich-server container with traefik. As described in [immich reverse proxy documentation](https://documentation.immich.app/docs/administration/reverse-proxy), the API calls have to be redirected to immich-server container. That's why I put **Pathprefix(`/api`)** on the traefik rule. In addition, I decided to implement my torblock and api-strip middlewares as well as force HTTPS entrypoint due to security concerns.

#### immich-server behind traefik

```yaml
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-release}
    command: [ "start.sh", "immich" ]
    networks:
      - traefik-public
      - immich
    labels:
      traefik.enable: "true"
      traefik.http.services.media-immich-api.loadbalancer.server.port: "3001"
      traefik.http.routers.media-immich-api.rule: "Host(`immich.<MY_DOMAIN>`) && Pathprefix(`/api`)"
      traefik.http.routers.media-immich-api.middlewares: my-torblock@docker, service-immich-api-strip
      traefik.http.middlewares.service-immich-api-strip.stripprefix.prefixes: "/api"
      traefik.http.routers.media-immich-api.tls: true
      traefik.http.routers.media-immich-api.tls.certresolver: letsencrypt
      traefik.http.routers.media-immich-api.entrypoints: websecure
    volumes:
      - /path/to/upload:/usr/src/app/upload
    env_file:
      - ./services/immich/.env
    depends_on:
      - redis
      - database
    restart: unless-stopped
```

I need to reconfigure my immich-web service too:
#### immich-web behind traefik

```yaml
  immich-web:
    container_name: immich_web
    image: ghcr.io/immich-app/immich-web:${IMMICH_VERSION:-release}
    networks:
      - traefik-public
      - immich
    env_file:
      - ./services/immich/.env
    labels:
      traefik.enable: "true"
      traefik.http.services.media-immich.loadbalancer.server.port: "3000"
      traefik.http.routers.media-immich.rule: "Host(`immich.<MY_DOMAIN>`)"
      traefik.http.routers.media-immich.middlewares: my-torblock@docker
      traefik.http.routers.media-immich.tls: true
      traefik.http.routers.media-immich.tls.certresolver: letsencrypt
      traefik.http.routers.media-immich.entrypoints: websecure
    restart: unless-stopped
```


and define my traefik-public network on my docker-compose file.
#### Networking definition

```yaml
networks:  
  immich:
  traefik-public:
    external: true
```

Note: It's an external network due to I have already defined the network in my traefik project.

Now I can access to my immich instance from anywhere via the Internet just typping **https://immich.MY_DOMAIN** 

## Source documentation

+ https://documentation.immich.app/docs/developer/architecture
+ https://documentation.immich.app/docs/administration/reverse-proxy
  


