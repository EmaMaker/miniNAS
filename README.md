# miniNAS Setup

Files for my miniNAS Server, a self-hosted server for different services.<br> Built from computers parts I had laying around, and a custom-built wooden case built from scrap wood.<br>
Currently running an AMD A10-5700 4/4 APU @3.4GHz, 2x4GB DDR3 RAM, 2x1TB HDD in RAID1 and a 120GB Kingston SSD<br>
miniNAS ships different services: nextcloud, gitea, searx and whatever I decide to add to it.<br>
This repo contains the file for configuring the server and an explanation of how they work

## Contents of the repo
* docker_compose: docker compose files for the different services. They use environment variables for volume directories and other things ([more info on environment variables](#env-vars)).
Plase those files wherever you want, but keep in mind that systemd entries will need to be changed accordingly. The directory where they are placed should follow this scheme: *compose_dir*/*service_name*/docker-compose.yml. For example gitea.yml should be placed in *compose_dir/gitea/ and renamed to docker-compose.yml

* systemd: systemd unit entries for starting the docker containers. Those need to be placed in /etc/systemd/system, then reload the daemon with

        sudo systemctl daemon-reload

    Now you can enable/start/disable/stop them. Use

        sudo systemctl enable docker-compose@service_name --now

    to enable and start the service on the spot
Also note that systemd entries use docker installed from snap (stored in /var/snap/docker). If using another distro, you'll have to change this. I used this as an example 
https://gist.github.com/mosquito/b23e1c1e5723a7fd9e6568e5cf91180f

* gitea: contains app.ini config file for gitea. Place it in $GITEA_DATADIR/gitea/conf/
* nginx: contains custom nginx configs. $NGINX_CONF_DIR needs to point to it.

* ddclient: DDClient config script, location depends on the distro. Activate its systemd entry. DDclient updates the dns record with the public ip of the machine. I use google domains so it's configured for that, stripped down of the creds of course. Examples for different services can be found on the ddclient wiki.


## Backup strategy
I recently saw a [video about backups by Jeff Geerling](https://www.youtube.com/watch?v=S0KZ5iXTkzg) and I decided to apply his 3-2-1 rule, although mine looks a bit more scuffed. I keep all of my important files on my nextcloud instance, and use my gitea as my git service.

1) All of the data lives on the RAID1, which copies the data on two different drives at the same time
2) I use Nextcloud's sync client on my machines (mostly my desktop and laptop), which downloads the data locally on the machine it's used. I tend to sync all of my nextcloud data, although most times I sync just part of it, so most of my data as a couple more copies on two different machines.
3) I keep my projects on Nextcloud too, and every project has its own git repo which is then pushed to my own gitea. Gitea data lives on the 1TB drive of miniNAS, this means that I have two copies of my Gitea data on two different medias. I tend to sync my Projects on both my desktop and laptop, this means that I have a bit more redudancy for my projects too (see point 2)
4) My Gitea data gets mirrored to my GitHub periodically and automatically
5) Both my compressed nextcloud backups and my gitea data gets compressed and copied to an external 750GB drive periodically.

I'm still looking for an offsite backup solution for nextcloud that is cost-effective. After a friend's suggestion, I started looking into [IPFS](https://ipfs.io/).


## Notes on services:

### Nginx
* Taken from: https://github.com/gilyes/docker-nginx-letsencrypt-sample/blob/master/docker-compose.yml
and adapted to my needs.
* Ports: Each port needed by a service must be listed under "ports" in the docker-compose. The router needs to port forward the ports needed to the machine. Ports 3000 and 2222 are needed by gitea, ports 80 and 443 are needed for letsencrypt and https traffic and need to be always open. This service also takes care of creating new certificates for new subdomains when needed and renewing existing certificates. Ports needed by each service will need to be exposed in the corresponding docker-compose.yml



## Environment variables {#env-vars}

Environment variables can be placed wherever on the system, in a single text file. Then change *EnvironmentFile* in *docker-compose@.service* to point to the absolute path of the file you have created. <br>
These are the available environment variables:

### Docker
* **SERVER_DOCKER_NETWORKNAME**: name of the network. Create it with

        docker network create my_network

### Letsencrypt (https/ssl certificates)
* **LETSENCRYPT_EMAIL**: email to send letsencrypt messages to
* **LETSENCRYPT_DOMAIN**: domain to request certificates for

### Certs
* **CERTS_DIR**: directory where letsencrypt certificates are stored, with owner UID 1000 and GID 1000. Generate them with

        certbot certonly

    then follow the on-screen instructions and copy them to CERTS_DIR

### Gitea
* **GITEA_DATADIR**: directory where gitea data will be stored

### Nginx
* **NGINX_CONF_DIR**: custom nginx configurations

### Nextcloud

* **NEXTCLOUD_DATADIR**: directory where nextcloud data is stored (files, apps, etc.)
* **NEXTCLOUD_DB_DATADIR**: directory where nextcloud database data is stored. Database is needed when restoring a backup

#### Autoconfiguration for nextcloud. More about it on [Nextcloud's official docker repo](https://github.com/nextcloud/docker)
* **NEXTCLOUD_ADMIN_USER**
* **NEXTCLOUD_ADMIN_PWD**
* **NEXTCLOUD_MYSQL_ROOTPWD**
* **NEXTCLOUD_MYSQL_PWD**
