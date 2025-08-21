---
title: "Encrypting files with SOPS, AGE, and Yubikey on atomic systems"
author: ""
type: ""
date: 2025-08-22T01:08:06+07:00
subtitle: ""
image: ""
tags: [
  "sops",
  "security",
  "yubikey",
  "atomic systems",
  "immutable systems"
]
---

If you’re here, you’re probably looking for a way to store secrets securely. This post isn’t about Kubernetes or managing cluster secrets — it’s about keeping stuff safe **locally** on your own machine. User-focused.

On a regular mutable system, setting up tools like SOPS and AGE is pretty straightforward. But on atomic/immutable systems (like Fedora Silverblue or openSUSE MicroOS), things get trickier. That’s what I’m tackling here.

---

## Foreword

For the past few years I’ve been using [*CryFS*](https://www.cryfs.org/) to keep my secrets safe: keys, tokens, backups of my password manager — basically anything private. It worked great, but it started feeling like time to move to something more modern and future-proof.

That’s where [*SOPS*](https://github.com/getsops/sops) comes in — it integrates multiple encryption solutions under one tool. [*AGE*](https://github.com/FiloSottile/age) is another piece: simple, modern, with a clean CLI and plugin support. And finally, [*Yubikey*](https://www.yubico.com/): a hardware security key that adds extra safety.

## My list of requirements

* Modern and future-proof
* Simple and secure
* Open source
* Easy enough to use on atomic systems
* Works across different systems
* Keeps me in full control of my secrets
* No dependency on a third-party service

Most of this is covered by SOPS + AGE + Yubikey. The tricky part is just making them play nicely on an atomic/immutable host.

## The design

AGE is perfect for creating key pairs to encrypt/decrypt files. For example:

```bash
age-keygen -o mykey.txt
```

That’s simple, but now I’ve just created yet another secret to keep safe. Luckily, I’ve got two Yubikeys, so I can build a redundant setup that avoids crypto-locking myself.

Here’s the plan:

* Use Yubikeys for everyday encryption/decryption
* Keep an AGE-generated software keypair as a fallback, stored offline (USB drive + password manager copy)

That way, even if both Yubikeys are lost, I’m not locked out forever.

## Preparing the system

I’m currently running [*Fedora Silverblue*](https://fedoraproject.org/atomic-desktops/silverblue/). I recently ditched [*distrobox*](https://github.com/89luca89/distrobox) and went back to devcontainers for development — more secure, more interoperable. So I need this setup to work on the **host** system.

Back when I was on openSUSE MicroOS, I got into the habit of installing CLI tools with Homebrew, and I’ve stuck with it. Luckily, everything I need is packaged there:

```bash
brew install sops age age-plugin-yubikey
```

That installs SOPS, AGE, and the plugin that connects AGE with the Yubikey.

But that alone isn’t enough. I don’t want to type my password every time I use the Yubikey, and brew installs don’t live in root’s `$PATH`. To fix this, I added a *polkit* rule that lets my user talk to the Yubikey directly:

```javascript
# nvim /etc/polkit-1/rules.d/10-pcsc-custom.rules

polkit.addRule(function(action, subject) {
    if (action.id == "org.debian.pcsc-lite.access_pcsc" &&
        subject.user == "myusername") {
            return polkit.Result.YES;
    }
});

polkit.addRule(function(action, subject) {
    if (action.id == "org.debian.pcsc-lite.access_card" &&
        action.lookup("reader") == 'Yubico YubiKey OTP+FIDO+CCID 00 00' &&
        subject.user == "myusername") {
            return polkit.Result.YES;
    }
});
```

Restart *polkit*:

```bash
sudo systemctl restart polkit
```

And enable the *pcscd* service so the Yubikey is ready at boot:

```bash
sudo systemctl enable --now pcscd.service pcscd.socket
```

One more fix: brew-installed packages can’t see the `pcscd` socket directly. Quick workaround — symlink it:

```bash
sudo ln -s /run/pcscd/pcscd.comm /home/linuxbrew/.linuxbrew/var/run/pcscd.comm
```

Now the AGE Yubikey plugin works. I can generate a Yubikey-stored keypair:

```bash
age-plugin-yubikey --generate
```

Then save the identity and recipient:

```bash
age-plugin-yubikey --identity >> sops_vault.id
age-plugin-yubikey --list >> yubikey_5NFC.pub
```

This will ask for the PIN and require a Yubikey touch to confirm. Do this twice for both keys. After that, I also generate a fallback software keypair:

```bash
age-keygen -o backup.txt
```

And append it to `sops_vault.id` so it’s included as a backup option.

### Quick summary

At this point I have:

* The whole toolchain installed on the host
* Brew packages talking to `pcscd`
* Two Yubikeys each with a keypair
* A fallback software keypair
* A *polkit* rule that lets my user access the Yubikey without root

## Configuring SOPS

Now with keys ready, time to configure SOPS. Create a `.sops.yaml` file that tells SOPS which keys to use:

```yaml
# .sops.yaml
creation_rules:
  - path_regex: .*/.*
    age: "age1yubikey1ae5dsekqqmamy29fkr8tccyw8rxl20zl0pgxj7kxzswusuaf0yxs8x56pl,age1yubikey1dvyezkkz763ukzh54g2t6xt9as9h04mcmyujxarwwavrlvvggc5q6hyq00,age1gav0dwax4eejeas4q70g502ps6nvckulhn33nh5k3vv7c3klquvswfnvcr"
  - path_regex: .*
    age: "age1yubikey1ae5dsekqqmamy29fkr8tccyw8rxl20zl0pgxj7kxzswusuaf0yxs8x56pl,age1yubikey1dvyezkkz763ukzh54g2t6xt9as9h04mcmyujxarwwavrlvvggc5q6hyq00,age1gav0dwax4eejeas4q70g502ps6nvckulhn33nh5k3vv7c3klquvswfnvcr"
```

This tells SOPS to use the AGE keys on the Yubikeys. Both regex rules here match everything — one for directories, one for root files.

## Encrypting and decrypting files

SOPS shines with structured formats like YAML/JSON, where you can encrypt individual fields. That’s nice, but in my case I just want **binary in/out**. That’s where the `--input-type` and `--output-type` flags are handy.

Encrypt a file:

```bash
sops --encrypt --input-type binary --output-type binary --in-place apis
```

Decrypt it:

```bash
SOPS_AGE_KEY_FILE=$PWD/sops_vault sops --decrypt --input-type binary --in-place apis
```

Here, `SOPS_AGE_KEY_FILE` points to the file with my Yubikey keys, so SOPS knows how to decrypt. The `--in-place` flag updates the file directly. Without it, SOPS just prints the decrypted content to stdout — useful if I just need to quickly grab my master password.

---

## Closing thoughts

With this setup, I’ve got Yubikey-backed encryption working directly on _Fedora Silverblue_. No root hacks needed after setup, and it should carry over to other atomic/immutable systems, other _Linux_ distributions, and likely compatible with _MacOS_, and _Windows_ too.

It’s modern, secure, and doesn’t lock me into a single device or third-party service. Exactly what I wanted.

---
