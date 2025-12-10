---
title: "Tools"
layout: "single-notitle"
---

*Curated list of privacy & security-focused software and services I recommend*

## Browsers
~~[Mozilla Firefox](https://www.firefox.com/de/) with [arkenfox.js](https://github.com/arkenfox/user.js) applied.~~ Switched to [LibreWolf](https://librewolf.net/) with my own [hardening script](https://github.com/kullbachxyz/librewolf-hardening) applied to it. If you need to use Chrome, use [ungoogled-chromium](https://ungoogled-software.github.io/ungoogled-chromium-binaries/). Take a look at the recommended [Browser Extensions](#browser-extensions) also!

## Search Engines
**_Don't use Google!_** ~~My standard recommendation would be [DuckDuckGo](https://duckduckgo.com/?t=ffa)~~. I have been using a self-hosted instance of [SearNGX](https://github.com/searxng/searxng) for some time now and I am quite happy with it. [Brave Search](https://search.brave.com/) and [Mojeek](https://www.mojeek.com/) maintain thier own indexes, seperate from Google and Bing. You can find a list of public SearNXG instances [here](https://searx.space/).

## Password Managers
[KeepassXC](https://keepassxc.org/) on my laptop [KeePassDX](https://www.keepassdx.com/) on my phone.

*I have used 1Password and Bitwarden before, but I dont need my vault to be synced all the time. If I need the current version of my vault on my phone, I use [LocalSend](https://localsend.org/de) to transfer it to my phone.*

<!--
## 2-Factor Authentication
I use the build in 2FA function in KeePass.
-->

## Messaging
[Matrix](https://matrix.org/) all the way! I host my own server for family and friends (if I can convince them to switch). [Signal](https://signal.org/) for everyone else. Also, **don't use WhatsApp!**

## Email
***Email is generally not secure*** - use PGP whenever possible. Avoid using free email providers (e.g. Gmail, Outlook, GMX).  Providers to use instead: [Posteo](https://posteo.de/en) and [Mailbox.org](https://mailbox.org/en/) both cost only 1€/month for basic functionality. If you don't want to spent any money, at least choose the free plan of [ProtonMail](https://proton.me/mail) or [Tuta](https://tuta.com/de).

## Email Clients
I use [Mozilla Thunderbird](https://www.thunderbird.net/en-US/).

## Browser Extensions
[Privacy Badger](https://privacybadger.org/) identifies trackers by monitoring when the same third-party domains show up on different websites and use techniques like cookies or fingerprinting. [uBlock Origin](https://ublockorigin.com/) relies on filter lists for blocking. [Decentraleys](https://decentraleyes.org/) provides local copies of common scripts instead of calling to 3rd-party network. [I still don't care about cookies](https://github.com/OhMyGuus/I-Still-Dont-Care-About-Cookies) removes annoying cookie warnings from websites. [Vimium](https://vimium.github.io/) is nice if you want to add vim-style shortcuts to your browser.
One of my favorite extensions is [LibRedirect](https://libredirect.github.io/). It redirects YouTube, Instagram, Reddit, TikTok and other websites to alternative privacy-friendly frontends.

## Mobile Apps
**_I hate smartphones_**. I own a Pixel 6a with [GrapheneOS](https://grapheneos.org/) installed on it, but it has no mobile plan and never leaves the house.  
Still here are a few useful apps: [Orbot](https://support.torproject.org/glossary/orbot/) is a system-wide proxy that encrypts all your traffic and hides it from your ISP. It uses the TOR Network. [Molly](https://molly.im/) (Android-only) is a fork of Signal that encrypts your message data at rest, prompting you to enter a passphrase when you unlock the app. It also securly shreds memory content after closing. [KeepassXC](https://keepassxc.org/) is a great KeePass client for Android. Use [KeePassium](https://keepassium.com/pricing/) if you are on iOS. [Organic Maps](https://organicmaps.app/) is a very good offline maps app for Anroid and iOS. ~~[FitoTrack](https://codeberg.org/jannis/FitoTrack) is a app for tracking your workouts.~~ I track my workouts with a [Garmin fenix 5](https://www.garmin.com/en-US/p/552982/) and visualize them with my app [garmr](https://github.com/kullbachxyz/garmr).

If you are on Android **don't use the Play Store**, use [F-Droid](https://f-droid.org/) or [Obtainium](https://obtainium.imranr.dev/) to download your apps. If you are dependant on Apps that is only avaliable on the Play Store, use the [Aurora Store](https://f-droid.org/packages/com.aurora.store/) to obtain them.
