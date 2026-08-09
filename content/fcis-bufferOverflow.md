---
title: Ain Shams CTF - Buffer Overflow I
draft: false
date: 2026-08-09
---
This challenge was a part of "Cyber CUP Ain Shams CTF" that was held for 24 hours on **08/08/2026**.
# Challenge Description

![[buff.png]]

Category: `secure coding`
# Solution
This was a super simple challenge. For this one we were provided with the following files:

![[Pasted image 20260810002005.png]]

I looked at the provided `README` first and this is what it had:

![[Pasted image 20260810002059.png]]

This should be simple enough. The `challenge.c` had this code:

![[Pasted image 20260810002212.png]]

I spotted the problem right away. This is a classic buffer overflow where the user could enter a huge input, much larger than the allocated `BUFFER_SIZE` and enter anything malicious that would later end up on the stack.\
The main reason this is possible is due to using `strcpy` that copies everything from `src` to `dst` stopping only once it encounters a null terminator `\0`.

To fix this, there are two things to do. First, use `strncpy` to copy the exact number of bytes we want. Second, put the null terminator ourselves at the end of the `buffer`.

All in all, this is the solution:

![[Pasted image 20260810003121.png]]

Like I mentioned, I used `strncpy` with a size of `BUFFER_SIZE-1` so I can leave the last byte for the null terminator that I put into the buffer at the beginning.

Running the checker gave me the flag:

![[Pasted image 20260810003235.png]]