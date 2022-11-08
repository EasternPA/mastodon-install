# mastodon-install
Instructions for installing the Mastodon Fediverse client using docker-compose

# Introduction

https://peterbabic.dev/blog/running-mastodon-with-docker-compose/

# Assumptions

# Prerequisites

1. Paid domain name that works with Let's Encrypt
1. Configure your domain to use Cloudflare for DNS
1. VPS on Oracle Cloud Free Tier
1. SSH public key to log into your OS image (ssh-keygen -t rsa)

## Buy a domain
`.xyz` domains typically cost $10.38 for 2 years at Namecheap
Note that `Let's Encrypt` does not support TLDs you can get for free when using Nginx Proxy Manager (NPM) in the way I describe below

## Migrate the DNS to Cloudlfare
Cloudflare will give you the two DNS servers assigned to manage your account
Update the DNS servers at your registrar to point to the two servers given to you by Cloudflare

## Register for a free Oracle Cloud account

## Create an AlwaysFree instance on Oracle Cloud
1. Select Ubuntu 22 for the image
1. Select Ampere arm for the platform and configure it with a minimum of 1 CPU and 6 GB RAM; use more if needed
1. Upload the public keyfile you generated for SSH logins
1. Create the instance
1. Wait 10 minutes for the instance to start
1. Locate the IP address from the instance information panel
1. Copy the IP address to the clipboard

## Create a DNS entry for your instance
1. Go to your Cloudflare dashboard
1. Click DNS from the main menu
1. Click Create to create a new DNS record
1. Choose `A` for the record type
1. Enter `fediverse` for the hostname, and paste the IP address for your cloud server instance
1. Choose `DNS Only` for the `Proxy status` option
1. Click Save
1. Click Create to create another DNS record
1. Choose `CNAME` for the record type
1. Enter `mstdn` for the hostname and `fediverse.` followed by the full domain name you purchased (i.e. `fediverse.mydomain.com`)
1. Choose `DNS Only` for the `Proxy status` option
1. Click Save

# Installation

## Install `docker` and `docker-compose`

### Update image to prepare for `docker` installation

1. `ssh ubuntu@mstdn.mydomain.com` (replace `mydomain.com` with the domain you purchased)

1. `sudo apt update && sudo apt upgrade -y`
1. Once the upgrade completes, hit enter on `Ok` then `<tab>` and `<enter>` on the next screen
1. `sudo reboot`
1. Wait about 45 seconds for the server to reboot and log back in with `ssh ubuntu@mstdn.mydomain.com` (again, replacing `mydomain.com`)
Note: If the upgrade does not proceed through to the end as expected, you may need to destroy and recreate the image. This happened to me once.

### Install `docker`
Install dependencies:
1. `$ sudo apt update`
1. `$ sudo apt install ca-certificates curl ufw apt-transport-https software-properties-common git -y`

Configure firewall:
1. `$ sudo ufw allow OpenSSH`
1. `$ sudo ufw enable`
1. `$ sudo ufw allow http`
1. `$ sudo ufw allow https`
1. `$ sudo ufw status` to verify the ufw status

Add `docker`'s GPG key to apt:
1. `$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`

Verify the GPG key was successfully added:
1. `$ sudo apt-key fingerprint 0EBFCD88`

Look for this output in the response:
```
pub   rsa4096 2017-02-22 [SCEA]
      9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
sub   rsa4096 2017-02-22 [S]
```

Note: change "arch=[amd64]" to "arch=[arm64]"
SKIP step to usermod -- that gives your non-root user root privileges.. no good
All future docker commands must be preceded by 'sudo'
step 4 - install compose
go to https://github.com/docker/compose in your browser
click "tags" above the top of the source code files
take note of the most recent release (currently v 2.12.2)

Copy and paste the following line into your terminal, replacing "2.12.2" with the current release:
sudo curl -L "https://github.com/docker/compose/releases/download/v2.12.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

Contine with the next command in step 4:
sudo chmod +x /usr/local/bin/docker-compose

Skip the command completion step

sudo docker-compose --version

git clone https://github.com/mastodon/mastodon.git
cd mastodon
date | md5sum
(copy sum for pg pwd)

sudo docker run --name postgres14 -v /home/ubuntu/mastodon/postgres:/var/lib/postgresql/data -e POSTGRES_PASSWORD=<pg pwd> --rm -d postgres:14-alpine

sudo docker exec -it postgres14 psql -U postgres
> CREATE USER mastodon WITH PASSWORD '<pg pwd>' CREATEDB;
> exit

sudo docker stop postgres14

touch .env.production
screen
sudo docker-compose run --rm -e DISABLE_DATABASE_ENVIRONMENT_CHECK=1 web bundle exec rake mastodon:setup

when you see the .env.production contents WAIT
copy the .env.production contents into the clipboard
ctrl-a d to exit screen
vi or nano .env.production
paste the .env.production contents
save and exit
run screen -r to return to the setup screen

continue with the setup process

SAVE YOUR ADMIN PASSWORD

docker-compose up -d
docker-compose down
sudo chown -R 70:70 ./postgres
sudo chown -R 991:991 ./public
#sudo chown -R 1000:1000 ./elasticsearch
sudo docker-compose up -d

run `date | md5sum` twice to generate a new passwords for the NPM MySQL root account and database

add NPM to the end of the docker-compose stack services section but before the networks: section. you need the volumes: declarative to be outdented all the way, like networks:

```
  npm-db:
    image: jc21/mariadb-aria:latest
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=<NPM MySQL root pwd>
      - MYSQL_DATABASE=npm
      - MYSQL_USER=npm
      - MYSQL_PASSWORD=<NPM MySQL pwd>
    volumes:
      - npm-db:/var/lib/mysql
    networks:
      - internal_network

  npm-app:
    image: jc21/nginx-proxy-manager:latest
    restart: always
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    environment:
      - DB_MYSQL_HOST=npm-db
      - DB_MYSQL_PORT=3306
      - DB_MYSQL_USER=npm
      - DB_MYSQL_PASSWORD=<NPM MySQL DB pwd>
      - DB_MYSQL_NAME=npm
    volumes:
      - npm-data:/data
      - npm-ssl:/etc/letsencrypt
    networks:
      - external_network
      - internal_network


volumes:
  npm-db:
  npm-ssl:
  npm-data:
```

docker-compose up -d

exit from the instance

ssh back in but add `-L 8082:localhost:81` to the end of the ssh command line

point your local browser at http://localhost:8082
log into NPM with admin@example.com / changeme
fill out your admin details

Get an API key from Cloudflare
Get an SSL certificate from Let's Encrypt using NPM; use the Cloudflare API key for the DNS challenge

Add a Proxy host for Mastodon pointing to the host called web on port 3000

Point your browser to your FQDN. 

log into mastodon with your admin id/pwd
go to preferences / invite people
generate a link; copy it
open a private browser window
pull up the invite link URL
fill in your non-root user details

go back to your ssh shell
`sudo docker exec -it mastodon-streaming-1 /bin/bash`
`RAILS_ENV=production bin/tootctl accounts modify <your non-root user> --confirm`
(look for `OK`) in return

go back to your private browser window
return to the main page for your mastodon FQDN
click the 3 vertical dots next to your username and choose logout at the bottom
verify that you can log back in with your non-root username/password
