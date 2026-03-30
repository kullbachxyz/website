+++
date = '2026-03-27T20:08:06+01:00'
draft = false
title = 'Alpine Linux Install Guide'
+++

<!--
ToDo:
- Run through the dotfiles setup again and check everything works
- Re-write statusbar scripts and make them as simple as possible (maybe even no clicking needed).
- Explain the configuration files in more detail
- [maybe] Add a more detailed program list
- [DONE] Improve introduction
-->

Alpine Linux is a very lightweight Linux distribution. In comparison to other distributions like Debian or Arch Linux it uses simpler implementations of the **core utils** (commands like `ls`, `cp` or `cat`) and the **OpenRC init system** (service control, including boot and shutdown). Alpine also has a very efficient package manager called `apk`.

The following article is about how I like set up a new system.

## Download the ISO
Visit the [download page](https://alpinelinux.org/downloads/).
In the **Standard** section, choose the **x86_64** version.

## Prepare the USB
If you are on Windows you can use **Rufus** to flash the image to the USB drive. On Mac or Linux you can simply use `dd`:
```
dd if=alpine-standard-3.23.3-x86_64.iso of=/dev/sdX bs=4M status=progress; eject /dev/sdX
```

## Start base installer
After the installer has loaded, login with username `root` and no password.

Alpine comes with a convenient install script that automates the basic installation steps like partitioning, user creation and kernel install. Launch the base installer by typing:
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

To check the group memberships your user type:
```
getent group | grep <username>
```

If your user is not a member of the `wheel` group, you can add your user with this command:
```
adduser <username> wheel
```

Change back to your non-root user by typing `exit`.

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
Type `chsh <username>` and enter `/bin/bash/`:
```
Changing the login shell for <username>
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
[add doas here]
During the base install we set up the keymap, but only for tty sessions. To set the keymap in X11, add following to `/etc/X11/xorg.conf`:
```
Section "InputClass"
    Identifier  "Keyboard Default"
    MatchIsKeyboard "yes"
    Option      "XkbLayout" "de"
EndSection
```
<br>

***If you plan to [setup existing dotfiles](/setup-your-dotfiles-repo), it would be easiest to do this now.***


## Setup the login profile
The `.profile` file is used for customizing the user environment. It stores everything that should be run at tty login.
```
~/.profile
---
#!/bin/sh
# set ENV to a file invoked each time sh is started for interactive use.
export ENV="$HOME/.ashrc"

# Add scripts to $PATH
PATH="$PATH:$(find ~/.local/bin -type d)"

# Default programs
export BROWSER="librewolf"
export EDITOR="vim"
export TERMINAL="st"
export READER="zathura"

# Automatically start dwm in tty
[ "$(tty)" = "/dev/tty1" ] && ! pidof -s Xorg >/dev/null 2>&1 && exec startx
```

## Install basic programs and fonts
```
doas apk add xset util-linux dbus-x11 librewolf dunst xwallpaper font-hack 
```

To support Emojis, as well as Chinese, Japanese, and Korean characters, install:
```
doas apk add font-noto-emoji font-noto-cjk
```

## Suckless software install
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

To tell Xorg what to do when starting you need to create the file ~/.xinitrc and add the execution command for dwm:
```
 echo "exec dwm" >> ~/.xinitrc 
```
Now log out and back in, otherwise you will get an error when trying to start X.

Sine we added `[ "$(tty)" = "/dev/tty1" ] && ! pidof -s Xorg >/dev/null 2>&1 && exec startx` in `~/.profile`, dwm should start automatically. 

If you have not added that line, you can start dwm by entering `startx`.


## Setup audio
Install required packages:
```
doas apk add pipewire wireplumber pipewire-pulse pipewire-pulse pipewire-pulse pamixer pulsemixer
```
To make PipeWire work `XDG_RUNTIME_DIR` must be set in the running environment.
Create the file `~/xprofile` - it stores the environment variables for the Xsession.
```
~/.xprofile
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

<!-- figure out why `rc-update -U add pipewire default` says `XDG_RUNTIME_DIR unset.` -->

Add `. "$HOME/.xprofile"` to your `~/.xinitrc` to source it when xinit is executed.

## Setup Wifi
Alpine uses `wpa_supplicant` for wifi by default with the `networking` service. To make it easier to administer wifi configuration, install **NetworkManager**.
The packages needed are:
```
doas apk add networkmanager networkmanager-wifi networkmanager-tui
```
<!--Add doas here-->
Add the following to the /etc/NetworkManager/NetworkManager.conf file as follows:
```
/etc/NetworkManager/NetworkManager.conf
---
[main] 
dhcp=internal
plugins=ifupdown,keyfile

[ifupdown]
managed=true

[device]
wifi.scan-rand-mac-address=yes
wifi.backend=wpa_supplicant
```

 Now you need to stop conflicting services:
```
doas rc-service networking stop
doas rc-service wpa_supplicant stop 
```

Now restart NetworkManager:
```
doas rc-service networkmanager restart
```

Now connect to a network with `doas nmtui`. If that works, you can disable `networking` and `wpa_supplicant` and enable `networkmanager` on boot:
```
doas rc-update add networkmanager default
doas rc-update del networking boot

# Only needed if you connected via wifi during install
doas rc-update del wpa_supplicant boot
```
<!--figure out why`adduser ph plugdev` did not work (nmtui still says not permitted) -->

To add a new wifi connection, you can type `doas nmtui` in the terminal.

## Setup XDG user dirs

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
Initialize the bare repo and set up the alias as before.

Add the remote:
```
conf remote add origin git@github.com:<username>/dotfiles.git
```

Download the remote:
```
conf fetch origin
```

Check the remote branches:
```
conf branch -r
```

Switch to the correct remote branch:
```
conf switch -c main origin/main
# or
conf switch -c master origin/master
```

You might get an error like this:
```
error: The following untracked working tree files would be overwritten by checkout:
        .xinitrc
        .xprofile
Please move or remove them before you switch branches.
Aborting
```

Move the files to a temporary backup location and re-run the `conf switch` command like before.

After adding a commit push the changes:
```
conf push --set-upstream origin main
```

## Configuration files
.profile - Login shell startup file (tty)

.xinitrc - Executed on startx

.xprofile - Environment variables for the x session

.ashrc - Per-interactive-shell startup file



