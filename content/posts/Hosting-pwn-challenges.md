---
title:  "Hosting pwn challenges: a simple tutorial"
date:   2021-06-26 17:28:35 +0800
author: "152334H"

lastmod:   2021-06-26 17:28:35 +0800

draft: false
subtitle: ""
description: "It's really not hard."
tags: [ "pwn", "docker", "my setups" ]
categories: [ "Tech" ]

---

<!--more-->

In light of recent [ev](https://docs.google.com/document/d/1nGMHg4F4jEzvL-RaqpQtxIF4Da_Gxycxqru1VWTP6CI/edit)[en](https://seanseah.tech/writeups/2021/06/25/CDDC.html)[ts](https://www.notion.so/sheepymeh/CDDC21-Review-f239e9f81a32434f8e7af3053c9c74e8), I've been convinced to share a basic tutorial on how _You_ --- the ordinary programmer with minimal Linux/Docker/Networking experience --- can host [binary exploitation](https://caprinux.github.io/lawofpwn/prologue/whatispwn) challenges with minimal hassle and reasonable security.

## Requirements
* A stable linux server with the <a href="" title="If you're not using something debian-based, you wouldn't need this guide.">packages</a> `docker-ce` and `git` installed.
* One (or more) pwn challenge(s) you intend to host 
* Basic linux experience

That's it. 

## Setting up a challenge folder
Let's say you have a binary named [`chal`](https://github.com/152334H/pwntutorial/blob/master/ret2libc/ret2libc.c), as well as a `flag`.

We'll start by [git cloning](https://docs.github.com/en/github/creating-cloning-and-archiving-repositories/cloning-a-repository-from-github/cloning-a-repository) a [`template folder`](https://github.com/152334H/ctf_xinetd):
```bash
git clone https://github.com/152334H/ctf_xinetd
```
Move the `chal`lenge binary & `flag` into `ctf_xinetd/bin`:
```bash
mv chal flag ctf_xinetd/bin
```
If you plan on hosting multiple pwn challenges, you might want to rename the cloned folder:
```bash
mv ctf_xinetd my_uniq_chal_name
```
The final result should look something like this:
```
/path/to/my_uniq_chal_name$ tree
.
├ bin
│   ├ chal
│   └ flag
├ ctf.xinetd
├ Dockerfile
├ README.md
├ setup.sh
└ start.sh
```
## Putting the challenge online
Spend a few minutes to come up with a snazzy port number (`$PORT`) for your challenge.

To get your challenge running, execute
```bash
/path/to/my_uniq_chal_name$ ./setup.sh $PORT chal my_uniq_chal_name
```
After that, you're done! Using the example challenge linked:
```bash
$ nc localhost $PORT
Type something with 7fdde7ac8e10: hi
You put: hi
```
If you ever need to change the challenge file, you can update the running service by running `./rebuild.sh`.

## Benefits of using this
* it's really simple and you can't mess it up (fingers crossed)
* shell permissions are limited and you won't have players [messing with the shell](https://media.discordapp.net/attachments/823190327576756314/857630922513842216/unknown.png) or [erasing flags](https://media.discordapp.net/attachments/837571570876284929/857952303838527498/unknown.png) halfway through a CTF
* you run an `.sh` file controlled by me without ever reading its contents

## Downsides of using this
* might not scale very well
* hard to justify getting paid by the hour for something this simple
* some people just don't like docker

[Sieberrsec](https://github.com/IRS-Cybersec/) has been, and will be using this setup for Sieberrsec CTF events, so if you find any vulnerabilities in my fork of `ctf_xinetd`, be sure to ~~save them until the next CTF~~ **dutifully inform us** if you find anything of note.
