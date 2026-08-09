---
title: Ain Shams CTF - just_add
draft: false
date: 2026-08-09
---
This challenge was a part of "Cyber CUP Ain Shams CTF" that was held for 24 hours on **08/08/2026**.
# Challenge Description

![[just_add1.png]]

Category: `machine`
# Solution
Once I started the challenge, it dropped me in a shell:
![[just_add2.png]]

Going off the description of the challenge, this most likely has something to do with either the Justice League or `sudo`. I'm pretty sure it's the latter.

So I checked what my current user `ttyduser` is allowed to do using `sudo`:
![[just_add3.png]]

What have we got here, we can run the script `/opt/write.py` using `sudo` and we don't have to provide a password at all! (as shown through `NOPASSWD`)

Unfortunately I forgot to take a screenshot of the contents of the script, but it allowed me to choose a file and add any line to it as you'll see.

With this super power, I wanted more. Writing to any file? My head went straight to the `/etc/sudoers` file. That file contains a list of rules about different permissions of different users including what users are allowed to do using `sudo`.

It would be extremely powerful if I could somehow write to that file and give myself permissions to do anything using `sudo`. Which is exactly what I did.

But I first needed to know what to even write, because that file follows a specific syntax. A quick google search gave me this:

![[just_add4.png]]

https://askubuntu.com/questions/334318/sudoers-file-enable-nopasswd-for-user-all-commands

TLDR; All I needed to do was write `ttyduser ALL=(ALL) NOPASSWD: ALL` to `/etc/sudoers` to be able to run any command I want.

![[just_add7.png]]

Then to confirm that I then had full privileges to do anything:

![[just_add8.png]]

After that, I simply switched to `root`:

![[just_add9.png]]

Alright dear reader, I'll admit that it took me a minute to figure out where the flag might even be.

I thought that it might perhaps be in the home directory that we start in. Nope. Maybe in `/`. Nope.

After banging my head a bit, I finally realized that it was in the home directory of `root`. Bruh.

![[just_addfinal.png]]
