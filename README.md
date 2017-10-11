# Éclats de Voix - Journal Lyonnais

This repository is about the setup of our Ghost blog using Docker. It is hosted on a VPS (OVH).
This README is both a tutorial/explanation/reminder for myself and others who are interested to host Ghost on their own server using Docker.

## Pre-requisite

- Docker (https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu) *(Could be installed already if using OVH with a VPS including Docker)*

## Straight-forward

Assuming you've got Docker installed and nothing running on port `80`, the whole setup is only a few command lines to put the Ghost blog online:

1. `docker run -d -p 80:80 -v /var/run/docker.sock:/tmp/docker.sock:ro jwilder/nginx-proxy:alpine`
1. `docker run -e VIRTUAL_HOST=my-blog.fr,www.my-blog.fr -e url=http://my-blog.fr/ -d --name blog -p 3002:2368 -v /var/www/blog:/var/lib/ghost/content ghost:1.12.1-alpine`

You now have a Ghost instance running on your `localhost:3002` but also on both http://my-blog.fr and www.my-blog.fr (assuming you got a redirection there for both of them towards your VPS IP).

Detailled explanations below. *I use `blog.eclats-de-voix` instead of `blog` for my own setup.*

---

# Testing/Dev

If you wanna play around before deploying anything anywhere but on your localhost.

## Create and start the docker container

`docker run -e url=http://eclats-de-voix.fr/ -d --name blog.eclats-de-voix -p 3002:2368 -v /var/www/blog.eclats-de-voix:/var/lib/ghost/content ghost:1.12.1-alpine`

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
`docker run -e VIRTUAL_HOST=eclats-de-voix.fr,www.eclats-de-voix.fr -e url=http://eclats-de-voix.fr/ -d --name blog.eclats-de-voix -p 3003:2368 -v /var/www/blog.eclats-de-voix:/var/lib/ghost/content ghost:1.12.1-alpine`

This serves our Ghost both on 3002 and on the defined `VIRTUAL_HOST`. See http://eclats-de-voix.fr/ and http://www.eclats-de-voix.fr/.

Also, we define our `url` through an environment variable, which is necessary for many Ghost internal operations such as inviting team members, etc.

**NOTE**: The `-e` option must be given first.

## Start a new container based on existing data

`docker run -e VIRTUAL_HOST=eclats-de-voix.fr,www.eclats-de-voix.fr -d --name blog.eclats-de-voix-2 -p 3003:2368 -v /var/www/blog.eclats-de-voix:/var/lib/ghost/content ghost:1.12.1-alpine`

The volume at `/var/www/blog.eclats-de-voix` will be used to setup this new container and therefore you won't have any difference between the first and second container.

**Note that if two instances can run at the same time, it's not something you should do in production.** (You'll rush into data corruption issues because the DB/cache isn't correctly shared between the two instances and therefore won't automatically update)

---

# Backups

Having a blog online is nice. Having it run on a VPS that costs 3€/month is nice (could be done on a Raspberry PI for $0 though). Having to type two command line to set it up is even nicer. But what happens when your VPS crash because you forgot to handle the disk usage? Or because it got a malfunction? (yeah, you're the unlucky type)

That's what backups are for, right? And, to be honest, I wouldn't run a blog in production without **backups that actually work!**

## Automated (cron), incremential backups using Rsync

I used the Rsync protocol, which is very common in Linux. It basically sync a folder from a computer, to another computer. That's it.

You could have your own server, and do the setup to handle Rsync requests through ssh. I've used http://rsync.net/cloudstorage.html instead, for the sake of simplicity (and cost). I pay about $8/month for 100Go of data, with unlimitted bandwith.

Honnestly, Rsync website is a bit ugly, and the docs are worse (css is has-been :/). Once you pay, you gotta wait for 4-8h before you get the email giving you the credentials for the server. There is a lot of explanation and links towards the doc in the email though. Helpful.

Basically, you gotta setup a SSH key on your VPS, and store it to your Rsync.net account through cli. Then, you have to setup a cron job:

```
$ crontab -e

15 0,7,10,13,16,18,21 * * * /root/rsyncnet_backup.sh
```

Create the file `/root/rsyncnet_backup.sh` and make it executable `chmod +x /root/rsyncnet_backup.sh`.

```bash 
#!/bin/sh

LOCKFILE=/root/.rsyncnet_lock

if [ -e ${LOCKFILE} ] && kill -0 `cat ${LOCKFILE}`; then
     echo "A backup is already running"
     exit
fi

# make sure the lockfile is removed when we exit and then claim it
trap "rm -f ${LOCKFILE}; exit" INT TERM EXIT
echo $$ > ${LOCKFILE}

/usr/bin/rsync -avH /var/www/blog.eclats-de-voix 12074@ch-s012.rsync.net:

rm -f ${LOCKFILE}
```

This will do a backup at midnight, 7, 10am, then 1, 4, 6 and 9pm. It will send all files (recursively) from `/var/www/blog.eclats-de-voix` into a `blog.eclats-de-voix/` folder on the remote. Additionally, a backup must be finished before another can start. (the script takes care of that)

In addition, I **strongly recommend** to setup an "Idle warning" through the Rsync Dashboard at https://www.rsync.net/am/dashboard.html so you get notified if backups stop being done for any reason.


### Helpers

- **Manual backup**: Run `rsync -avH /var/www/blog.eclats-de-voix 12074@ch-s012.rsync.net:`
- **List Backup data**: Run `rsync --list-only 12074@ch-s012.rsync.net:blog.eclats-de-voix/`
- **Delete all backups [DANGEROUS]**: Run `rsync -avH --delete /var/www/blog.eclats-de-voix 12074@ch-s012.rsync.net:` (good to know when playing around)
- **Downloading backup**: Run `rsync -chavzP --stats 12074@ch-s012.rsync.net: /var/www/blog.eclats-de-voix-backup/`, a new `/var/www/blog.eclats-de-voix-backup/` will contain a `blog.eclats-de-voix` folder containing the actual data.
- **Listing all snapshots, per day**: Run `rsync --list-only 12074@ch-s012.rsync.net:.zfs/snapshot/` The .zfs folder is hidden and won't show otherwise.

## Checking if the backup actually works

To do so, I simply run another container on another port and make it load the dowloaded backup.

`docker run -d --name blog.eclats-de-voix-2 -p 3003:2368 -v /var/www/blog.eclats-de-voix-backup/blog.eclats-de-voix:/var/lib/ghost/content ghost:1.12.1-alpine`

Then, go to http://localhost:3003 and check if it worked. :)


