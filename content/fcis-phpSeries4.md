---
title: Ain Shams CTF - PHP series 4
draft: false
date: 2026-08-09
---
This challenge was a part of "Cyber CUP Ain Shams CTF" that was held for 24 hours on **08/08/2026**.
# Challenge Description

![[Pasted image 20260810042002.png]]

Category: `web security`
# Solution
This was the most interesting challenge for me that I solved in this CTF. Mainly because I had no prior experience using PHP so I took it as a personal challenge to try and tackle something unfamiliar.

Once the challenge launched, I was able to see the following web page:

![[Pasted image 20260810042053.png]]

A link that redirects to the page's source code, that's in php, and a field to enter some sort of a token.

I just clicked directly on the link where I was presented with:

![[Pasted image 20260810042038.png]]

The php code basically takes my input `receivedToken`, passes it to `isTokenValid` alongside the existing list of hashes `admin_hashes` and the global salt `salt` that gets appended to my input before it gets hashed using `md5`.\
This generated hash is then compared to the list of hashes using `in_array` to check if my hash exists in the list.

If my hash does exist, we get the flag.

The first thing that popped in my mind was I had to somehow find some input that when appended to the salt would cause a hash collision. I remembered that `md5` is known to be prone to hash collisions.

So I went to google and searched for "php md5 hash collisions" and found the following:
https://security.stackexchange.com/questions/261975/do-we-know-a-md5-collision-exploiting-php-loose-type-comparision-0123e2-123e
https://security.stackexchange.com/questions/261975/do-we-know-a-md5-collision-exploiting-php-loose-type-comparision-0123e2-123e

Basically in php, there's something called "magic hashes". These hashes are generated from hashing algorithms but are interpreted as something else by php. This would be especially useful for us here when loose comparison is involved which is exactly what happens with `in_array`.

In our case, if there's a hash that begins with `0e` followed by a number, and loose comparison is used, php would treat that hash as a scientific-notation number.

Example: `0e45646` would be interpreted as `0 x 10^45646` which results in zero.
So no matter what number exists after `0e`, it would result in `0`.

How is this useful to us? Because in the `admin_hashes` list, there is indeed a hash that begins with `0e` followed by a number.

So my next goal would be to find the input that would give me this kind of hash.


![[Pasted image 20260810042108.png]]

![[Pasted image 20260810042114.png]]