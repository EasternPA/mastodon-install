# Self-host Mastodon at home using CloudFlare Tunnels

## Note: This is a work-in-progress where I am documenting the process of figuring this out

You will need:
- a paid domain name that works with Let's Encrypt
- Migrate the DNS management of the domain to Cloudflare

Open the Cloudflare One dashboard, click Access, then Tunnels, and click create a tunnel

Name the tunnel `mastodon`.

Locate the id token in the install instructions. You don't need it yet, but make sure you can find it because you will need it later. Click Next at the bottom.

Enter `mstdn` as the public hostname and choose your Domain from the dropdown list in the Domain field.

Under Service / Type, choose `HTTP` and enter a temporary URL of `192.168.1.2:4000`. 

Now, open two ssh sessions on the target host

Select your primary session

Use these steps to locate the docker-compose installed with docker-compose and copy it into `/usr/local/bin`:

```
sudo su -
cd /snap/docker
find -name docker-compose | grep cli-plugins
cp /snap/docker/<x>/usr/libexec/docker/cli-plugins/docker-compose /usr/local/bin
```

Note that `<x>` will vary on each install

- Return to the portainer session in your web browser
- click Stacks from the main menu
- click `+ Add Stack`
- name it `mastodon` and copy this block to start a new portainer stack:

```
version: "3"
name: "mastodon"
volumes:
  pg:
  rd:
  publicsys:

services:

  cloudflared:
    image: cloudflare/cloudflared
    restart: unless-stopped
    command: tunnel run
    environment:
      - TUNNEL_TOKEN=${CFTOKEN}
#    networks:
#      - external_network
```

Note that the `networks:` block is commented out because the named network doesn't exist yet.

Create a variable in the variables section below the stack yml called `CFTOKEN` and copy/paste the Cloudflare tunnel token id for the tunnel you created earlier

Deploy the stack. This establishes the directory structure on the host.

Return to your primary ssh session

```
cd ~/Downloads
git clone https://github.com/mastodon/mastodon.git  
sudo su -
cd /var/snap/docker/common/var-lib-docker/volumes
find -name docker-compose
```

locate the docker-compose.yml representing the stack you are currently building in /var/snap/docker/common/var-lib-docker/volumes/<x>/_data/compose/<y>

where `<x>` is a very long hex string representing the docker volume and `<y>` is a small number representing which stack you are managing with portainer.

`cd` into the folder with the new stack's `docker-compose.yml`

`cat ~/Downloads/mastodon/docker-compose.yml >> docker-compose.yml`

Note: be sure to use the double chevron `>>` to append the new yaml content to the end of the existing file

open the updated `docker-compose.yml` for editing and locate the top of the content you just appended to the end of the file

delete the duplicate `version` and `services` lines you just copied in

uncomment the last two lines commented out in the `cloudflared` section attaching the container to the network hosting your mastodon website

leave the `es` section commented out; leave the `tor federation` lines commented out

save and exit the updated docker-compose.yml

resume the normal instructions from https://github.com/EasternPA/mastodon-install/blob/gh-pages/README.md

Skip down to:
https://github.com/EasternPA/mastodon-install/blob/gh-pages/README.md#initialize-the-database

Then follow the steps in  https://github.com/EasternPA/mastodon-install/blob/gh-pages/README.md#initialize-the-database

Skip the 'prepare the final build' section

Follow the steps in  https://github.com/EasternPA/mastodon-install/blob/gh-pages/README.md#initialize-the-database

Skip the steps to add NPM to your stack
Skip the steps to get a certificate from Let's Encrypt

Go to portainer and refresh the page showing your stack

Locate the internal IP address of your 'streaming' container in the stack (usually begins with 172., but may also be a 192.168. network you have not seen before, this is ok)

Pull up your Cloudflare Tunnels dashboard and click configure next to the tunnel you created

Click "public hostname" under the tunnel name on the main menu

Click "Edit" at the right end of your tunnel definition

Locate the URL field and replace 192.168.1.2 with the 172. (or 192.168) address that portainer showed for your 'streaming' container

Click Save hostname at the bottom

Confim on your tunnel configuration page that your public hostname 'mstdn.<yourdomain.tld> points to the service at 'http://172.x.y.z:4000' 

If all looks good, open a new browser tab and pull up the web site name you show in your tunnel. You should see a plain Mastodon web page, ready for login. Make sure that you see a closed lock to the left side of the URL for your web site and that no errors are shown relating to the SSL connection.

You will skip over all of the NPM configuration steps in the instructions, and resume the steps at https://github.com/EasternPA/mastodon-install/blob/gh-pages/README.md#create-your-admin-account

Then, once you have created (and successfully tested logging out AND IN using your Admin credentials), move on to creating a non-Admin userid here https://github.com/EasternPA/mastodon-install/blob/gh-pages/README.md#create-your-normal-user-account

Log out of your Admin id and log in using your non-Admin userid. Do not use your instance as the Admin user, this is a bad practice. Using your non-Admin userid limits the scope of what is at risk if an attacker compromises your session.

Finally, you should always keep your OS image up to date and also periodically update the docker image running Mastodon. In Portainer, pull up the stack. Select the entire stack and click Stop. Do NOT click Remove with the containers selected and do NOT click "Delete stack" up top.

Once the containers are stopped, click the Edit tab up top, then scroll down and click the blue "Update stack" button. That will check for new versions of the many docker images behind your instance without deleting any of the data you have created so far.
