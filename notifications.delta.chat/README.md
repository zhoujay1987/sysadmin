# notifications.delta.chat

(Originally notifications.testrun.org)

Authors: missytake@systemli.org and janek@merlinux.eu

This service is running on the testrun.org machine. It's purpose is to wake up
Delta Chat iOS clients regularly with push notifications, so they can get Delta
messages in the background.

## DNS Entry

In dns.hetzner.com `notifications.testrun.org` was added as subdomain and
points to the usual  A/AAAA record.

## Create User

First we added a new user with `sudo adduser notifications` and stored the
password in `bg-delachat/secrets/notifications.testrun.org`. We logged in with
`sudo su -l notifications`.

## Install Rust-toolchain

As the notifications user, we could install rustup, even without privileges:
`curl -sf -L https://static.rust-lang.org/rustup.sh | sh`

Then we logged out and in again, to update PATH. Now running `cargo` produced a
help page.

## Clone git repository

Now we could clone the repository and build the project:

```
git clone https://github.com/dignifiedquire/notifiers
cd notifiers
cargo build --release
```

## Save Cert

We saved the cert at /home/notifications/cert.pem, with the following
permissions:

```
-r-------- 1 notifications notifications 1,5K Jan 12 18:36 aps.cer
```

This certificate is valid until 2021-08-28, then it needs to be replaced.

## Systemd service

Then we wrote a systemd service file at /etc/systemd/system/notifications.service.
It contains the password for the certificate, so we didn't want it to be
readable by anyone else but root and notifications: 
`sudo chmod 640 /etc/systemd/system/notifications.service`

It contains the following configuration:

```
[Unit]
Description=Notification Service
After=syslog.target network.target
StartLimitIntervalSec=300
StartLimitBurst=3

[Service]
WorkingDirectory=/home/notifications
ExecStart=/home/notifications/notifiers/target/release/notifiers --certificate-file /home/notifications/aps.cer --password ""
Restart=unless-stopped
RestartSec=60
KillSignal=SIGQUIT
Type=simple
StandardError=syslog
NotifyAccess=all
User=notifications
Group=notifications

[Install]
WantedBy=multi-user.target
```

To activate the service, we ran:

```
sudo systemctl daemon-reload
sudo systemctl start notifications.service
```

## NGINX Config

We added a file at `/etc/nginx/sites-enabled/notifications` with the following
content:

```
server {
        server_name notifications.testrun.org;

        location / {
                proxy_pass http://127.0.0.1:9000;
                proxy_http_version 1.1;
        }

        listen [::]:443 ssl ipv6only=on ;
        listen 443 ssl ; 
        ssl_certificate /etc/dehydrated/certs/notifications.testrun.org/fullchain.pem;
        ssl_certificate_key /etc/dehydrated/certs/notifications.testrun.org/privkey.pem;

}

server {
        if ($host = notifications.testrun.org) {
            return 301 https://$host$request_uri;
        }

        listen 80 ;
        listen [::]:80 ;
        server_name notifications.testrun.org;
        return 404;

}
```

We needed to add `proxy_http_version 1.1` because the notifie.rs doesn't speak
HTTP 1.0.

## Let's Encrypt

First we needed to comment out the upper server block, so `sudo service nginx
reload` didn't throw an error.

Then we appended `notifications.testrun.org` to `/etc/dehydrated/domains.txt`,
and ran `sudo dehydrated -c` to generate an extra TLS certificate for
notifications.testrun.org. We readded the server block for the ssl, reloaded
nginx, and tested it with `curl -Li http://notifications.testrun.org`. The
results were as expected.

## Commit to etckeeper

Finally we committed the changes to etckeeper with `etckeeper commit "Setting
up notifications.testrun.org". The user creation is not included in this commit
though.

## Changed domain to notifications.delta.chat

On 2021-01-15 we realized that it makes more sense to run this service as
notifications.delta.chat, as it actually *is* a centralized service, and can't
be decentralized (without Apple changing their notification delivery scheme).

### DNS Changes

Author: missytake@systemli.org

So first I created notifications.delta.chat A and AAAA records:

```
A	notifications	176.9.92.144		3600
AAAA	notifications	2a01:4f8:151:338c::2	3600
```

Then deleted the A & AAAA records for notifications.testrun.org and instead
created a CNAME record pointing to notifications.delta.chat:

```
CNAME notifications	notifications.delta.chat	86400
```

### Change NGINX configuration

Then I changed the NGINX configuration:

```
sudo mv /etc/nginx/sites-enabled/notifications /etc/nginx/sites-available/notifications.delta.chat
sudo vim /etc/nginx/sites-available/notifications.delta.chat
```

I changed every mention of notifications.testrun.org to
notifications.delta.chat. Then I commented out the `ssl_certificate` and
`ssl_certificate_key` lines, because nginx would fail to reload before I
changed the dehydrated/Let's Encrypt configuration.

Now I activated the new configuration and reloaded the service, so I could
continue with Let's Encrypt:

```
sudo ln -s /etc/nginx/sites-available/notifications.delta.chat /etc/nginx/sites-enabled/notifications.delta.chat
sudo service nginx reload
```

### Create new Let's Encrypt certificate

Then I changed `notifications.testrun.org` to `notifications.delta.chat` in
`/etc/dehydrated/domains.txt`, and ran `sudo dehydrated -c` to generate an
extra TLS certificate for notifications.delta.chat. I readded the
`ssl_certificate` and `ssl_certificate_key` lines to the nginx config, reloaded
nginx, and tested it with `curl -Li http://notifications.delta.chat`. The
results were as expected.

Finally I committed the changes to etckeeper with `sudo etckeeper commit
"Changed notifications.testrun.org to notifications.delta.chat"`.