# mastodon-install
Instructions for hosting a small Mastodon server instance for free (well, *almost*) using docker-compose

# Introduction

Mastodon is a micro-blogging application (similar in style to Twitter) that uses ActivityPub as the federation protocol on the backend. Mastodon is just one of several social networking applications in the Fediverse. You can learn more at [fediverse.party](https://fediverse.party).

This document provides step-by-step instructions for hosting an instance of Mastodon for almost no cost (under $11USD for the first 2 years).

I tried to follow numerous instructional documents that had been assembled over the years, but I needed to combine (and tweak) steps from each of these documents in order to arrive at a working configuration. The steps explained below were drawn from the following documents, and I thank each of the authors for their contributions to this space.

https://peterbabic.dev/blog/running-mastodon-with-docker-compose/

https://www.howtoforge.com/how-to-install-mastodon-social-network-with-docker-on-ubuntu-1804/

https://sleeplessbeastie.eu/2022/05/02/how-to-take-advantage-of-docker-to-install-mastodon/

# Assumptions
Familiarity with Linux, domain registration, DNS management, and web hosting, including SSL encryption.

# Prerequisites

1. Paid domain name that works with Let's Encrypt
1. Domain reconfigured to use Cloudflare for DNS
1. A Virtual Private Server (VPS) on Oracle Cloud's Free Tier
1. An SSH public key for logging into your OS image (run `ssh-keygen` in Linux)

## Register for a free Cloudflare account

Register at https://cloudflare.com

## Buy a domain
Let's Encrypt offers free SSL certificates for domain owners, but they do not make it easy get certificates for Top Level Domains (TLDs) you can get for free at various websites. I prefer the easy web UI method, and that requires a domain with a non-free TLD. Namecheap has been selling domains ending in `.xyz` for $10.38 for the first 2 years and they work fine for this. Domains that end in `.social` cost just a little more and tend to be more popular on the Fediverse.

## Migrate the DNS to Cloudflare
Cloudflare will give you the two DNS servers assigned to manage your account. The process to change the DNS servers managing your domain will vary by registrar and is done at your registrar's site. Update the DNS servers at your registrar to point to the two servers given to you by Cloudflare.

## Register for a free Oracle Cloud account

Note: You must set `Shields Down` for this site if you use the _Brave_ browser. The website will not function properly with the shields up.

## Create an AlwaysFree instance on Oracle Cloud
1. Change the name of the instance to `mastodon` if you do not plan to host other Fedi services; use `fediverse` if you do
1. Click `Edit` in the `Image and Shape` section
1. Click the `Change image` button next to `Oracle Linux 8`
1. Select Canonical Ubuntu for the image
1. Click `Select image` at the bottom
1. Click the `Change shape` button next to `AMD VM Standard`
1. Select `Ampere` from the `Shape Series` options list
1. Select the only shape available on the `Shape name` list
1. Configure it with a minimum of 1 CPU and 6 GB RAM; use more if you will host several users
1. Click `Select Shape` at the bottom
1. Scroll down beyond the `Networking` section of the configuration panel
1. Upload the public keyfile you generated for SSH logins (located at `~/.ssh/id_rsa.pub`)
1. Click `Create` at the bottom
1. Wait 10 minutes for the instance to start
1. Locate the IP address in the top right corner of the instance information panel
1. Copy the IP address to the clipboard

## Create a DNS entry for your instance
1. Go to your Cloudflare dashboard in another browser tab
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

Note: If the upgrade does not proceed through to the end as expected, you may need to destroy and recreate the image. This happened to me once while developing these instructions.

### Install `docker`
Install dependencies:
1. `sudo apt install ca-certificates curl ufw apt-transport-https software-properties-common git -y`

Configure firewall:
1. `sudo ufw allow OpenSSH`
1. `sudo ufw enable`
1. `sudo ufw allow http`
1. `sudo ufw allow https`
1. `sudo ufw status` to verify the ufw status

Add the GPG key for the `docker` repository to apt:

`curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`

Verify the GPG key was successfully added:

`sudo apt-key fingerprint 0EBFCD88`

Look for this output in return:
```
pub   rsa4096 2017-02-22 [SCEA]
      9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
sub   rsa4096 2017-02-22 [S]
```

Add the `docker` repository for the `arm` architecture:

`sudo add-apt-repository "deb [arch=arm64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"`

Note: I do not like to add my non-root user to the docker group as this grants broad powers to the user. Just use `sudo` for commands that require `root` privileges, such as `docker` and `docker-compose`.

Install `docker-compose`:
1. Go to `https://github.com/docker/compose` in your browser
1. Click `tags` above the top of the source code files
1. Take note of the most recent release (currently v2.12.2)

Copy and paste the following line into your terminal, replacing "2.12.2" with the current release:

```
sudo curl -L "https://github.com/docker/compose/releases/download/v2.12.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

Run `sudo docker-compose --version` and ensure the current version number is returned

## Install `mastodon`

### Download the source code

1. `git clone https://github.com/mastodon/mastodon.git`
1. `cd mastodon`
1. `date | md5sum`

You will see output like this:

`e351f61406c8ba6bdc489fdc4606c7c3  -`

Copy the long string into your clipboard. Do not include the space or `-` at the end. You will paste it where you see `<pg pwd>` below (two places!)

```
sudo docker run --name postgres14 -v /home/ubuntu/mastodon/postgres:/var/lib/postgresql/data -e POSTGRES_PASSWORD=<pg pwd> --rm -d postgres:14-alpine`
sudo docker exec -it postgres14 psql -U postgres
```

While inside the PostgreSQL container, run:

`CREATE USER mastodon WITH PASSWORD '<pg pwd>' CREATEDB;`

Note: include the single quotes `'` in the above command. Look for `ROLE CREATED` in the response. If successful, type `exit`.

Stop the container:

`sudo docker stop postgres14`

Kick off the build:
1. `touch .env.production`
1. `screen`
1. `sudo docker-compose run --rm -e DISABLE_DATABASE_ENVIRONMENT_CHECK=1 web bundle exec rake mastodon:setup`

## DETAILED ANSWERS TO THE QUESTIONS MISSING HERE.. still working on it!

When you see the suggested contents of `.env.production` shown, WAIT

1. Copy the suggested contents of `.env.production` into the clipboard
1. Hit `Ctrl-a` then `d` to exit `screen`
1. Edit `.env.production` with `vi` or `nano`, whichever you're comfortable using
1. Paste in the contents of `.env.production` from the clipboard
1. Save and exit (`Ctrl-x, y, <Enter>` in `nano` or `<Esc>:wq<Enter>` in `vi`)
1. Run `screen -r` to return to the setup screen

Continue with the setup process

## MORE DETAILS MISSING HERE

When the process completes, you will see your admin password. 

## SAVE YOUR ADMIN PASSWORD

## Prepare the final build
1. Run `docker-compose up -d`
1. Wait a few seconds, then run`docker-compose down`
1. Ensure some critical file permissions are correct:

`sudo chown -R 70:70 ./postgres`

`sudo chown -R 991:991 ./public`

## Stand up the mastodon stack
`sudo docker-compose up -d`

## Add NPM to your `docker-compose` stack

1. Run `date | md5sum` twice to generate two new passwords: one for the NPM MySQL root account and one for the database
1. Edit the `docker-compose.yml` with `vi` or `nano`, whichever you prefer
1. Copy and paste the content below to add `NPM` immediately above the `networks:` section at the end of the `services:` section in your `docker-compose.yml`. Ensure the `volumes:` section is fully outdented to the first column.

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

1. Save and exit
1. Run `docker-compose up -d` to add `NPM` services to your stack
1. Run the `exit` command to logout of the server instance
1. Run `ssh` to log back in, but add `-L 8082:localhost:81` to the end of the ssh command line. This will let you configure `NPM` via the web UI.
1. Pull up `http://localhost:8082` in your browser
1. Log into NPM using `admin@example.com` and `changeme` as the credentials
1. Update the registration fields with your name, nickname, and a strong password

## Get an SSL certificate from Let's Encrypt
### Get an API key from Cloudflare
1. In the Cloudflare tab in your browser, click the person icon in the top right to bring up your Account menu
1. Click on `My Profile`
1. Click `API Tokens` on the left
1. Click `Create Token` on the right
1. Click `Use template` next to `Edit Zone DNS` (the top menu item)
1. In the top section `Permissions`, click `+ Add more`
1. Drop down `Account` and choose `Zone`
1. Drop down the menu with `Select item` and choose `Zone`
1. Drop down `Select` and choose `Read`
1. In the next section `Zone Resources`, click `Select` and choose your domain name
1. Scroll down to the bottom of the page and click on `Start date` to bring up the calendar
1. Click on today's date for the start date and a date far out into the future for the end date
1. Click `Continue to summary`
1. Click `Create token` on the next page
1. Copy the token shown

### Get an SSL Certificate in NPM
1. In your Nginx Proxy Manager browser tab, go to the `SSL Certificates` section
1. Click the `Add SSL Certificate` button
1. Choose `Let's Encrypt` from the dropdown menu
1. Enter `mstdn.<yourdomain.tld>` for the domain name, replacing `<yourdomain.tld>` with the domain you bought
1. Turn on the `Use a DNS Challenge` slider
1. Choose `Cloudflare` from the dropdown list of providers
1. Replace the sample API token shown with the API token displayed to you on the Cloudflare API token screen
1. Click the slider at the bottom to indicate you agree with the Terms and Conditions
1. Click the `Save` button at the bottom
1. It should take less than a minute to receive your certificate

### Add a Proxy host for `web`
1. Click `Dashboard` to return to your NPM Dashboard view
1. Click on the `Proxy hosts` button on the left
1. Click the `Add proxy host` button at the right end of the menu
1. Enter `mstdn.<yourdomain.tld>` in the domain name field, replacing `<yourdomain.tld>` with the domain you bought Note: You *must* click on the name that you just typed as it appears in the dropdown list to lock it into the field. Do NOT tab out of that field without clicking it.
1. Enter `web` in the `Forward Hostname/IP` field and `3000` in the `Forward port` field
1. Enable `Block Common Exploits`
1. Click the `SSL` tab up top
1. Click `None` under `SSL Certificate` and choose the domain name you entered on the Details tab (i.e. `mstdn.<yourdomain.tld>`)
1. Enable `Force SSL`, `HTTP/2 Support`, and `HSTS Enabled`
1. Click the `Save` button

## Log into your admin account
1. Point your browser to `mstdn.<yourdomain.tld>`
1. You should see your main Mastodon screen
1. Log into mastodon with your admin id and password that you saved before

## Create your 'normal user' account
1. Click `Preferences` at the bottom of the menu on the right
1. Click `Invite people` in the menu on the left
1. Generate an invite link on the right and copy it into your clipboard
1. Open a private browser window
1. Paste the invite link to bring up the signup page
1. Fill in the details for your non-root user; remember your password
1. You will see a message to look for a verification email; you will not get one (unless you setup email earlier)
1. Return to your `ssh` shell
1. `sudo docker exec -it mastodon-streaming-1 /bin/bash`
1. At the container command prompt, run `RAILS_ENV=production bin/tootctl accounts modify <your non-root username> --confirm`
1. (look for `OK`) in return

## Verify your normal account works
1. Go back to your private browser window
1. Edit the URL to return to `https://mstdn.<yourdomain.tld>` replacing `<yourdomain.tld>` with your actual domain
1. You should already be logged in
1. Click the 3 vertical dots next to your username and choose logout at the bottom
1. Verify that you can log back in with your non-root username/password
