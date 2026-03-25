---
title: "Programs and Equipment I Use"
layout: "single-notitle"
---

## Software
*Configuration for most things listed here is on my [GitHub](https://github.com/kullbachxyz/conf).*

### Operating System
***Arch Linux***. I used Windows and macOS before - switched to macOS in 2023, then moved to Linux in mid-2025. I like Arch for the control it gives you over every package on the system.

### Window Manager
A heavily modified [build of dwm](https://github.com/kullbachxyz/dwm), with [dmenu](https://github.com/kullbachxyz/dmenu) as my launcher and [dwmblocks](https://github.com/kullbachxyz/dwmblocks) as my status bar. I've tried [MangoWC](https://github.com/DreamMaoMao/mangowc) on Wayland as well, but I need X11 for some work software.

### Terminal
[st](https://st.suckless.org/) - minimal codebase, single binary, customization through the source code.

### Text Editor
[(neo)vim](https://neovim.io/). The keyboard-based workflow is far more efficient than any GUI editor I've used. Still learning. I also use [LibreOffice](https://www.libreoffice.org/) for documents that need proper formatting.

### Web Browser
[LibreWolf](https://librewolf.net/) - a hardened Firefox fork without tracking and telemetry. I use it with the following extensions:
- [uBlock Origin](https://ublockorigin.com/) — ad and tracker blocker
- [Privacy Badger](https://privacybadger.org/) — tracker detection
- [Decentraleyes](https://decentraleyes.org/) — local CDN emulation
- [LibRedirect](https://libredirect.github.io/) — redirects to privacy-friendly frontends
- [I still don't care about cookies](https://github.com/OhMyGuus/I-Still-Dont-Care-About-Cookies) — cookie banner removal
- [Vimium](https://vimium.github.io/) — vim-style keyboard navigation

For sites that don't work in Firefox, I fall back to [ungoogled-chromium](https://github.com/ungoogled-software/ungoogled-chromium).

### Mail Client
[aerc](https://aerc-mail.org/), synced offline via [mbsync](https://github.com/gburd/isync) with [goimapnotify](https://gitlab.com/shackra/goimapnotify) for notifications.

### Document / Image Viewer
[Zathura](https://pwmt.org/projects/zathura/) with *zathura-pdf-mupdf* for PDFs. [nsxiv](https://codeberg.org/nsxiv/nsxiv) for images.

### Audio / Video
I mostly listen to music on my iPod, but at my computer I use [mpd](https://www.musicpd.org/) with [ncmpcpp](https://github.com/ncmpcpp/ncmpcpp). [beets](https://beets.io/) for tagging, [mpv](https://mpv.io/) for video.

## Hardware

### Laptop
My main laptop is a corebooted **ThinkPad T480** (with Tianocore EDK2). Coreboot replaces the proprietary BIOS/UEFI firmware with open-source firmware, giving me full control over what runs on my device. I also run a **Thinkpad x220** (with SeaBIOS) as my secondary device.

### Phone
I don't use a smartphone - I use a basic phone, that only does texts and calls. Not for security reasons (a modern smartphone would actually be more secure in most cases), but because I'd rather not have a device that forces me into depending on whatever app some company decided is now mandatory.

### Homeserver
I use a **UGREEN NASync DXP2800** as my file server, as well as to run some apps in Docker containers. I run Debian on the NAS instead of the proprietary UGOS.
