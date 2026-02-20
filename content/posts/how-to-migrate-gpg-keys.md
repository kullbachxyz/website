+++
date = '2026-01-12T22:23:55+01:00'
draft = true
title = 'Moving GPG keys to a new system'
+++

I recently switched laptops and wanted to cleanly migrate my GPG keys from my old machine. This is a quick guide on how to do just that (and be a way for me to look this up in case I forget this...).

## Check your existing keys
```bash
gpg --list-secret-keys --keyid-format LONG
```

You will get something like:
```bash
sec   ed25519/D4FG68BA0440566E 2025-07-09 [SC]
      A7791BFAA22E3DE78F9G8D4BD4FD68BAG940466E
uid                 [ultimate] Tyler Test <tyler@test.xyz>
ssb   cv25519/312GCC7C39676A60 2025-07-09 [E]
```

## Exporting your key and transferring it to the new machine

Export your **public key**:
```bash
gpg --export -a <KEY_ID> > gpg.pub
```

...and your **private key**:
```bash
gpg --export-secrect-keys -a <KEY_ID> > gpg.sec
```

Transfer to the new machine:
```bash
scp gpg.pub gpg.sec <IP>:
```

Make sure to ***shred*** these files as soon as you moved them to the new machine!

## Import your key on the new machine
```bash
gpg --import gpg.pub
gpg --import gpg.sec
```
