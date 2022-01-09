---
layout: post
title:  "Decrypting the Arch Linux boot partition with a FIDO2 security key"
date:   2022-01-09 17:00:00 +0100
---
If you own a laptop, it's certainly not the worst idea to have its hard drive encrypted, so your data won't be in other people's hands, even if your device gets stolen.
Using a normal passphrase for that isn't perfect though: It's usually either too short to be secure, or too long to not be a hassle when rebooting your machine.
Using a FIDO2 Security key *can* be a better option, and I'll show you how to set it up.

## Warnings ##
The following sections will instruct you to modify various sensitive files. Errors on your or my part may render your system unbootable, so make sure you have plenty of time to fix eventual problems, and keep a copy of your old GRUB configuration and initramfs around until you verify that everything works as intended.

## Prerequisites ##
To keep this post simple, I'll only describe one way to set this up. I'll focus on the hmac-secret FIDO2 extension, LUKS-encrypted partitions and Arch Linux (with systemd >=248). What I'm describing may be applicable to other setups, but this is what I use, and what worked for me.
Most importantly, you'll need a FIDO2 security key that supports the hmac-extension. Manufacturers often don't clearly advertise this feature, but many modern security keys such as the YubiKey 5 support it. I personally own and use a "Token2 T2F2-NFC FIDO2, U2F and TOTP Security Key".

## Registering your security key ##
This is the trivial part. Plug in your security key (make sure it's the only one plugged in), then run the following command:

`sudo systemd-cryptenroll --fido2-device=auto /dev/sda3`

Make sure to replace `/dev/sda3` with the path to your LUKS-encrypted partition, and be prepared to enter your passphrase.
When you're done, one of the key slots in the LUKS header of this partition will be used for your FIDO2 security key.

## Unlocking during boot ##
This is the more complicated part, and it took me a while to get this going. Most other instructions tell you to simply modify a line in `/etc/crypttab`, but it appears that this doesn't work for the root filesystem - after all, this file lies inside the partition we're trying to decrypt.
If you're (like me) booting your system with GRUB 2, you might be able to make the necessary adjustments on the kernel command line in `/etc/default/grub`, but I couldn't make this work.
So what we're doing instead is letting systemd take care of the partition decryption. First of all, create the file `/etc/crypttab.initramfs`. When we're done, this file will end up as `/etc/crypttab` in your initramfs.
Add the following line to it:

`luks /dev/sda3 - fido2-device=auto`

Replace `luks` with your mapper name (if you can't remember, run `mount | grep "on / "` to find it), and `/dev/sda3` with the path to your LUKS-partition. If you're using an SSD, add `,discard` at the end.

Next, we need to make sure there's nothing interfering with our file. Edit `/etc/default/grub`, and find the line starting with `GRUB_CMDLINE_LINUX`. It should contain something like `cryptdevice=/dev/sda3:luks:allow-discards`, delete this part, save the file, and update your grub.cfg (for me, that's `sudo grub-mkconfig -o /boot/grub/grub.cfg`).

Now, we need to configure and create our initramfs. Open `/etc/mkinitcpio.conf`, find the line starting with `HOOKS=`, and add the following hooks: `systemd`, `sd-vconsole`, and `sd-encrypt`, and remove `encrypt`. For me, the line looks like this now:

`HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block sd-encrypt filesystems fsck)`.

Save the file, and run `sudo mkinitcpio -p linux`.

## Trying it out ##
When you're done, you should be ready to try it out. Make sure your FIDO2 security key is plugged in, and reboot your system. When your key's LED blinks, press the button (or however this works on your key), and your system should continue booting, without asking for a passphrase.

When no FIDO2 key is found, systemd should ask you for your passphrase. I noticed that this takes quite a few seconds, but I intend to mostly use my key from now on, so this is just a minor inconvenience for me.

## Further reading ##
Here's a small collection links that dive further into the topics covered above:

[https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition)

[http://0pointer.net/blog/unlocking-luks2-volumes-with-tpm2-fido2-pkcs11-security-hardware-on-systemd-248.html](http://0pointer.net/blog/unlocking-luks2-volumes-with-tpm2-fido2-pkcs11-security-hardware-on-systemd-248.html)
