---
title: "Installing an Akkoma Instance"
date: 2023-01-15T17:29:56-08:00
draft: false
---

I've decided to self-host my own [Akkoma](https://docs.akkoma.dev/stable/) (a hard fork of [Pleroma](https://pleroma.social/)) instance, after actively and happily using Mastodon for 2.5 years. Inspired by [The Teabag Ninja's awesome and detailed writeup on Akkoma's OTP install](https://the.teabag.ninja/posts/2022/11/installing-akkoma-on-your-host/), I'm also sharing my source installation notes along the way for all the folks on the internet, including my future self -- since I did have a few surprises during the process.

Unless otherwise noted, all commands below are tested on an ARM-based machine with Ubuntu 22.04, for an Akkoma install from source. 

# A Word of Warning: Hosting the Instance on a (Sub)domain With a Different Account Identifier
I'm putting this down first, because this burnt me so bad, that I had to tear down the entire installation, and re-install the instance under a different subdomain. Read this if you're like me, who:
- already hosts content on `domain.com`, 
- is thinking of hosting the instance under `subdomain.domain.com`, 
- while ensuring your username has the form `@user@domain.com` instead of `@user@subdomain.domain.com`.

TL;DR: this is achievable on Akkoma (or any fediverse software that supports this), but with a few very important caveats:
- You need to have the ability to set up multiple external 301 redirects for `domain.com`. The Pleroma/Akkoma doc [explains why with the WebFinger protocol](https://docs.akkoma.dev/stable/configuration/how_to_serve_another_domain_for_webfinger/), but turns out you will have to proxy/redirect not only `.well-known/host-meta`, but **all three** different well-known endpoints([reference](https://aeracode.org/2022/11/01/fediverse-custom-domains/)): `host-meta`, `nodeinfo`, and `webfinger`. Especially the `webfinger` endpoint mentioned in the docs of [Mastodon](https://docs.joinmastodon.org/admin/config/#web_domain) and [GoToSocial](https://docs.gotosocial.org/en/latest/installation_guide/advanced/#can-i-host-my-instance-at-fediexampleorg-but-have-just-exampleorg-in-my-username) -- which Akkoma/Pleroma docs seem to be missing.
- You have **only one chance** to configure this right -- before your instance starts federating with other instances. Otherwise, the other instaces will be confused about the identity of users on your instance, and there's no way to ask them to correct it -- because you don't control those other instances.

Since my contents on `domain.com` is hosted on Gitlab Pages, which per [their instructions](https://docs.gitlab.com/ee/user/project/pages/redirects.html) does not support external redirects (yet), here are my alternatives:
- Serving static assets for `host-meta`, `nodeinfo` (these two are just static content), and each user on my instance for `webfinger`(this endpoint has query parameters), so that the lookup finds them.
- Just settle for identifiers with the subdomain like `@user@subdomain.domain.com`. (I eventually went for this this option.)

Just so that you mess things up and have to start from scratch like I did, [here](https://johnmee.com/how-to-reinstall-postgresql-on-ubuntu)'s how you uninstall and reinstall Postgres on Ubuntu 🥲.

# Setting Up a Host Server

#### With Oracle Cloud Infrastructure

My final setup is on an ARM-based OCI instance on their generous [free tier offering](https://www.oracle.com/cloud/free/), with the acknowledgement/expectation that this free tier may change in the future. To setup an account and create an instance, I follow [this tutorial](https://www.seenlyst.com/blog/oracle-cloud-free-vps-out-of-capacity/) -- admittedly including the script that, once in a while, checks if an instance is available in my home region. I'm lucky to have an instance created in about a week for my region.

Once the instance is created and a public IP address attached, I have no issues logging in with the ssh key. The default user is `ubuntu`, but running `sudo <command>` or switching to root (`sudo su - root`) does not require a password -- this is important because the super user permission is required to install Akkoma on the server.

OCI's Ubuntu instances are firewall protected by both the OS's iptables, and the VCN attached to the instance (we'll see later), so don't be surprised if a fresh instance doesn't allow most ingress TCP connections. 

#### With Vultr

I also get an instance with [Vultr](https://vultr.com) -- they run year-round promos, allowing you to try out their instances for free for some initial periods of time. Akkoma's resource consumption is minimal, so a humble $5/month instance with 1 vCPU, 1GB memory, and 1T bandwidth should be able to handle myself as the sole user of the app.

Once you have an account registered, select Cloud Compute -> Regular Performance -> 25GB SSD. Choose a location closest to you, and give the server a hostname. An SSH key pair can be [generated and added](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) from my Mac:
- Generate with `ssh-keygen -t ed25519 -C "email@example.com"`, and
- add to ssh-agent with `ssh-add -K ~/.ssh/<name_of_private_key_file>`.

Also make sure to disable IPv6 and Auto Backups to avoid additional charges of $1/month.

Vultr's Ubuntu instances are initially firewalled with `ufw` -- the Ubuntu OS firewall. Initially only port 22 are allowed:
```console
root@server:~# ufw status
Status: active

To                         Action      From
--                         ------      ----
22                         ALLOW       Anywhere                  
22 (v6)                    ALLOW       Anywhere (v6) 
```
To enable port 80 and 443 for ingress connections, `ufw allow 80/tcp` and `ufw allow 43/tcp`, then reload with `ufw reload`.

#### Setting Up DNS for the Subdomain
It's good to start setting up the DNS now, since the records usually take a while to propagate because of delays with your DNS server, TTL, caches everywhere etc. 

I'm trying to set up my Akkoma instance on a subdomain `fedi.<domain>.com`, where `<domain>.com` already hosts my blog on Gitlab Pages. So the subdomain `fedi.<domain>.com` should point to my server. [Vultr's official DNS doc](https://www.vultr.com/docs/introduction-to-vultr-dns) isn't helpful on the subdomain issue, in that it directs you to use its own nameservers regardless of alternatives (OCI also has DNS servers available) -- but in my case, I'm already using DNS at my domain registar for `<domain>.com`. So I end up directly going to my domain registar, and adding my `A` record for `Hostname:akkoma, Type:A, value:<my_instance_ip>`. This is it!

Like all DNS configurations, this takes a while to take effect. After a couple of hours, I'm able to verify with `nslookup fedi.<domain>.com`, which returns my instance's IP.


# Installing Akkoma (From Source)
On the OCI instance, I was following [the OTP guide](https://docs.akkoma.dev/stable/installation/otp_en/) until the config generation step gave me:
```console
$ su akkoma -s $SHELL -lc "./bin/pleroma_ctl instance gen --output /etc/akkoma/config.exs --output-psql /tmp/setup_db.psql"
/opt/akkoma/releases/3.5.0-0-gd9508474b/../../erts-13.1.2/bin/erl: 12: exec: /opt/akkoma/erts-13.1.2/bin/erlexec: Exec format error
```
This is when I realized [Akkoma doesn't support ARM builds yet at the time of writing](https://akkoma.dev/AkkomaGang/akkoma/issues/424) -- but unfortunately I'm on an ARM-based machine:
```console
$ uname -m
aarch64
```
This means I'll have to install from source! All of the following commands are derived from the official Akkoma documentation, on installing from source, for [Arch Linux](https://docs.akkoma.dev/stable/installation/arch_linux_en/), and [Alphine Linux](https://docs.akkoma.dev/stable/installation/alpine_linux_en), adapted for Ubuntu.


#### Installing Dependencies
1. Switch to the root user: 
    ```console
    sudo su - root
    ```
2. Install the required deps:
    ```console
    apt install postgresql elixir git cmake file build-essential erlang
    ```
3. Install the optional deps:
    ```console
    apt install imagemagick ffmpeg libimage-exiftool-perl
    ```
4. Start the Postgres service:
    ```console
    systemctl start postgresql.service
    ```

#### The Akkoma Backend
1. Create the akkoma user if not already (`less /etc/passwd` to check): 
    ```console
    adduser --system --shell  /bin/false --home /opt/akkoma akkoma
    ```
    I took this from the OTP install guide, so that the home directory is `/opt/akkoma` instead.
2. Make the akkoma user the owner of `/opt/akkoma`, and clone the stable branch of akkoma code into it: 
    ```console
    mkdir -p /opt/akkoma
    chown -R akkoma /opt/akkoma
    su akkoma -s $SHELL -lc "git clone https://akkoma.dev/AkkomaGang/akkoma.git -b stable /opt/akkoma"
    ```
3. Change into the directory:
    ```console
    cd /opt/akkoma
    ```
4. Install akkoma dependencies -- say yes when you're asked to install `Hex`.
    ```console
    su akkoma -s $SHELL -lc "mix deps.get"
    ```
5. Generate configurations in `config/generated_config.exs`: 
    ```console
    su akkoma -s $SHELL -lc "MIX_ENV=prod mix pleroma.instance gen"
    ```
    - As the offical doc explains, this step also compiles parts of akkoma, so it could take a bit while.
    - Say yes when you're asked to install `rebar3`.
    - It also interactively asks a few questions. Pay extra attention to the domain that the instance uses since it's bad to change it later. Also I said no to storing the configuration in the database -- I'll enable it later. (Well, almost all the options can be changed later.) Additionally I used the OTP locations for uploads (`/var/lib/akkoma/uploads`) and static (`/var/lib/akkoma/static`) instead of the default.
6. Rename the config file to `/opt/akkoma/config/prod.secret.exs`: 
    ```console
    su akkoma -s $SHELL -lc "mv config/{generated_config.exs,prod.secret.exs}"
    ```
7. Create the database:
    ```console
    su postgres -s $SHELL -lc "psql -f /opt/akkoma/config/setup_db.psql"
    ```
8. Run the db migration -- this compiles more files so again it takes a while.
    ```console
    su akkoma -s $SHELL -lc "MIX_ENV=prod mix ecto.migrate"
    ```
9. Finally start the Akkoma backend: 
    ```console
    su akkoma -s $SHELL -lc "MIX_ENV=prod mix phx.server"
    ```
    From another terminal (ssh'ed into my server) I'm able to `curl http://localhost:4000/api/v1/instance`, and get back some json. Sweet! 

#### A Quick Summary of Where Things Are So Far
- The config: `/opt/akkoma/config/prod.secret.exs`
- The code: `/opt/akkoma` -- this is also system user `akkoma`'s home directory
- The uploads directory: `/var/lib/akkoma/uploads`
- The static directory (custom emojis, frontend bundle overrides, robots.txt, etc.): `/var/lib/akkoma/static`  

Moving on...

#### Reverse Proxy and TLS Handling
Any reverse proxy will do (the repo actually has a few other options in `/opt/akkoma/installation`), but I'll stick with nginx for now 😃 
1. Stop nginx ot free up port 80: 
    ```console
    systemctl stop nginx
    ``` 
    And verify port 80 is clear with `netstat -ltpn`.
2. Get a Let's Encrypt certificate: 
    ```console
    certbot certonly --standalone --preferred-challenges http -d fedi.<domain>.com
    ```
    On the OCI ARM instance, this step the command initially failed for me on connection problems:
    ```console
    Certbot failed to authenticate some domains (authenticator: standalone). The Certificate Authority reported these problems:
    Domain: fedi.<domain>.com
    Type:   connection
    Detail: <ip>: Fetching http://fedi.<domain>.com/.well-known/acme-challenge/9JJYxsrHYt-aD86w9Mlp0k4JZajbEGlL1eSSXHkG7c8: Timeout during connect (likely firewall problem)
    ```
    An nmap scan (`nmap -Pn -p 22,80,443 --reason <ip>` from outside the server) also shows ports 80 and 443 as "filtered":
    ```console
    PORT     STATE    SERVICE        REASON
    22/tcp   open     ssh            syn-ack
    80/tcp   filtered http           no-response
    443/tcp  filtered https          no-response
    ```
    "Filtered" just means there's a firewall blocking the port. Turns out two firewalls are blocking the connection -- the host firewall and OCI's Virtual Cloud Networks setup. 

    - I follow [this article](https://blogs.oracle.com/developers/post/enabling-network-traffic-to-ubuntu-images-in-oracle-cloud-infrastructure) to enable 80 and 443 for the host firewall -- essentially manipulate the iptable rules directly. 
    - On how to add the ingress rule to the VCN that my instance belongs to, credit to [this article](https://medium.com/@fathi.ria/oracle-database-cloud-open-ports-on-oci-1af24f4eb9f2). 
    
    After the two steps, rerunning this command succeeds for me.

    I also notice this line from stdout when I request the certificate:
    ```console
    This certificate expires on 2023-xx-xx.
    These files will be updated when the certificate renews.
    Certbot has set up a scheduled task to automatically renew this certificate in the background.
    ```
    So [it did](https://community.letsencrypt.org/t/what-does-this-mean-exactly-certbot-has-set-up-a-scheduled-task-to-automatically-renew-this-certificate-in-the-background/174615), which means I don't have to manually set up auto-renew. Neat!
3. Copying the nginx config and enabling it:
    ```console
    cp /opt/akkoma/installation/nginx/akkoma.nginx /etc/nginx/sites-available/akkoma.conf
    ln -s /etc/nginx/sites-available/akkoma.conf /etc/nginx/sites-enabled/akkoma.conf
    ```
4.  ```console
    nano /etc/nginx/sites-available/akkoma.conf
    ``` 
    And replace all occurrences of `example.tld` to `fedi.<domain>.com`.
5. Verify the config with 
    ```console
    nginx -t
    ```
6. Start nginx service, and optionally Akkoma backend in the foreground with 
    ```console
    systemctl start nginx
    su akkoma -s $SHELL -lc "MIX_ENV=prod mix phx.server"
    ```
    Pointing my browser at `https://fedi.<domain>.com`, I get:
    ```console
    Welcome to Akkoma!
    If you're seeing this page, your server works!
    ...
    ```
    with some more instructions and links to install a frontend. Yay!
7. Now to set up the Akkoma backend as a system service. Looking at the the service config file, the only thing I need to modify this the akkoma user's home directory. So 
    ```console
    nano /opt/akkoma/installation/akkoma.service
    ```
    and change this line `Environment="HOME=/var/lib/akkoma"` to:
    ```console
    Environment="HOME=/opt/akkoma"
    ```
8. Copy the service config file, start it and enable it on boot:
    ```console
    cp /opt/akkoma/installation/akkoma.service /etc/systemd/system/akkoma.service
    systemctl start akkoma
    systemctl enable akkoma
    systemctl restart nginx
    ```
    Pointing the browser to `https://fedi.<domain>.com` again, I get the same instructions to install a frontend.

#### (Optional) Setting Up Custom Domain Identifiers
If you're using a custom (sub)domain and want the usernames to be identified differently from the instance (sub)domain, this is when you want to set up two things:
- the Akkoma config ([reference](https://docs.akkoma.dev/stable/configuration/how_to_serve_another_domain_for_webfinger/)), and
- the redirects for WebFinger related endpoints ([reference](https://aeracode.org/2022/11/01/fediverse-custom-domains/)).  

Make sure to get it right **now** before your instance starts federating! 

#### Installing A Frontend
1. From what I heard, `pleroma-fe` is a light-weight minimalist frontend. I'll use it to play around for now:
    ```console
    cd /opt/akkoma
    su akkoma -s $SHELL -lc "MIX_ENV=prod mix pleroma.frontend install pleroma-fe --ref stable"
    ```
    Says it's installed to `/var/lib/akkoma/static/frontends/pleroma-fe/stable`.
2. And now install `admin-fe`:
    ```console
    su akkoma -s $SHELL -lc "MIX_ENV=prod mix pleroma.frontend install admin-fe --ref stable"
    ```
3. Since we now have a frontend to do administration with, let's not forget to [activate in-database configuration](https://docs.akkoma.dev/stable/configuration/howto_database_config/) -- I initially opted out during the config generation step.
    - ```console
      nano config/prod.secret.exs
      ```
      And set `config :pleroma, configurable_from_database: true` (it's one of the last lines on the very bottom).
    - ```console
      su akkoma -s $SHELL -lc "MIX_ENV=prod mix pleroma.config migrate_to_db"
      ```
      This also triggers some compiling.
    - ```console
      systemctl restart akkoma
      ```
      so that the change takes effect.


#### Creating the admin user
```console
su akkoma -s $SHELL -lc "MIX_ENV=prod mix pleroma.user new <username> <your@emailaddress> --admin"
```
The command prints a URL to reset the password -- copy and pasting it to the browser, and logging in -- it works! Finally time to play around with some fun customizations 🎉.

# Quick Validations
- Federation: I have another account on a Mastodon instance, and I try to have the two accounts follow each other and send direct messages -- to see if they get each other semi-instantly. During my first attempt at the custom subdomain setup, I was able to uncover the federation issue this way. 
- I notice that for the remote accounts I follow, my instance initially pulls posts from back a few months or even a year ago -- but as my followees generate new contents, the new posts are up to date.
