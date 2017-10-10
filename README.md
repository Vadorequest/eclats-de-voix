# Éclats de Voix - Journal Lyonnais

This repository is about the setup of our Ghost blog using Docker. It is hosted on a VPS (OVH).

## Production

### Pre-requisite

- Docker (https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu)

### Create and start the docker container

`docker run -d --name blog.eclats-de-voix -p 3002:2368 -v /var/www/blog.eclats-de-voix:/var/lib/ghost/content ghost:1.12.1-alpine`

This will create a container `blog.eclats-de-voix`, available through `localhost:3002`.
A volume will also be available at `/var/www/blog.eclats-de-voix`, on the **host**. This is **very important** as it allows the data (DB, images, themes, ...) to live outside of the container life cycle. If the container is destroyed for any reason, your DB is safe.

Note that if the host itself is destroyed, the data are **lost**.

This container will be based on the official `ghost:1.12.1-alpine` version. See https://hub.docker.com/_/ghost/

It is recommended to use a specific version of the image (1.12.1 here).

### Stop/Start the container

`docker stop blog.eclats-de-voix`

`docker start blog.eclats-de-voix`

### Start a new container based on the same data

`docker run -d --name blog.eclats-de-voix-2 -p 3003:2368 -v /var/www/blog.eclats-de-voix:/var/lib/ghost/content ghost:1.12.1-alpine`

The volume at `/var/www/blog.eclats-de-voix` will be used to setup this new container and therefore you won't have any difference between the first and second container.
Note that if two instances can run at the same time, it's not something you should do in production.
