+++
date = '2026-03-10T19:41:00+01:00'
draft = false
title = 'My Minimal Git Hosting Setup'
+++

This tutorial will show you how to get a basic git server with static HTML pages going for your repos. 

# Setup git on your server


## Install git

```bash
apt install git
```

## Create a git user
```bash
useradd -m git -d /var/git -s /bin/bash
```

## Set up SSH login for the git user
This assumes you will use the same key you use to log in to the server to push to git repositories. These commands will crate the required directory, copy over the authorized keys from your current user and set proper ownership of the directory to the git user.
```bash
mkdir /var/git/.ssh
cp ~/.ssh/authorized_keys /var/git/.ssh/
chown git:git -R /var/git/.ssh
```
To check the setup run:
```bash
ssh git@git.example.com
```

## Create your first repo
Per convention we will create the repo with the .git ending.
Login as the git user and type:
```bash
git init --bare repo-name.git
```

# Setup stagit

## Install dependencies
```bash
apt install libgit2-dev gcc make git
```
Clone the stagit repo and build it:
```bash
git clone git://git.codemadness.org/stagit
cd stagit
make
make install
```

## Create the Generation script
This script will be used to create the static files for each repo. Save it to <code>/usr/local/bin/stagit-gen</code>
```bash
#!/bin/sh
REPODIR="/var/git"
WEBROOT="/var/www/git"

for repo in "$REPODIR"/*.git; do
    dest="$WEBROOT/$(basename "$repo" .git)"
    mkdir -p "$dest"
    ln -sf ../style.css "$dest/style.css"
    ln -sf ../logo.png "$dest/logo.png"
    ln -sf ../favicon.png "$dest/favicon.png"
    cd "$dest" && stagit "$repo"
done

stagit-index "$REPODIR"/*.git > "$WEBROOT/index.html
```

Make it executable:
```bash
chmod +x /usr/local/bin/stagit-gen
```

Copy over the standard style.css:
```bash
cp /usr/local/share/doc/stagit/style.css /var/www/git/style.css
```

# Setup the repo
We will use <code>repo-name</code> that we created earlier.

## Set Up Post-Receive Hooks
This makes the site regenerate automatically on every push.
```bash
ln -sf /usr/local/bin/stagit-gen /var/git/repo-name/hooks/post-receive
```
You will need to to this manually for each new repo.

## Setup repo metadata
```bash
echo "My project description"   > /var/git/myrepo.git/description
echo "Your Name"                > /var/git/myrepo.git/owner
echo "https://example.org"   > /var/git/myrepo.git/url
```

## Generate the site
```bash
stagit-gen
```

Check the output:
```sh
ls /var/www/git
ls /var/www/git/repo-name
```

# Enable http cloning
Install <code>fastcgiwrap</code> to enable anonymous <code>git clone https://example.com/repo-name.git</code> over HTTPS:
```sh
apt install fcgiwrap
systemctl enable --now fcgiwrap
```

## Setup the service to run as the git user
```sh
systemctl edit fcgiwrap
```
Add:
```ini
[Service]
User=git
Group=git
```
Restart:
```sh
systemctl restart fcgiwrap
```

## Enable cloning per repo
```sh
touch /var/git/repo-name.git/git-daemon-export-ok
```

# Configure nginx
```bash
server {
    listen 443 ssl;
    listen [::]:443 ssl ipv6only=on;
    server_name git.example.org;

    ssl_certificate     /etc/letsencrypt/live/git.example.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/git.example.org/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    root  /var/www/git;
    index index.html;

    # Static stagit pages
    location / {
        try_files $uri $uri/ =404;
    }

    # Git HTTP clone (requires fcgiwrap)
    location ~ ^/[^/]+\.git(/.*)?$ {
        root /var/git;
        fastcgi_pass unix:/var/run/fcgiwrap.socket;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME /usr/lib/git-core/git-http-backend;
        fastcgi_param GIT_HTTP_EXPORT_ALL "";
        fastcgi_param GIT_PROJECT_ROOT /var/git;
        fastcgi_param PATH_INFO $uri;
    }
}

server {
    listen 80;
    listen [::]:80;
    server_name git.example.org;
    return 301 https://$host$request_uri;
}
```

## Link nginx
```bash
ln -s /etc/nginx/sites-available/git /etc/nginx/sites-enabled/
```

Reload
```sh
systemctl reload nginx
```

# Add a logo to the site
The logo for stagit is saved as `/var/www/git/logo.png`.
A 32x32 png is sufficient. The favicon of the site is `favicon.png` in the same dir.

Transfer the logo to the server:
```sh
scp logo.png git.example.org:
```

Copy it to the required dir and fix permissions:
```sh
sudo mv logo.png /var/www/git/
sudo cp /var/www/git/logo.png /var/www/git/favicon.png
sudo chown -R git:git /var/www/git/
```

