title: "Home Server"
date: 2023-04-25T10:11:41+02:00
summary: "A guide to create your perfect home server."
aliases: ["/posts/home-server","/posts/server","/server"]
tags: ["home-server","home-lab","server","lab","media","download","docker","docker-compose"]
author: "Garnajee"
draft: true

# About

In this very complete guide (available [here on github](https://github.com/garnajee/home-server/)), you'll be able to install from scratch a perfect home-server for a full media donwload and stream automation

This is my near-perfect docker-compose for my streaming server.

You'll find two 2 main docker-compose, one to create the streaming services and the other one to access them through a reverse proxy.

The only thing to modify is the `.env` file to suits your setup.

Here, we are going to install everything under `/opt/chill/` (all services for full automation) and `/opt/docker/` (reverse proxy).

To install everything, just follow this Readme in this order.

## Requirements

* [docker](https://docs.docker.com/engine/install/)
* [docker-compose](https://docs.docker.com/compose/install/)
* a server maybe? I deployed these docker-compose on 2 servers: a Synology DS923+ and a custom server
* create docker [user](https://docs.docker.com/engine/install/linux-postinstall/)

Synology Server:

* Download the Docker App available on Synology Package Center
* Create a docker [user](https://trash-guides.info/Hardlinks/How-to-setup-for/Synology/#create-a-user) 
  * once it's done, connect on SSH and type `id <username>` and write down the `uid` and `gid` respectively `PUID` and `PGID` for the `.env`.
* Connect on SSH and download `docker-compose` [latest command version](https://docs.docker.com/compose/install/other/): 
  * if you are on a system that support `apt` packages, you can use the `compose` plugin: `apt install docker-compose-plugin` (it's better for updates) 
    * check version: `$ docker compose version` 
      * note: with the `compose` plugin you don't need the `-` between `docker` and `compose`
  * or you can use the binary: (I use the binary)

```bash
$ curl -SL https://github.com/docker/compose/releases/download/v2.17.2/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
# check version
$ docker-compose version
```

## Medias Server

### What's inside

This docker-compose contains:

* [Transmission-openvpn](https://github.com/haugene/docker-transmission-openvpn)
* [Jackett](https://github.com/Jackett/Jackett)
* [FlareSolverr](https://github.com/FlareSolverr/FlareSolverr)
* [Jellyfin](https://github.com/jellyfin/jellyfin)
* [Ombi](https://github.com/Ombi-app/Ombi)
* [Radarr](https://github.com/Radarr/Radarr)
* [Sonarr](https://github.com/Sonarr/Sonarr)
* [Removarr](https://github.com/garnajee/removarr)

### Installation

* Delete previous installation **be careful**

```bash
$ rm -rf /opt/chill
```

* Create the structure for every services

```bash
$ mkdir -p /opt/chill/{transovpn/{config,ratio},jackett/config,jellyfin/config,ombi/config,radarr/config,sonarr/config,bazarr/config,storage/downloads/{completed,incomplete,watch,medias/{movies,series}}}
```

Here is what it's going to create:

```bash
opt
└── chill
    ├── jackett
    │   └── config
    ├── jellyfin
    │   └── config
    ├── ombi
    │   └── config
    ├── radarr
    │   └── config
    ├── sonarr
    │   └── config
    ├── storage
    │   └── downloads
    │       ├── completed
    │       ├── incomplete
    │       ├── medias
    │       │   ├── movies
    │       │   └── series
    │       └── watch
    └── transovpn
        ├── config
        └── ratio
```

The `/opt/chill/transovpn/ratio/` folder is used to fake your upload stats on (semi-)private indexers. See [Fake Ratio](/posts/home-server/#fake-ratio).

The `/opt/chill/storage/downloads/watch/` folder is used when you manually put `.torrent` files, so it's going to be downloaded automatically without any access to the Transmission interface.

* To check everything was properly created

```bash
$ ls -lR /opt/chill
```

Download the `docker-compose-medias.yml` and `.env-example` in `/opt/chill/`, and rename them:

```bash
$ cd /opt/chill
$ wget https://raw.githubusercontent.com/garnajee/home-server/master/docker-compose-medias.yml -O docker-compose.yml
$ wget https://raw.githubusercontent.com/garnajee/home-server/master/.env-example -O .env
```

#### Modify `.env`

Normally, you don't need to modify the docker-compose.yml file. Only the `.env` file.

* list of [VPN PROVIDERS](https://haugene.github.io/docker-transmission-openvpn/supported-providers/)
* local subnet mask for `NETWORKIP`: `ip route | awk '!/ (docker0|br-)/ && /src/ {print $1}'`
* PUID and PGID

#### Create a docker network

This part is optional. It's going to be created automatically. But you can still do it manually if you want.

I did it manually because of the reverse proxy causing errors if I attached it to the automatically created docker subnet.

All these images are going to be in the same docker network. To avoid any further conflict, just create it before running the docker-compose.

Here the subnet is `10.10.66.0/24`. If you want to change this by something else, you'll need to modify the `docker-compose.yml` too. But remember, it's just a subnet *inside* docker, it's not going to affect your local network.

```bash
$ docker network rm net-chill
$ docker network create net-chill -d bridge --subnet 10.10.66.0/24
```

### Run!

For the first execution of the docker-compose, I recommend to use this command, to check if there is no errors:

Move the `docker-compose.yml` and `.env` file in `/opt/chill/`.

```bash
$ cd /opt/chill/
$ docker-compose up
```

It's going to download all the images, and start all the services.

**By the way, each request made by Jackett and FlareSolverr goes through the VPN.**

Now, if you don't see any red lines among the hundreds of lines that have just scrolled through the terminal, it's usually a good sign.

So you can now stop this by pressing `CTRL+c` (one or two times to force stop), and then run the docker-compose again, but in background this time:

```bash
$ cd /opt/chill/
$ docker-compose up -d
```

Check the status of each docker:

```bash
$ docker ps -a
```

`Status` should be `Up`.

### Check VPN connection

Now everything is up, just to be sure, you can check if Transmission goes through VPN:

```bash
$ docker exec -it transovpn curl ifconfig.co
XXX.XXX.XXX.XXX
```

And then check the location of this ip address [here](https://ifconfig.co).

### Update docker's images

#### Update one specific image

To update one specific docker image:

```bash
$ cd /opt/chill
$ docker-compose stop <container_name>
$ docker-compose rm <container_name>
$ docker-compose pull <container_name>
$ docker-compose up -d
```

#### Update all images

To update all images:

```bash
$ docker-compose down
$ docker-compose pull
$ docker-compose up -d
```

### Access services

To access services: (`IP` is the ip of your server)

| **Service** | **Address** |
|---------|---------|
| Transmission-openvpn | `<IP>:8000` |
| Jackett | `<IP>:8001` |
| FlareSolverr | `<IP>:8002` |
| Jellyfin | `<IP>:8003` |
| Ombi | `<IP>:8004` |
| Radarr | `<IP>:8010` |
| Sonarr | `<IP>:8011` |
| Removarr | `<IP>:8012` |

## Reverse Proxy

To access Jellyfin and Ombi from outside your local network, you'll need to set up a reverse proxy.

For that purpose, I'm going to use [Nginx Proxy Manager](https://nginxproxymanager.com/setup/).

You also need to open 2 ports on your router:

| Application/Service | Internal Port | External Port | Protocol | Equipment |
|---------------------|---------------|---------------|----------|-----------|
| HTTP | 8080 | 80 | TCP/UDP | Your-Server |
| HTTPS | 4443 | 443 | TCP/UDP | Your-Server |

### What's inside

This docker-compose contains:

* [Nginx Proxy Manager](https://github.com/NginxProxyManager/nginx-proxy-manager)
* [Maria DB Aria](https://github.com/jc21/docker-mariadb-aria)

### Installation

* Create the directory structure

```bash
$ mkdir -p /opt/docker/{nginx-proxy-manager,npm-db}
```

Download the `docker-compose-rp.yml` in `/opt/docker/`, and rename it:

```bash
$ cd /opt/docker
$ wget https://raw.githubusercontent.com/garnajee/home-server/master/docker-compose-rp.yml -O docker-compose.yml
```

I didn't create a `.env` file for this docker-compose mainly as this service is not going to be exposed on the web.

Feel free to modify `user`, `password`, `name`, and so on directly in the `docker-compose.yml`.

### Run

Simply execute this:

```bash
$ cd /opt/docker/
$ docker-compose up
```

Again, it's going to download the docker images. Check if there is no error, and if it's good, hit `CTRL+c` to stop everything and run this command:

```bash
$ docker-compose up -d
```

The service will be accessible at: `<IP>:81`.

The default Admin User is:

```
email: admin@example.com
password: changeme
```

You'll be ask to change these informations after the first logging.

### Get a domain name & SSL certificate

I bought a domain name from OVH. If you do the same, follow this:

* create an account on [OVH](https://ovh.com)
* buy a domain name
* once your domain name has been activated, create the necessary token for SSL
* use this [link](https://www.ovh.com/auth/api/createToken?GET=/domain/zone/*&POST=/domain/zone/*&PUT=/domain/zone/*&DELETE=/domain/zone/*)
* **Make sure to write everything**

You'll need something like this:

```
dns_ovh_endpoint = ovh-eu
dns_ovh_application_key = 109XXX7595XXXXXa
dns_ovh_application_secret = a1z2g3y435TGcazbXXXXXXXXa45e
dns_ovh_consumer_key = agf6hU1g13uj86XXXXXXXXXfv1l2n3g4j
```

* To create your certificate, go back on NPM (`<IP>:81`), "SSL Certificates" tab, "Add ..."
* fill the information for: `*.yourdomain.com` and `yourdomain.com`

## Setup all the services

### Transmission

Nothing to setup or maybe the "Speed Limits" depending on your internet connection.

### Jackett

Check the box:

* External access
* Cache enabled (recommended)

Add these values:

* "Cache TTL (seconds):" `2100`
* "Cache max results per indexer:" `1000`
* "FlareSolverr API URL:" `http://10.10.66.102:8191`    # **if you don't change anything on the docker-compose, that's it**
* "FlareSolverr Max Timeout (ms):" `100000`

Leave the rest blank.

Add your indexer(s).

### Radarr & Sonarr

Follow these [guides](https://trash-guides.info/).

I'll add more information later.

### Jellyfin

Follow the steps, it's easy.

Create an API key for Ombi.

Create all the users who are going to access to Jellyfin **and** Ombi.

To have better images for your libraires, you can use [these images](https://imgur.com/a/Guqk15B).

### Ombi

In the setting page:

* "Configuration" tab: 
  * (General) base URL: `/ombi`
  * (User Management) check the box. It means, that the Jellyfin account (user/password) is the same for Ombi.
* "Media Server" tab: 
  * Hostname/IP: put the internal docker ip: `10.10.66.103`.
  * Port: `8096`
  * API key: the one you created just before
  * then load libraries, submit.

Now for every service you're going to add (Sonarr, Radarr, ...) make sure to write the *internal docker ip address and internal port of the application*. The API keys are available in Sonarr/Radarr/... settings tab.

## Bonus

Don't want to use a reverse proxy?

You can use a VPN:

* Official Synology VPN package -> *need to open port on your router*

And if you don't want to open port on your router (or if you can't):

* [Tailscale](https://tailscale.com/kb/1131/synology/) on Synology
* [CloudFlare Tunnel](https://github.com/cloudflare/cloudflared)

Pay attention to not use CloudFlare tunnel for Jellyfin streaming, you may be banned for breaking TOS [term 2.8](https://www.cloudflare.com/en-gb/terms/).

### Webhooks

I provide some webhooks for Jellyfin. These webhooks are used for:

* Discord
* Microsoft Teams
* WhatsApp

First of all, you need to install the Jellyfin plugin called "Webhooks": Jellyfin > Dashboard > Plugins > Catalog > Webhook

Then, you need to restart Jellyfin:

```bash
$ cd /opt/chill/
$ docker-compose restart jellyfin
```

Now, go back to Jellyfin in the Plugins tab and clic on Webhook.

The "*Server Url*" is your Jellyfin URL. If you expose it on internet, it's something like this: https://yourdomainname.com/jellyfin

*If you don't have a domain name, Jellyfin will not be able to display images (posters) in the Discord/Teams webhooks.*

To add a **Discord** webhook:

* clic on "Add Discord Destination"
* "*Webhook Name*": what you want
* "*Webhook Url*": the discord webhook url
* "*Notification type*": 
  1. if you want to receive notification when a new item (movie/tv show/...) is added, check "*Item Added*"
  2. or when a user is locked out (because of too much wrong password during connection), check "*User Locked Out*"
  3. or when a user is created/deleted, when a password is changed, ... (for every use case check [this file](https://github.com/garnajee/home-server/blob/master/webhooks/jellyfin/discord/discord-users.handlebars)), check "*Authentication \**" and "*User \**"
* "*User Filter*": personnaly, I don't check anything here
* "*Item Type*": check everything (depending on your webhook template) **except** "*Send All Properties*"
* "*Template*": (copy and paste the content) 
  1. [discord-item.handlebars](https://github.com/garnajee/home-server/blob/master/webhooks/jellyfin/discord/discord-item.handlebars)
  2. [discord-users-locked-out.handlebars](https://github.com/garnajee/home-server/blob/master/webhooks/jellyfin/discord/discord-users-locked-out.handlebars)
  3. [discord-users.handlebars](https://github.com/garnajee/home-server/blob/master/webhooks/jellyfin/discord/discord-users.handlebars)
* "*Avatar Url*": just change the avatar profil picture directly on Discord
* "*Webhook Username*": should not be empty, but you can write what you want, the real username is defined directly in Discord
* "*Mention Type*": I never used this
* "*Embed Color*": color of the embeded message

Then clic save.

To add a **Microsoft Teams** webhhok:

* clic on "Add a Generic Destination"
* check the steps for Discord, it's the same
* "*Template*": 
  1. "*Item Added*": [teams-items.handlebars](https://github.com/garnajee/home-server/blob/master/webhooks/jellyfin/ms-teams/teams-items.handlebars)
  2. "*User Locked Out*": [teams-users-locked-out.handlebars](https://github.com/garnajee/home-server/blob/master/webhooks/jellyfin/ms-teams/teams-users-locked-out.handlebars)
  3. "*User*": [teams-users.handlebars](https://github.com/garnajee/home-server/blob/master/webhooks/jellyfin/ms-teams/teams-users.handlebars)

To add a **WhatsApp** webhook:

Not as easy as the others.

You need to connect you WhatsApp account as if you were logging on WhatsApp Web.

To acheive this, you're going to use another docker-compose to self-host a WhatsApp HTTP API. I'm using [this API](https://github.com/devlikeapro/whatsapp-http-api).

Follow these steps:

```bash
$ cd /opt/chill
$ wget https://raw.githubusercontent.com/garnajee/home-server/master/docker-compose-whatsapp.yml
$ docker-compose --file docker-compose-whatsapp.yml up -d
```

> Note that the docker-compose I provided is not really optmized, you can add environment variable to better configure. You can check the documentation [here](https://waha.devlike.pro/docs/how-to/config/).

> **Feel free to modify and perhaps make a pull request!**

Then follow the [official](https://github.com/devlikeapro/whatsapp-http-api#3-start-a-new-session) from step **3** to **5**. For any further information, like the id of a contact or a group, please read the [documentation](https://waha.devlike.pro/docs/how-to/).

Go back to Jellyfin > Plugin > Webhook:

* clic on "Add Generic Form Destination"
* "*Webhook Url*": the **internal** docker ip, if you don't change anything in the docker-compose it should be: `http://10.10.66.200:3000/api/sendText`
* then check what you want depending on the template

Then copy and paste the template you want:

1. "*Item Added*": [whatsapp-items.handlebars](https://github.com/garnajee/home-server/blob/master/webhooks/jellyfin/whatsapp/whatsapp-items.handlebars)
2. (very basic) "*User created/deleted*": [whatsapp-basic-user.handlebars](https://github.com/garnajee/home-server/blob/master/webhooks/jellyfin/whatsapp/whatsapp-basic-user.handlebars)

And finally you need to 2 Headers:

1. "*Key*": "accept", "*Value*": "application/json"
2. "*Key*": "Content-Type", "*Value*": "application/json"

> Please note, that we cannot send images with this API (it's a paid feature).

And that's it, you can save.

### Fake Ratio

If you need to fake your upload in order to have a ratio >= 1, you can use [Ratio.py](https://github.com/garnajee/Ratio.py).

I don't recommend using this indefinitely, please consider sharing to the community.
