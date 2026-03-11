+++
date = '2026-03-10T19:41:00+01:00'
draft = false
title = 'My Minimal Git Hosting Setup'
+++

In this tutorial will learn how to install "*your own GitHub*" on a Debian/Ubuntu server. By the end you will be able to clone repositories like you do on GitHub:
```bash
git clone git@github.com:repo-name.git
```
becomes
```bash
git clone git@git.example.org:repo-name.git
```

The tutorial will cover the following parts:
- Setup **git** on your server to manage repositories
- Compile and configure **stagit**, to render repositories as static html pages
- Setup **nginx** as the webserver
- Configure **fcgiwrap** for anonymous https cloning of repositories
- Add a logo to the **stgit** pages

If not mentioned differently all the commands should be ran as `root` or with `sudo`.

# Setup git on your server
At its core, a git server is just a server that provides SSH access for the git user and stores repositories as plain files.

## Install git
You probably have this already installed, if not:

```bash 
apt install git
```

## Create a git user
Create a git user with `/vaŕ/git/` as their home directory. This is also where the repositories will be stored.
```bash
useradd -m git -d /var/git -s /bin/bash
```

## Set up SSH login for the git user
This assumes you already have SSH login setup to your server and you want to use the same keys to push to git repositories. 
```bash
# Create the .ssh directory
mkdir /var/git/.ssh
# Copy the keys from your current user
cp ~/.ssh/authorized_keys /var/git/.ssh/
# Fix permissions
chown git:git -R /var/git/.ssh
```
To check the setup run:
```bash
ssh git@git.example.com
```

## Create your first repo
Create the repo with the `.git` ending. Login as the git user or change to it with `su -l git` and run:
```bash
git init --bare repo-name.git
```

## Pushing to the repo
To push an existing repo:
```bash
cd existing-repo
git remote set-url origin git@git.example.org:test-repo.git
git add .
git commit "commit message"
git push -u origin master
```

To create a completely new repo:

```bash
mkdir test-repo
cd test repo
git init
git add remote origin git@git.example.org:test-repo.git
echo "# README" > README.md
git add README.md
git commit "initial commit"
git push -u origin master
```

# Setup stagit
stagit is a simple static page generator for git repositories. You will need to build it from source and create a simple script to generate the html pages when a git repo is updated.

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
This script will be used to create the static files for each repo. Save it to <code>/usr/local/bin/stagit-gen</code>:
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

Copy over the default style.css:
```bash
cp /usr/local/share/doc/stagit/style.css /var/www/git/style.css
```

# Setup the repository
You might use the repository `repo-name` that you initialized earlier.

## Setup repository metadata
**stagit** expects the following metadata files  inside the repo: 
```bash
echo "My project description"   > /var/git/myrepo.git/description
echo "Your Name"                > /var/git/myrepo.git/owner
echo "https://example.org"   > /var/git/myrepo.git/url
```

## Set Up Post-Receive Hooks
Link the `stagit-gen` script to the `post-receive` hook of the repo. This will regenerate the corresponding html page automatically on every push.
```bash
ln -sf /usr/local/bin/stagit-gen /var/git/repo-name/hooks/post-receive
```
You will need to to this manually for each new repo.

## Generate the site
Change to the git user with `su -l git` and run the script:
```bash
stagit-gen
```

Check the output:
```sh
ls /var/www/git
ls /var/www/git/repo-name
```

# Configure nginx
Probably also already installed, if not run:
```bash
apt install nginx
```

## Obtain a HTTPS certificate
To have our site available via HTTPS you need to obtain a certificate with `certbot`. Make sure to have `git.example.org` pointing to your servers IP.

Install certbot:
```bash
apt install python3-certbot-nginx
```

Request the certificate:
```bash
certbot certonly --standalone -d git.example.org
```

## Add the nginx configuration

Create `/etc/nginx/sites-available/git`:
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

Enable the site by linking it to `sites-enabled`:
```bash
ln -s /etc/nginx/sites-available/git /etc/nginx/sites-enabled/
```

Test the configuration:
```bash
nginx -t
```

Reload the configuration:
```sh
systemctl reload nginx
```

# (Optional) Enable https cloning
To enable anyone to clone your repos with `git clone https://git.example.orh/repo-name.git` you will need it install `fcgiwrap`:
```sh
apt install fcgiwrap
systemctl enable --now fcgiwrap
```

## Setup the service to run as the git user
For security reasons change the service to run as the git user:
```sh
systemctl edit fcgiwrap
```
Add:
```ini
### Editing /etc/systemd/system/fcgiwrap.service.d/override.conf
### Anything between here and the comment below will become the contents of the drop-in file

[Service]
User=git
Group=git

### Edits below this comment will be discarded
```
Restart:
```sh
systemctl restart fcgiwrap
```

## Enable cloning per repo
You will need to allow the cloning for each repo. To do that, just touch the `git-deamon-export-ok` file in the repo root:
```sh
touch /var/git/repo-name.git/git-daemon-export-ok
```


# Add a logo and favicon to the site
By default **stagit** has the logo and the favicon saved under `/var/www/git/`.

I used a 32x32 pixel png image.

Transfer the logo to the server:
```sh
scp logo.png git.example.org:
```

Copy it to the directory and fix permissions:
```sh
sudo mv logo.png /var/www/git/
# Re-use the logo as favicon
sudo cp /var/www/git/logo.png /var/www/git/favicon.png
sudo chown -R git:git /var/www/git/
```
