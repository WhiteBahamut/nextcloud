# Description 
This repo will show how to run a [Nextcloud server](https://nextcloud.com/) with [Traefik](https://containo.us/traefik/) as reverse proxy and [Let's Encrypt](https://letsencrypt.org/) on a [Raspberry Pi 4](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/) Docker Swarm cluster.
I created this repo mainly as my setup before had the [nginx](https://github.com/jwilder/nginx-proxy) and a [lets-encrypt-companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion). As this was also the example nextcloud has in their [repo](https://github.com/nextcloud/docker/tree/master/.examples/docker-compose/with-nginx-proxy-self-signed-ssl/mariadb/fpm).
Sadly the nginx and companion do not run on a Raspberry Pi 4 with its ARM architecture.
Most of the Traefik setup can also be found in their [docs](https://docs.traefik.io/user-guides/docker-compose/acme-http/). Yet most documentation I found only shows a single compose setup. But in my case I have multiple different stacks up and running.
 
# Installation
## Pre requisits
- Docker swarm running
- Basic knowladge of nextcloud setup

## Setup Traefik
Traefik will be used as reverse proxy and also updates the Let's Encrypt certificate.
Setup the commands to start Traefik with.
```
    command:
      - "--api.insecure=true"
```

First of all enable the docker swarm integration. 
```
      - "--providers.docker.swarmmode=true"
      - "--providers.docker.network=traefik_traefikfront"
      - "--providers.docker.exposedbydefault=false"
```
With `--providers.docker.network=traefik_traefikfront` you specify to use the network `traefik_traefikfront` for the loadbalancing (I had not done this in the first place and traefik was constantly rotating the different docker network IPs of my nextcloud instance). The additional `traefik` in front of the network comes from the stackname Traefik is deployed to.
In addition set `--providers.docker.exposedbydefault=false`, so not all services are exposed.

Next set the web and secureweb endpoints
```
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
```

And then set the Let's Encrypt parameter (ACME Challange)
```
      - "--certificatesresolvers.myhttpchallenge.acme.httpchallenge=true"
      - "--certificatesresolvers.myhttpchallenge.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.myhttpchallenge.acme.email=admin@your_domain"
      - "--certificatesresolvers.myhttpchallenge.acme.storage=/letsencrypt/acme.json"
```
Actually `myhttpchallenge` can be changed with another value. It is used to identify the resolver later on.
Mount the docker.sock and acme.json
```
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /traefik/acme.json:/letsencrypt/acme.json
```
The file needs to created before and requires `chmod 660` (Traefik log will also indicate this, if you forgot it).

See the full example  in `traefik-compose.yaml`.

## Setup nextcloud
To enable Traefik to reverse-proxy to the nextcloud instance some lables need to be applied to the nextcloud service.
```
    deploy:
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.nextcloud.rule=Host(`nextcloud.your_domain`)"
        - "traefik.http.routers.nextcloud.entrypoints=websecure"
        - "traefik.http.routers.nextcloud.tls.certresolver=myhttpchallenge"
        - "traefik.http.services.nextcloud.loadbalancer.server.port=80"
        - "traefik.docker.network=traefik_traefikfront"
```
The nextcloud service will only have an entrypoint in the `websecure` endpoint in Traefik. Also, you can find the certresolver from the Traefik command in `traefik.http.routers.nextcloud.tls.certresolver=myhttpchallenge`.

To let Traefik and the nextcloud service communicate with each other, the network needs to be share.
```
...
    networks:
      - traefik_traefikfront
      - back
      
...

networks:
  traefik_traefikfront:
    external: true
  back:

```

See the full example in `nextcloud-compose.yaml`.

## Deploy the stacks
Now deploy the compose files with `docker stack deploy ...` or some Swarm UI like https://www.portainer.io/

## Docker network
The example creates a the proxy network called 'traefik_traefikfront' as part of `traefik-compose.yaml`. This means Traefik must be deployed before Nextcloud (or other services). Depending on the overall setup the proxy network should be created separately (`docker network create`).  

# Contribution
Feel free to improve/correct the compose files or text here.
