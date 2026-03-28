+++
date = '2026-03-27T20:08:06+01:00'
draft = false
title = 'Alpine Linux Install Guide'
+++
## Download the iso
Head over to https://alpinelinux.org/downloads/
In the **Standard** section, choose the **x86_64** version.

## Write image to usb
```
dd if=alpine-standard-*-x86_64.iso of=/dev/sdb bs=8M status=progress
```
## Boot from usb
Login with username root (no password) and launch the base installer by typing:
```
setup-alpine
```
Follow the instructions on screen.

## Install doas
Change to the root user:
```
su -
```
Install doas and vim:
```
apk add doas vim
```
Edit the doas configuration:
```
vim /etc/doas.conf
---
# Uncomment to allow group "wheel" to become root.
permit persist :wheel
```

Add your user to the wheel group:
```
adduser ph wheel
```

To check the group memberships of a user type:
```
getent group | grep username
```


## Enable community repositories
Open the repositories configuration file:
```
doas vim /etc/apk/repositories
```
Uncomment the *community* line.

Update the package-lists:
```
doas apk update
```

## (Optional) Change the default shell
Alpine comes with `/bin/sh` as its shell by default. To change it to `/bin/bash` you need to install bash itself, as well as the `shadow` package to be able to use `chsh`:
```
doas apk add shadow bash
```
Type `chsh ph` and enter `/bin/bash/`:
```
Changing the login shell for ph
Enter the new value, or press ENTER for the default
        Login Shell [/bin/sh]: /bin/bash
```

To use the new shell you need to log out and back in.


## Setup xorg
Setup Xorg by typing:
```
doas setup-xorg-base
```

## Fix keymap
During the base install we set up the keymap, but only for tty sessions. To set the keymap in X11 first install `setxkbmap`:
```
doas apk add setxkbmap
```
Add following to `/etc/X11/xorg.conf`:
```
Section "InputClass"
    Identifier  "Keyboard Default"
    MatchIsKeyboard "yes"
    Option      "XkbLayout" "de"
EndSection
```

## Install basic programs and fonts
```
doas apk add xset util-linux dbus-x11 librewolf dunst xwallpaper font-hack 
```

To support Emojis, as well as Chinese, Japanese, and Korean characters, install:
```
doas apk add font-noto-emoji font-noto-cjk
```

## suckless software install
Install build dependencies:
```
doas apk add git make gcc g++ libx11-dev libxft-dev libxinerama-dev ncurses 
```

Create a source build folder:
```
mkdir -p ~/.local/src
```

Clone source for dwm, st and dmenu:
```
git clone https://codeberg.org/kullbachxyz/dwm.git
git clone https://codeberg.org/kullbachxyz/st.git
git clone https://codeberg.org/kullbachxyz/dmenu.git
```

Go into each folder and run `doas make clean install`.

To tell Xorg what to do when starting you need to create the file ~/.xinitrc:
```
xset r rate 200 35 &

exec dwm;
```
Now log out and back in, otherwise you will get an error when trying to start X.

Start dwm by entering `startx`.



## Setup audio
Install required packages:
```
doas apk add pipewire wireplumber pipewire-pulse pipewire-pulse pipewire-pulse pamixer pulsemixer
```
To make PipeWire work `XDG_RUNTIME_DIR` must be set in the running environment.
Create the file `~/xprofile` - it stores the environment variables for the Xsession.
```
vim ~/.xprofile
---
#!/bin/sh

if test -z "${XDG_RUNTIME_DIR}"; then
  export XDG_RUNTIME_DIR=/tmp/xdg/"${UID}"-xdg-runtime-dir
    if ! test -d "${XDG_RUNTIME_DIR}"; then
      mkdir -p "${XDG_RUNTIME_DIR}"
      chmod 0700 "${XDG_RUNTIME_DIR}"
    fi
fi 

# Start pipewire
export "$(dbus-launch)"
/usr/libexec/pipewire-launcher &
```

Add `. "$HOME/.xprofile"` to your `.xinitrc` to source it when xinit is executed.


# Setup XDG user dirs

Install `xdg-user-dirs´:
```
doas apk add xdg-user-dirs
```

Create custom directories:
```
mkdir ~/docs
mkdir ~/dl
mkdir ~/music
mkdir ~/pics
mkdir ~/vids
```

Create the configuration file:
```
~/.config/user-dirs.dirs
---
XDG_DOCUMENTS_DIR="$HOME/docs"
XDG_DOWNLOAD_DIR="$HOME/dl"
XDG_MUSIC_DIR="$HOME/music"
XDG_PICTURES_DIR="$HOME/pics"
XDG_VIDEOS_DIR="$HOME/vids"
XDG_DESKTOP_DIR="$HOME/"
XDG_TEMPLATES_DIR="$HOME/"
XDG_PUBLICSHARE_DIR="$HOME/"
```

Update the user directories:
```
xdg-user-dirs-update
```

## Wallpaper
Since we have xwallpaper installed we can use it to set a wallpaper within `.xprofile`:
```
xwallpaper --zoom ~/pics/walls/wall1.jpg &
```

## Setup your dotfiles repo
Set your name and email address for git:
```
git config --global user.name "John Doe"
git config --global user.email johndoe@example.com
```

To create a new ssh key run:
```
ssh-keygen -t ed25519 -C "your_email@example.com"
```

### Setup a new dotfiles repository
Initialize the bare repo:
```
git init --bare $HOME/.dotfiles
```

Add an alias for the git command targeting the dotfiles repo to your shell startup file:
```
~/.ashrc
---

alias conf='/usr/bin/git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME'
```

Git will show all files in `~/` as untracked files by default. To disable this type:
```
conf config --local status.showUntrackedFiles no
```

### Setup an existing dotfiles repository
Initialize the bare repo and set up the alias as before. Make sure your local `HEAD` points to the corrent branch.


Checkout the existing files in your home directory:
```
config checkout
```

You might get an error like this:
```
error: The following untracked working tree files would be overwritten by checkout:
        .xinitrc
        .xprofile
Please move or remove them before you switch branches.
Aborting
```

Remove or move the files somewhere else for now. After that run `git checkout` agian to pull from the remote.

Switch to SSH auth if not already done:
```
git remote add "origin" git@github.com:<username>/kullbachxyz.git
```

After adding a commit push the changes:
```
git push --set-upstream origin main
```


## Librewolf Configuration



## Configuration files
.xinitrc - Executed on startx

.xprofile - Environment variables for the x session

.bashrc - Per-interactive-shell startup file

.bash_profile - Login shell startup file


