+++
date = '2026-02-22T13:28:14+01:00'
draft = false
title = 'Using Signal Without a Smartphone'
+++

This is a quick guide on how to setup <code>signal-desktop</code> without using the official Signal app, but by using the unofficial CLI tool [signal-cli](https://github.com/AsamK/signal-cli).

## Installation
### Arch Linux
```bash
sudo pacman -S signal-desktop
yay -S signal-cli
```

### Windows
Install [Java Standard Edition](https://www.java.com/en/download/manual.jsp), as well as [Signal for Desktop](https://signal.org/download/). Also download the pre-compiled binary for [signal-cli](https://github.com/AsamK/signal-cli/releases).

Extract the <code>signal-cli</code> archive and cd into the <code>bin</code> folder.

For the following commands, replace <code>signal-cli</code> with <code>signal-cli.bat</code>.

## Start Registration
Make sure to write you phone number with its country code (like +32 for Belgium).

Open [https://signalcaptchas.org/registration/generate.html](https://signalcaptchas.org/registration/generate.html) and solve the captcha. Right-click on "Open Signal" and press "Copy Link".
```bash
signal-cli -u <phone-number> register --captcha "<copied-captcha>" 
```


## Verify
Verify the registration with the code you received via SMS. If you set up a Signal PIN before, enter that too:
```bash
signal-cli -u <phone-number> verify <code> --pin <signal-pin>
```

## Decode signal-desktop qr code
Take a cropped screenshot of the QR code shown in Signal Desktop and decode with <code>zbarimg</code>:
```bash
maim -s /tmp/qr.png && zbarimg /tmp/qr.png
```
Alternatively you could use one of the many [QR-Code Decoder websites](https://qrcode-decoder.com/).

## Approve the addition of the device
Add the device with the decoded QR-Code:
```bash
signal-cli -u <phone-number> addDevice --uri "<decoded-qr>"
```

## Set your name and profile picture
If you want you can also specify your Name and profile picture:
```bash
signal-cli -u <phone-number> updateProfile --name "<name>" --avatar <path/to/pic>
```

## Update info and chats
In my case my recent chats did not appear in the desktop app, until I queried the server for new messages with:
```bash
signal-cli -u <phone-number> receive
```
