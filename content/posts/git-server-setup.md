+++
date = '2026-03-10T19:41:00+01:00'
draft = false
title = 'How to setup your own Git Server'
+++

In this tutorial you will learn how to install "*your own GitHub*" on a Debian/Ubuntu server. By the end you will be able to clone repositories like you do on GitHub:
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
- Add a logo to the **stagit** pages

If not mentioned differently all the commands should be ran as `root` or with `sudo`.

## Setup git on your server
At its core, a git server is just a server that provides SSH access for the git user and stores repositories as plain files.

### Install git
You probably have this already installed, if not:

```bash 
apt install git
```

### Create a git user
Create a git user with `/vaŕ/git/` as their home directory. This is also where the repositories will be stored.
```bash
useradd -m git -d /var/git -s /bin/bash
```

### Set up SSH login for the git user
This assumes you already have SSH login setup to your server and you want to use the same keys to push to git repositories. 
```bash
## Create the .ssh directory
mkdir /var/git/.ssh
## Copy the keys from your current user
cp ~/.ssh/authorized_keys /var/git/.ssh/
## Fix permissions
chown git:git -R /var/git/.ssh
```
To check the setup run:
```bash
ssh git@git.example.com
```

### Create your first repository
Create the repo with the `.git` ending. Login as the git user or change to it with `su -l git` and run:
```bash
git init --bare repo-name.git
```

## Pushing to your repository
On your local machine:
```bash
## Create the directory and navigate into it
mkdir repo-name
cd test-repo
## Initialize a new local repository
git init
## Link the local repository to the remote
git remote add origin git@git.example.org:repo-name.git
## Create a README and make the initial commit
echo "# README" > README.md
git add README.md
git commit -m "initial commit"
## show the current branch
git branch --show-current
## Push and set the upstream tracking branch
git push -u origin master
```
The `-u` flag sets the upstream tracking branch, so all git push and git pull commands in the future will work without extra arguments. 

*If you are migrating an existing repository, complete the **stagit** [repository setup below](#setup-the-repository) before pushing.*

## Setup stagit
stagit is a simple static page generator for git repositories. You will need to build it from source and create a simple script to generate the html pages when a git repo is updated.

### Install dependencies
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

### Create the Generation script
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

stagit-index "$REPODIR"/*.git > "$WEBROOT/index.html"
```

Make it executable:
```bash
chmod +x /usr/local/bin/stagit-gen
```

Copy over the default style.css:
```bash
cp /usr/local/share/doc/stagit/style.css /var/www/git/style.css
```

## Setup the repository
These steps use the `repo-name` repository created earlier as an example.

### Setup repository metadata
**stagit** expects the following metadata files  inside the repo: 
```bash
echo "project description"                        > /var/git/myrepo.git/description
echo "Your Name"                                  > /var/git/myrepo.git/owner
echo "https://git.example.org/repo-name.git"      > /var/git/myrepo.git/url
```

### Adjust the repository branch if needed
When initializing a bare repo, git sets `HEAD` to `master` by default. If your commits are on a different branch, **stagit** will not find them since it reads from `HEAD` to find commits.

On your local machine, check which branch `HEAD` currently points to:
```bash
cat repo-name.git/HEAD
## Example Output: ref: refs/heads/main
```

Before pushing to the repository you need to match the local branch on the server:
```bash
## on the server
git -C /var/git/repo-name.git symbolic-ref HEAD refs/heads/main
```

### Set Up Post-Receive Hooks
Login to the server and link the `stagit-gen` script to the `post-receive` hook of the repository. This will regenerate the corresponding html page automatically on every push. 
```bash
ln -sf /usr/local/bin/stagit-gen /var/git/repo-name.git/hooks/post-receive
```
You will need to do this manually for each new repo. You can also [set up a repository template](#bonus-git-templates).

### Configure the remote origin of the local repository
Link your local repository to the remote. 
```bash
git remote set-url origin git@git.example.org:repo-name.git
```
*If you don't have a remote set yet, use `git remote add origin` instead.*


### Push to the Repository
```bash
git push -u origin main
```

### Generate the site
*This is only needed for the initial generation, or if you need to manually regenerate the site.*

Change to the git user with `su -l git` and run the script:
```bash
stagit-gen
```

Check the output:
```sh
ls /var/www/git
ls /var/www/git/repo-name
```

## Configure nginx
Probably also already installed, if not run:
```bash
apt install nginx
```

### Obtain a HTTPS certificate
To have our site available via HTTPS you need to obtain a certificate with `certbot`. Make sure to have `git.example.org` pointing to your servers IP.

Install certbot:
```bash
apt install python3-certbot-nginx
```

Request the certificate:
```bash
certbot certonly --standalone -d git.example.org
```

### Add the nginx configuration

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

    # Git HTTP clone 
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

## (Optional) Enable https cloning
To enable anyone to clone your repos with `git clone https://git.example.org/repo-name.git` you will need it install `fcgiwrap`:
```sh
apt install fcgiwrap
## enable auto-start on boot
systemctl enable --now fcgiwrap
```

### Setup the service to run as the git user
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

### Enable cloning per repo
You will need to allow the cloning for each repo. To do that, just touch the `git-daemon-export-ok` file in the repo root:
```sh
touch /var/git/repo-name.git/git-daemon-export-ok
```
To automate this, set up a [repository template](#bonus-git-templates).

## Deleting a repository on the server
This is quite straightforward, just do the following:
```bash
## Remove the bare repo
rm -rf /var/git/repo-name.git
## Remove the generated static pages
rm -rf /var/www/git/repo-name
## Regenerate the index to remove it from the listing
stagit-gen
```

## Add a logo and favicon to the site
By default **stagit** has the logo and the favicon saved under `/var/www/git/`.

I used a 32x32 png image for the logo.

Transfer the logo to the server:
```sh
scp logo.png git.example.org:
```

Copy it to the directory and fix permissions:
```sh
sudo mv logo.png /var/www/git/
## Re-use the logo as favicon
sudo cp /var/www/git/logo.png /var/www/git/favicon.png
sudo chown -R git:git /var/www/git/
```

## (Bonus) Git templates
Git has a built-in template workflow we can use to automate the `post-receive` hook, and the `git-daemon-export-ok`.

On the server create the template directory:
```bash
mkdir -p /var/git/template/hooks
```

Add the post-receive hook:
```bash
ln -sf /usr/local/bin/stagit-gen /var/git/template/hooks/post-receive
```

Add git-daemon-export-ok:
```bash
touch /var/git/template/git-daemon-export-ok
```

Then configure git globally to use the template:
```bash
git config --global init.templateDir /var/git/template
```
