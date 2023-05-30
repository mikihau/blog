---
title: "Operating Akkoma"
date: 2023-05-15T12:02:48-07:00
draft: false
tags: ["how-to", "self hosting", "ubuntu", "operations"]
series: "Akkoma"
aliases:
- /post/operating-akkoma
---

A couple of months into [self-hosting my Akkoma instance]({{< ref "self-hosting-akkoma.md" >}}), I find myself doing a couple of operational tasks at a recurring basis. So might as well write them down here for future reference. Again, this is on my ARM-based machine with Ubuntu 22.04, where Akkoma is installed from source.

## Backing Up (Periodically)
The [doc's backup instructions](https://docs.akkoma.dev/stable/administration/backup/) essentially says we should back up the database, plus a couple of configs and static files/directories.

So first off, ensure we have a place to put the backup files by creating this backup directory:
```console
sudo su - root
mkdir -p /mnt/backups/akkoma
chown -R akkoma /mnt/backups/akkoma
chmod o=rw /mnt/backups/akkoma/
```
Then I whip up a quick script to back up all things mentioned in the doc:
```console
#!/usr/bin/env bash

echo "Akkoma backup starting"

BACKUP_DIR=/mnt/backups/akkoma/$(date -I)
mkdir -p $BACKUP_DIR/config

echo "Stopping akkoma instance"
systemctl stop akkoma

echo "Backing up postgres" 
sudo -Hiu postgres pg_dump -d akkoma --format=custom -f /tmp/akkoma.pgdump
mv /tmp/akkoma.pgdump $BACKUP_DIR/akkoma.pgdump

echo "Copying config and static files"
cp /opt/akkoma/config/prod.secret.exs $BACKUP_DIR/config/prod.secret.exs
cp /opt/akkoma/config/setup_db.psql $BACKUP_DIR/config/setup_db.psql
cp -r /var/lib/akkoma/uploads $BACKUP_DIR
cp -r /var/lib/akkoma/static $BACKUP_DIR

echo "Restarting akkoma instance"
systemctl start akkoma

echo "Akkoma backup finished at $BACKUP_DIR"
```
I put this script at `/usr/local/bin/akkomabackup`. 

My goal is to create a backup once per day, so I also set up a systemd timer to run the backup script periodically. Steps:
1. Create the backup service file as `/etc/systemd/system/akkoma-backup.service`:
   ```console
   [Unit]
   Description="Backing up my Akkoma instance"

   [Service]
   ExecStart=/usr/local/bin/akkomabackup
   ```
2. Create the timer file as `/etc/systemd/system/akkoma-backup.timer`:
   ```console
   [Unit]
   Description="Run akkoma-backup.service 5min after boot and every 24 hours relative to activation time"
   
   [Timer]
   OnBootSec=5min
   OnUnitActiveSec=24h
   OnCalendar=Mon..Sun *-*-* 10:00:*
   Unit=akkoma-backup.service
   
   [Install]
   WantedBy=multi-user.target
   ```
3. Validate the setup with:
   ```console
   systemd-analyze verify /etc/systemd/system/akkoma-backup.*
   ```
4. Enable the timer:
   ```console
   systemctl start akkoma-backup.timer
   systemctl enable akkoma-backup.timer
   ```
5. Verify that the backup timer works:
   ```console
   systemctl status akkoma-backup.timer
   systemctl status akkoma-backup.service
   ```
   If the timer file ever gets changed after it starts, do a `systemctl daemon-reload` to reapply the change.

## Renewing That TLS Certificate
I thought my TLS certificate was going to be renewed automatically, but at one point near my cert expiration, I started getting emails prompting me to renew my cert. Turns out (with `systemctl status certbot.service`) the renewal has been consistently failing because the script needs port 80, which conflicts with nginx, who also uses port 80. So to manually renew the cert, just do:
```console
systemctl stop nginx # free up port 80
/usr/bin/certbot renew
systemctl start nginx
```
For an automated solution, I made an edit to `/lib/systemd/system/certbot.service` by adding
```console
ExecStartPre=systemctl stop nginx
ExecStartPost=systemctl start nginx
```
under `[Service]`. After running `systemctl daemon-reload`, the service starts to succeed. (This isn't perfect since every time the renewal script runs, we have a tiny bit of downtime. But hey.)

## Upgrading the Akkoma Instance
Upgrading Akkoma usually doesn't come with a big fanfare -- except that, starting v3.8.0, Akkoma requires an Elixir version 1.14 that's not provided by the `apt` package manager. This brings me some confusion since I'll have to figure out how to install Elixir 1.14, and ensure the systemd service uses that upgraded Elixir runtime.

1. Switch to the `akkoma` user:
   ```console
   sudo -su akkoma /bin/bash
   cd /opt/akkoma/
   ```
2. Download [asdf](https://asdf-vm.com/guide/getting-started.html), the recommended multi-version runtime manager:
   ```console
   git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.11.3 # latest stable at the time of writing
   ```
3. Install asdf:
   ```console
   # add to ~/.bashrc:
   echo ". $HOME/.asdf/asdf.sh" >> "$HOME"/.bashrc
   echo ". $HOME/.asdf/completions/asdf.bash" >> "$HOME"/.bashrc
   source ~/.bashrc
   # verify
   asdf --version
   ```
4. Install Elixir 1.14.2 as required by Akkoma v3.8.0:
   ```console
   # lock into the specific versions
   touch $HOME/.tool-versions
   echo "erlang 25.0" > $HOME/.tool-versions
   echo "elixir 1.14.2" >> $HOME/.tool-versions
   # install
   asdf plugin-add erlang
   asdf plugin-add elixir
   asdf install
   # verify with
   elixir -v # should be 1.14.2
   ```
5. Make sure systemd uses the proper elixir version by editing `/etc/systemd/system/akkoma.service` ([reference](https://meta.akkoma.dev/t/akkoma-compilation-issues-running-under-v3-8-0-with-elixir-under-asdf-and-user-running-under-systemd/442/8)). Two changes to this file are required:
   - Adding
   ```console
   Environment="PATH=/opt/akkoma/.asdf/shims:/opt/akkoma/.asdf/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
   ```
   - And changing the `ExecStart` directive to
   ```console
   ExecStart=/opt/akkoma/.asdf/shims/mix phx.server
   ```

Moving on to the real upgrading work, [the doc](https://docs.akkoma.dev/stable/administration/updating/) had it pretty nicely.

1. Get the code:
    ```console
    sudo -su akkoma # switch to user 'akkoma'
    cd /opt/akkoma/
    git fetch
    git checkout $(git tag -l | grep -v 'rc[0-9]*$' | sort -V | tail -n 1)
    ```
2. Get the dependencies, and compile the code:
    ```console
    export MIX_ENV=prod
    mix deps.get
    mix compile
    ```
3. Migrate the DB and install frontends:
    ```console
    exit # (OCI specific) go back to the ubuntu user
    sudo systemctl stop akkoma
    sudo -su akkoma
    MIX_ENV=prod mix ecto.migrate
    MIX_ENV=prod mix pleroma.frontend install pleroma-fe --ref stable
    MIX_ENV=prod mix pleroma.frontend install admin-fe --ref stable
    ```
4. Optionally verify that the new backend version works, by starting the app without systemd (then stop the app):
    ```console
    MIX_ENV=prod mix phx.server
    ```
5. If all goes well, restart the service with systemd:
    ```console
    exit # exit # (OCI specific) go back to the ubuntu user
    sudo systemctl start akkoma
    ```
6. Verify with
    ```console
    systemctl status akkoma
    # and
    journalctl -u akkoma.service -f
    ```

## Extras: Onboarding Another Admin
I'm really fortunate to have a volunteer sys admin for this Akkoma instance. so I'm making sure that they have root access on the server to be able to handle any incidents/issues. Not Akkoma related, just Ubuntu/Linux stuff.
1. Create a linux user account:
    ```console
    sudo adduser <username>
    ```
    It'll prompt to set up a password, and a couple of other information. (Look, it even has a Room Number field! Wow in the old days sys admins used to sit in _separate rooms_.)

2. Make sure the user can ssh into the server. I don't have password-based access to the server (well I can enable password-based access, but let's say I don't), so I'll just copy the public key manually:
    ```console
    sudo -su root
    sudo -su <username>
    mkdir -p ~/.ssh
    echo <copied_public_key> >> ~/.ssh/authorized_keys
    ```
    Now testing it out:
    ```console
    ssh -i <path/to/private/key> <username>@<host>
    ```
3. Grant root access to the user:
    ```console
    sudo -su root
    usermod -aG sudo dbxadmin # add to the sudo group
    ```
    Optionally, if you want the user to run sudo commands without getting prompted for a password:
    ```console
    visudo
    ```
    And add this line `<username> ALL=(ALL) NOPASSWD:ALL` to the bottom of the editor.

## Monitoring
Looking into monitoring solutions for hobby projects, I bumped into [the Monitoring Basics by Steve Mookie Kong](https://ultramookie.com/2023/03/monitoring-basics/). Really the article is more than I need now, but I like the idea of using third-party hosted services to monitor my instance -- self-hosted monitoring solutions runs the risk of being down themselves. For now I use [Better Uptime](https://betterstack.com/better-uptime) for external health pings + uptime dashboard, and [Datadog](https://www.datadoghq.com/) for internal metrics.
