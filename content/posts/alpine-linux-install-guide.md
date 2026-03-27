+++
date = '2026-03-27T20:08:06+01:00'
draft = true
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
Install doas:
```
apk add doas
```
Edit the doas configuration:
```
vim /etc/doas.conf
```
```
# Uncomment to allow group "wheel" to become root.
permit persist :wheel
```

Add your user to the wheel group:
```
adduser ph wheel
```

## Change the default shell
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

Create the file `~/.bashrc` - the file is loaded every time you open a terminal.
To get a fany looking prompt add the following:
```
#!/usr/bin/env bash

PS1='\[\e[91;1m\][\[\e[93m\]\h\[\e[92m\]@\[\e[94m\]\u\[\e[0m\] \[\e[95;1m\]\w\[\e[91m\]]\[\e[0m\]\\$ '
```

## Enable community repositories
Install vim:
```
doas apk add vim
```

Open the repositories configuration file:
```
doas vim /etc/apk/repositories
```
Uncomment the *community* line.

Update the package-lists:
```
doas apk update
```

## Setup xorg
Setup Xorg by typing:
```
setup-xorg-base
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
doas apk add xset dbus-x11 librewolf dunst picom xwallpaper font-hack font-noto-emoji font-noto-cjk
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

After that's done you can start dwm by entering `startx`



## Setup audio
Install required packages:
```
doas apk add pipewire wireplumber pipewire-pulse pipewire-pulse pipewire-pulse pamixer pulsemixer
```
To make PipeWire work `XDG_RUNTIME_DIR` must be set in the running environment.
Create the file `~/xprofile` - it stores the environment variables for the Xsession.
```
vim ~/.xprofile
```
```
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


# NEXT STEPS 

## .xprofile ausostart progrqams


## Librewolf Configuration




## Setup your dotfiles repo
Set your name and email address for git:
```
git config --global user.name "John Doe"
git config --global user.email johndoe@example.com
```

```
git init --bare $HOME/.dotfiles
alias conf='/usr/bin/git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME'
conf config --local status.showUntrackedFiles no
echo "alias conf='/usr/bin/git --git-dir=$HOME/.cfg/ --work-tree=$HOME'" >> $HOME/.bashrc
```

## Configuration files
.xinitrc - Executed on startx
.xprofile - Environment variables for the x session
.bashrc - Per-interactive-shell startup file
.bash_profile - Login shell startup file

# Setup XDG user dirs
```bash
mkdir -p "${HOME}/docs" "${HOME}/music" "${HOME}/pics" "${HOME}/vids" "${HOME}/dl"
```

