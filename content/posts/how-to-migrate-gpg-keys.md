+++
date = '2026-01-12T22:23:55+01:00'
draft = false
title = 'Moving GPG keys to a new system'
+++

Here's a quick guide on how to migrate your GPG keys when switching to a new machine. 

## Check your existing keys
```bash
gpg --list-secret-keys --keyid-format LONG
```

You will get something like:
```bash
sec   ed25519/A3CF82DE09B7541A 2025-07-09 [SC]
      E92F1BAA7D6C3E058F4A8D4BA3CF82DE09B7541A
uid                 [ultimate] Tyler Test <tyler@test.xyz>
ssb   cv25519/7B2ECC4F61D83A95 2025-07-09 [E]
```
<code>SC</code> means the key can Sign (data/messages) and Certify (confirm another key is legit).<br>
<code>E</code> means the subkey is used for Encryption only.

## Exporting your key and transferring it to the new machine

Export your **public key**:
```bash
gpg --export -a <KEY_ID> > gpg.pub
```

...and your **private key**:
```bash
gpg --export-secret-keys -a <KEY_ID> > gpg.sec
```
This exports the private key in ***plain text***!

Transfer to the new machine:
```bash
scp gpg.pub gpg.sec <IP>:
```
This will transfer the files to the remote machines home directory. Alternatively you could use an USB stick to transfer the files.

Make sure to ***shred*** these files as soon as you move them to the new machine!
```bash
shred -u gpg.pub gpg.sec
```

## Import your key on the new machine
```bash
gpg --import gpg.pub
gpg --import gpg.sec
```
The keys are now imported, but are not trusted. You can either export the owner trust from the old machine, or trust them manually using the gpg prompt.

## Trusting the keys
On the old machine:
```bash
gpg --export-ownertrust > trust.txt
```

Transfer to the new machine and run:
```bash
gpg --import-ownertrust trust.txt
```

If you have lost access to the old machine or want to setup trust manually, you can type <code>gpg --edit-key key-ID</code>, which will open a gpg prompt <code>gpg></code> where you just type <code>trust</code>. You will be now asked what level of trust you want:
```bash
Please decide how far you trust this user to correctly verify other users' keys
(by looking at passports, checking fingerprints from different sources, etc.)

  1 = I don't know or won't say
  2 = I do NOT trust
  3 = I trust marginally
  4 = I trust fully
  5 = I trust ultimately
  m = back to the main menu
```
Select <code>5</code> for ultimate trust. Ultimate trust is needed because it is **your own** key, not just any key. You can exit the prompt by entering <code>save</code>. That's it!
