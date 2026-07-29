---
title: HackTheBox The Salt Crown CTF - RINg the Bell (Pwn)
draft: false
date: 2026-07-29
---
This challenge was a part of HackTheBox's The salt crown CTF that ran from `24/07/2026` to `29/07/2026`. It was in the pwn category. I solved it while playing with Arizona State University's "CTF Academy" team.

# Challenge description
*"Crownspire's fire-watch has one bell pattern that clears a street faster than any order could: three short tolls, a pause, then two long ones, the call for a granary fire, when every free hand is expected to run for water instead of standing post. Rin got the pattern out of a watchman who'd had one drink too many. She doesn't need the gate garrison dead or even distracted for long, just gone, for exactly as long as it takes Keir's wagon to cross from the tannery gate to the river road with cargo nobody's supposed to see. The door to the bell tower wasn't built to keep strangers out. It was built to keep whoever's already inside from leaving on their own terms, which tells her plenty about who usually comes through here. She has one chance to get past it before that wagon reaches the river road. Get the pattern wrong, or ring it a beat late, and every guard in earshot will know there's no fire while Keir's cargo sits exposed in the open street. Breach the lock, climb to the bell, and ring the toll exactly as she memorized it, then get out before anyone starts asking why the granary isn't burning."*

# Solution Walkthrough
After downloading the challenge files and unzipping them, I noticed that there was only one file `ring_the_bell`. When I run it, I am shown the following:

![[Pasted image 20260729163439.png]]

An ASCII art of a ringing bell (shocker) and a statement along with a prompt.

If I tried to input any random data, the program just ends.

![[Pasted image 20260729163603.png]]

Not really much to work with so I opened it in the best reversing tool there is, binary ninja, to take a look around.

![[Pasted image 20260729163727.png]]

Here we can see a few functions that are not called directly by main.
If we take a peek into `bell` we can see:
![[Pasted image 20260729163818.png]]

It gives us a shell! Now if only we can return to this function somehow...

When we ran the binary, we were prompted sooo that means our input gets taken in. That was in the `main` function and by inspecting the disassembly:
![[Pasted image 20260729164018.png]]
We can see that `read` uses `rbp-0x20` as a buffer to read input into. And we already know that the return address is placed right after the saved `rbp` pointer.
That means that we would need `0x20+0x8` of random data and then our return address.

Since `read` takes in `0x60` bytes, this is quite possible. Also `checksec` shows us that this binary is not a PIE, meaning that we don't need to worry about any random addresses. And we can see that there is no canary!
![[Pasted image 20260729164326.png]]

This makes it a straightforward solution using `pwntools`:
```python
from pwn import *

p = remote("154.57.164.75", "31557")

# bell() at 0x40176d
p.send(b"a"*0x28 + p64(0x40176d) + b"\n")

p.interactive()
```

And boom we get a shell and we can `cat` the flag.
