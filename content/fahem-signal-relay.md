---
title: FahemSec CTF - Signal Relay
draft: false
date: 2026-08-01
---
This was part of "FahemSec CTF 2026".

---
For this challenge, we're provided with a couple of files:

![[Pasted image 20260801205739.png]]

The main being `relay`.

It's always good to run the binary and see what it does:

![[Pasted image 20260801205843.png]]

Okay, I have no idea what "register uplink callsign" even means. So I'll just enter some random data and see what happens:

![[Pasted image 20260801205948.png]]

Well, it just died instantly.

I then opened binary ninja to see what I got:

![[Pasted image 20260801225232.png]]

The first thing that caught my eye was `data_4040a8` that very clearly is a function pointer. It also gets passed the address of our buffer `g_node`.

![[Pasted image 20260801225332.png]]

Well look at that! The buffer `g_node` that gets read into in the main function exists right before our function pointer. And `read`'s third argument is `0x180` which gives us more than enough to overwrite the address of our function pointer, making it point to any other function we want before it gets called.

Investigating further, there's a symbol called `relay_bootstrap` that upon looking at reveals a couple of hidden "functions" right after it:

![[Pasted image 20260801225654.png]]

Well, I say "hidden" because they didn't appear right away in the menu on the left.

Their names are really indicative. We can put something in `rdi` and zero out both `rsi` and `rdx` and then some function called `relay_exec_nr` which doesn't show straight away what it does here and finally `relay_syscall` that very clearly makes a `syscall` (the bytes `0f 05` confirm that this indeed is a `syscall`).

We can take those bytes and find some translator online, but what fun is that? We should take every chance we get to use `gdb`.

Opening the binary in `gdb`, we can have more sense of what the purpose of these function is:

![[Pasted image 20260801230541.png]]

Now, we can tell `relay_exec_nr` puts `0x3b` in `rax` and then jumps to the address that's in `rbx`. Well, every function of them actually ends in `jmp rbx`.

`0x3b` is the syscall number for `execve`. Might as well just hand me a plate of gold right now.

Taking a closer look at `relay_bootstrap` shows that it puts `rdi+0x28` into `rbp`, puts the address of `relay_dispatch` in `rbx` and then jumps to it.

`relay_dispatch` is super cool. It basically takes one address from `rbp+8` and jumps to it. This means that we can put a bunch of addresses in `rbp` and calling `relay_dispatch` every time would just make us jump to each and every address in `rbp` one by one.

Which is exactly what happens at the end of each one of these functions. Remember `jmp rbx`?\
`rbx` would already contain the address of `relay_dispatch` so it's just a matter of arranging the call order of each of these functions.

So what we have so far is:
1- We can execute any function we want by overwriting the pointer that is in `data_4040a8`.
2- We have a couple of really helpful functions that we can use to grab a shell.

Here's the script that I wrote for this challenge:
```python
from pwn import *
p = remote("t68-69228f19503f.chals.ctf.sd", 443, ssl=True)

pay = b"/bin/sh\x00" + b"a"*(0x20) + p64(0x401202) + p64(0x401216) + p64(0x404080) + p64(0x401220) + p64(0x401224) + p64(0x401228) + p64(0x40122f)

p.send(pay)
p.interactive()
```

The distance between the start of our buffer `g_node` and the function pointer `data_4040a8` that want to overwrite is 40 bytes.\
So I put `/bin/sh\x00`, which is 8 bytes, at the start of our buffer. This allowed me to be able to have a controllable string at an address that I already know which is `g_node` (`0x404080`).

That leaves us with an empty of space of 32 bytes which I just filled with garbage.

Like I mentioned, the distance between `g_node` and our function pointer is 40 bytes or `0x28` bytes. Recall how at the start of `relay_bootstrap` it puts the value of `rdi+0x28` in `rbp`.

In our case, `rdi` here is the address of our buffer `g_node` and adding `0x28` to it would make it point to our function pointer.

![[Pasted image 20260801231908.png]]

Then right at the start of `relay_dispatch` it adds 8 to `rbp` essentially making it point to right after our function pointer and jumps to whatever address is there.

![[Pasted image 20260801231938.png]]

In the function pointer, I put the address of `relay_bootstrap` which, like I mentioned, sets up `rbp` and calls `relay_dispatch` which in turn calls the address right after the function pointer.
Which in our case is the address of `relay_load_rdi`.

Right after it I put the address of the string we put at the start `/bin/sh\x00` so that it puts its address in `rdi`.

I then put `relay_zero_rsi`, `relay_zero_rdx`,  `relay_exec_nr` and `relay_syscall` after eachother which is essentially:
```C
execve("/bin/sh", NULL, NULL);
```

And that got me a shell which allowed me to cat the flag!

```
fahemsec{jump_oriented_dispatcher_relays_the_chain}
```

![[image.png]]