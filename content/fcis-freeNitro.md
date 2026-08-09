---
title: Ain Shams CTF - FreeNitro
draft: false
date: 2026-08-09
---
This challenge was a part of "Cyber CUP Ain Shams CTF" that was held for 24 hours on **08/08/2026**.
# Challenge Description

![[freeNitro.png]]

Category: `malware reverse engineering`
# Solution
For this challenge we were given a single `exe` file:
![[Pasted image 20260809221316.png]]

The challenge description already pointed us to the right direction by stating that this `exe` contains compiled python bytecode. To actually extract this bytecode we need a dedicated tool.

`pyinstxtractor` should do the trick (I just looked it up): https://github.com/extremecoders-re/pyinstxtractor.git

![[Pasted image 20260809221857.png]]

And these are the results:

![[Pasted image 20260809234100.png]]

The main files that really concern us here are `.pyc` files which are python bytecode files.
So I just moved them into their own folder.

![[Pasted image 20260809235346.png]]

Now, what we should be looking here for is a discord webhook identifier. A discord webhook is just a link that allows the user to send messages through it. It usually has this form:

```
https://discord.com/api/webhooks/{webhook.id}/{webhook.token}
```

One file really stood out: `DISCORD BIRTHDAY NITRO CLAIMER.pyc` because it was all caps so I decided to investigate it first by running `strings` on it.

Doing so revealed a suspicious base64-encoded string:

![[Pasted image 20260809235818.png]]

Decoding it gave me this:

![[Pasted image 20260809235852.png]]

BOOM! There's our webhook identifier! I finally put it in the flag and submitted it.

`flag{651401571514712074}`