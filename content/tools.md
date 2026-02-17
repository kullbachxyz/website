---
title: "Tools"
layout: "single-notitle"
---

*Curated list of privacy & security-focused software and services I recommend*

## Browsers
~~[Mozilla Firefox](https://www.firefox.com/de/) with [arkenfox.js](https://github.com/arkenfox/user.js) applied.~~ Switched to [LibreWolf](https://librewolf.net/) with my own [hardening script](https://github.com/kullbachxyz/librewolf-hardening) applied to it. If you need to use Chrome, use [ungoogled-chromium](https://ungoogled-software.github.io/ungoogled-chromium-binaries/). Take a look at the recommended [Browser Extensions](#browser-extensions) also!

## Search Engines
I use a self-hosted instance of [SearXNG](https://github.com/searxng/searxng) and am quite happy with it. [Brave Search](https://search.brave.com/) and [Mojeek](https://www.mojeek.com/) maintain their own indexes, separate from Google and Bing. You can find a list of public SearXNG instances [here](https://searx.space/).

## Password Managers
~~[KeepassXC](https://keepassxc.org/) on my laptop, [KeePassDX](https://www.keepassdx.com/) on my phone.~~ I use [pass](https://www.passwordstore.org/) now. Passwords are just GPG-encrypted files on disk.

## Messaging
I prefer [Matrix](https://matrix.org/) and host my own server for family and friends. [Signal](https://signal.org/) for everyone else.

## Email
~~I use [Mozilla Thunderbird](https://www.thunderbird.net/en-US/).~~ I self-host my own mail server and read mail locally with [aerc](https://aerc-mail.org/), synced via [mbsync](https://isstrux.sourceforge.net/mbsync.html) to a local Maildir. I use [goimapnotify](https://gitlab.com/shackra/goimapnotify) for desktop notifications. My mail sync scripts are in my [dotfiles](https://github.com/kullbachxyz/conf).

If self-hosting isn't for you, [Posteo](https://posteo.de/en) and [Mailbox.org](https://mailbox.org/en/) are good options at 1€/month. For free alternatives, [ProtonMail](https://proton.me/mail) and [Tuta](https://tuta.com/de) work well.

## Browser Extensions
- [uBlock Origin](https://ublockorigin.com/) — ad and tracker blocker
- [Privacy Badger](https://privacybadger.org/) — tracker detection
- [Decentraleyes](https://decentraleyes.org/) — local CDN emulation
- [LibRedirect](https://libredirect.github.io/) — redirects to privacy-friendly frontends
- [I still don't care about cookies](https://github.com/OhMyGuus/I-Still-Dont-Care-About-Cookies) — cookie banner removal
- [Vimium](https://vimium.github.io/) — vim-style keyboard navigation

