---
layout: post
title:  "Telling your ThinkPad to shut up (no more 2x five beeps!)"
date:   2019-10-07 02:55:00 +0200
---
If you followed my previous post to remove the whitelist in your ThinkPad T440p's firmware, you probably have noticed a major annoyance:
Sometimes after turning on, your Laptop will let out two groups of five beeps, and then boot up normally. The reason for this is that your UEFI firmware is cryptographically signed, and now that you modified it, the signature is invalid. The beeping signals to you that the signature check failed.

Fixing this manually is not a trivial task. It involves recalculating SHA-hashes of UEFI volumes contained in your image, putting those hashes in the correct places in the image, calculating new SHA-hashes of the blocks where you put the additional hashes, applying the RSA algorithm to this hash (with a private key you have to generate), and finally putting the corresponding RSA public key in the correct place in the image.

If you think this sounds tedious - well, then you're right.

That's why I didn't bother doing it manually, and instead wrote me some Python tools for that.

The first tool is a script called `verify.py`. What it does is similar to what your ThinkPad does on startup - it extracts the RSA public key (which is required to check RSA signatures), extracts and checks all hashes, and then checks if the hashes are properly signed. If you just want to check whether your firmware image is properly signed - this is the tool to use.

The second tool is a script called `sign.py`. It's meant to sign a firmware image that wasn't properly signed before. To do that, it recalculates and updates the SHA hashes in the image, generates a new 1024-bit RSA key-pair, calculates signatures for the blocks where the hashes are stored, stores those signatures, and then, finally, stores the public key of the generated key-pair in the file. If you have a firmware image with an invalid signature, and want to fix it, this is the tool to use.

Both tools can be found here: https://github.com/thrimbor/thinkpad-uefi-sign

## TL;DR: ##
Alright, so this is how to use it. First of all, these are Python 3 scripts, and they use the "pycryptodome" library for hashing, key generation and signing. If you're on Linux, I trust you to be able to install your operating systems's package management to install these requirements. For macOS, you'll probably have to use something like `homebrew` to install Python, and `pip` to install pycryptodome. If you're on Windows, installing Python is kinda annoying, so you can download pre-packaged binaries built from my scripts (here)[https://github.com/thrimbor/thinkpad-uefi-sign/releases]. Due to those being packaged, they're .exe-files instead of .py, so remember to replace the file ending in the following steps if you're on Windows.

Now, we assume you have a modified firmware image, like the one you'll get if you follow my previous post. Let's try to check it's signature:
`./verify.py uefi_patched.bin`
And we get:
```
INFO: Found public key modulus at offset  0x3b4576
INFO: TCPA block found at offset  0x2c3a00
INFO: Volume offset: 0
INFO: Volume size: 2818048
ERROR: TCPA volume hash mismatch
  TCPA volume hash: b15c2d0ec469188f3037d40d51df36cdf9758e88
Actual volume hash: faa2e5984938d11c2cf75898121c1194a6544fd1

SIGNATURES INCORRECT!
```
As expected, signature verification failed. We modified the file containing the whitelist, so the old hash is incorrect now.
Let's try fixing it:
`./sign.py uefi_patched.bin -o uefi_patched_signed.bin`
The output should be like the following:
```
INFO: Found public key modulus at offset  0x3b4576
INFO: TCPA block found at offset  0x2c3a00
INFO: Generating new 1024 bit key with 3 as public exponent...
INFO: Volume offset: 0
INFO: Volume size: 2818048
INFO: Volume hash updated
INFO: Signature calculated
INFO: TCPA volume block signed
INFO: Public key stored

IMAGE SIGNED!
```
Ok, this looks promising! Let's run our verify-script again, this time for our new file:
`./verify uefi_patched_signed.bin`
And we get:
```
INFO: Found public key modulus at offset  0x3b4576
INFO: TCPA block found at offset  0x2c3a00
INFO: Volume offset: 0
INFO: Volume size: 2818048
INFO: TCPA volume hash verified
INFO: Volume signature verified

SIGNATURES CORRECT!
```

Fantastic! This means that, if we flash the new, signed image to our Laptop, there should be no more beeps!
