# Éclats de Voix - Journal Lyonnais

This repository is about the setup of our Ghost blog using Docker. It is hosted on a VPS (OVH).

## Pre-requisite

- Docker (https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu) *(Could be installed already if using OVH with a VPS including Docker)*

## Straight-forward

Assuming you've got Docker installed and nothing running on port `80`, the whole setup is only a few command lines to put the Ghost blog online:

1. `docker run -d -p 80:80 -v /var/run/docker.sock:/tmp/docker.sock:ro jwilder/nginx-proxy:alpine`
1. `docker run -e VIRTUAL_HOST=my-blog.fr,www.my-blog.fr -d --name blog -p 3002:2368 -v /var/www/blog:/var/lib/ghost/content ghost:1.12.1-alpine`

You now have a Ghost instance running on your `localhost:3002` but also on both http://my-blog.fr and www.my-blog.fr (assuming you got a redirection there for both of them towards your VPS IP).

Detailled explanations below. *I use `blog.eclats-de-voix` instead of `blog` for my own setup.*

---

# Testing/Dev

If you wanna play around before deploying anything anywhere but on your localhost.

## Create and start the docker container

`docker run -d --name blog.eclats-de-voix -p 3002:2368 -v /var/www/blog.eclats-de-voix:/var/lib/ghost/content ghost:1.12.1-alpine`

This will create a container `blog.eclats-de-voix`, available through `localhost:3002`.
A volume will also be available at `/var/www/blog.eclats-de-voix`, on the **host**. This is **very important** as it allows the data (DB, images, themes, ...) to live outside of the container life cycle. If the container is destroyed for any reason, your DB is safe.

Note that if the host itself is destroyed, the data are **lost**.

This container will be based on the official `ghost:1.12.1-alpine` version. See https://hub.docker.com/_/ghost/

It is recommended to use a specific version of the image (1.12.1 here).

## Stop/Start the container

`docker stop blog.eclats-de-voix`

`docker start blog.eclats-de-voix`

---

# Deploiement

In order to deploy (production) without efforts, we are using Nginx through Docker, using a proxy.

We are using the awesome https://github.com/jwilder/nginx-proxy to do so. It basically listens to the port `80` and acts as a proxy to serve the right content depending on the url the request is coming from.

We need to load/run the proxy itself, running on port 80:

`docker run -d -p 80:80 -v /var/run/docker.sock:/tmp/docker.sock:ro jwilder/nginx-proxy:alpine`

Then, we can use any other container we wich to serve.
`docker run -e VIRTUAL_HOST=eclats-de-voix.fr,www.eclats-de-voix.fr -d --name blog.eclats-de-voix -p 3002:2368 -v /var/www/blog.eclats-de-voix:/var/lib/ghost/content ghost:1.12.1-alpine`

This serves our Ghost both on 3002 and on the defined `VIRTUAL_HOST`. See http://eclats-de-voix.fr/ and http://www.eclats-de-voix.fr/.

**NOTE**: The `-e` option must be given first.

## Start a new container based on existing data

`docker run -e VIRTUAL_HOST=eclats-de-voix.fr,www.eclats-de-voix.fr -d --name blog.eclats-de-voix-2 -p 3003:2368 -v /var/www/blog.eclats-de-voix:/var/lib/ghost/content ghost:1.12.1-alpine`

The volume at `/var/www/blog.eclats-de-voix` will be used to setup this new container and therefore you won't have any difference between the first and second container.

**Note that if two instances can run at the same time, it's not something you should do in production.** (You'll rush into data corruption issues because the DB/cache isn't correctly shared between the two instances and therefore won't automatically update)
