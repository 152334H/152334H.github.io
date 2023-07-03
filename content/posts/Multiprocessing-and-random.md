---
title:  "Multiprocessing and <code>random()</code>"
date:   2022-10-02 00:00:00 +0800
author: "152334H"

lastmod:   2021-10-02 00:00:00 +0800
draft: false
subtitle: "<code>srand()</code> but cooler"
tags: [ "code", "python" ]
categories: [ "Tech" ]

---

Let's say, for [some odd reason](https://huggingface.co/blog/stable_diffusion), you're hosting a Python service that:
1. takes requests that run large computations based on a randomly generated number (a "seed")
2. spreads work across multiple workers to handle many requests

Then, it's likely you'll introduce a subtle randomness bug that leads to duplicate seeds appearing. Let me explain:

<!--more-->

<iframe id="AudioNativeElevenLabsPlayer" width="100%" height="180" frameBorder="no" scrolling="no" seamless src="https://beta.elevenlabs.io/player/index.html?publicUserId=01e30727d011cacbba870a0524d6020b350e4855e45a2854e421a4f6152a9f30"></iframe>
<script src="https://beta.elevenlabs.io/player/audioNativeHelper.js" type="text/javascript"></script>

{{< admonition info "Regarding Reproducibility" false >}}
The following tests were done on a Ubuntu 20.04 LTS Hetzner instance using:
1. `numpy==1.22.4`
2. `python==3.8.10`

I additionally made efforts to test certain outcomes on Ubuntu 18 LTS with Python 3.6.9, albeit with much uglier logging due to the lack of f-string improvements.
{{</ admonition >}}

## Randomness: single-process single-threaded
```sh
$ python3 -c 'print(__import__("random").random())'
0.7457432716191408
$ python3 -c 'print(__import__("random").random())'
0.9558957103766177
$ # applies for numpy.random too!
$ python3 -c 'print(__import__("numpy").random.randint(9999))'
5388
$ python3 -c 'print(__import__("numpy").random.randint(9999))'
236
```

Typically, there's no need to fiddle with the internal configuration of Python's `random` module. Python seeds its Mersenne Twister thing with [os.urandom by default](https://docs.python.org/3/library/random.html#random.seed), and even if you're on some really obscure operating system, it seeds [by time](https://github.com/python/cpython/blob/main/Modules/_randommodule.c#L290) which is pretty hard to collide with milisecond accuracy:

```sh
$ # Test: Spawning 5 processes quickly by backgrounding (&) them
$ for i in `seq 1 5`
> do python3 -c 'print(__import__("random").random())' &
> done
0.4576075693594288
0.7804091843861601
0.9347442769592177
0.9520767305433322
0.12542097628111482
```

That's not to say that this is _cryptographically secure_ or anything, but for the purposes laid out in the introduction, it's more than sufficiently random for our purposes.

But, as with all things Python, things get a bit dicer as we scale.

## Multiprocessing and `fork()`
As explained on the [`multiprocessing` docpage](https://docs.python.org/3/library/multiprocessing.html#contexts-and-start-methods), the default methodology for spawning multiple Python processes on Linux is to abuse `os.fork()`.

The [fork syscall](https://www.man7.org/linux/man-pages/man2/fork.2.html), for those who don't know, is approximately "an OS function that creates a bit-for-bit copy of an existing process", with exceptions listed in the preceding link. The random state of a PRNG _should_ be stored somewhere in process memory, so it stands to reason that **multiple Python processes `fork()`ed from the same parent will have the same seed**.

We can test this theory with a simple script:
```python
import multiprocessing as mp
import random as r
import numpy as np

def f(_): return np.random.randint(1000), r.randint(0,999)

with mp.Pool(4) as p:
    for i,(np_v,py_v) in enumerate(p.map(f, range(50))):
        if not i & 0b11: print('-'*23)
        print(f'{i:2}: {np_v=:3} | {py_v=:3}')
```

TLDR: Run `np.random.randint` and `random.randint` with 4 python processes in parallel; print dividing line every 4 outputs.

The results from this script are fairly surprising:

```python
$ python3 rand_test.py
-----------------------
 0: np_v=711 | py_v=969
 1: np_v=893 | py_v=988
 2: np_v=650 | py_v=588
 3: np_v=539 | py_v=213
-----------------------
 4: np_v=711 | py_v=413
 5: np_v=893 | py_v=192
 6: np_v=650 | py_v= 72
 7: np_v=539 | py_v=895
-----------------------
 8: np_v=711 | py_v=625
 9: np_v=893 | py_v=133
10: np_v=650 | py_v=261
11: np_v=539 | py_v=401
-----------------------
12: np_v=711 | py_v=805
13: np_v=893 | py_v=995
14: np_v=650 | py_v=295
15: np_v=539 | py_v=429
-----------------------
16: np_v=237 | py_v=221
17: np_v=769 | py_v=661
18: np_v=136 | py_v=248
19: np_v=967 | py_v=505
-----------------------
20: np_v=237 | py_v=759
21: np_v=769 | py_v=659
22: np_v=136 | py_v=587
23: np_v=967 | py_v=562
-----------------------
24: np_v=237 | py_v=922
25: np_v=769 | py_v=365
26: np_v=136 | py_v=441
27: np_v=967 | py_v=755
-----------------------
28: np_v=802 | py_v=778
29: np_v=744 | py_v=886
30: np_v=359 | py_v=112
31: np_v=349 | py_v=839
-----------------------
32: np_v=237 | py_v=301
33: np_v=769 | py_v=483
34: np_v=136 | py_v=465
35: np_v=967 | py_v=937
-----------------------
```
If you will observe: The `numpy.random` outputs **repeat 4 times** (every 4 steps), while the randomness of vanilla `random` is secure across processes. So we can infer that:
1. `numpy.random`'s PRNG state is captured (pickled) by the multiprocessing library and copied identically across 4 workers;
2. `random`'s PRNG state is **not** duplicated. It could either be shared across all workers, or reseeded within each worker on startup.

To figure out the answer to (2), we can just modify the script to print out the initial state of the PRNGs on worker start:

```python
yes = True
def f(_):
	global yes
	if yes:
		s = np.random.get_state()
		np_state = hash((*s[1],*s[2:5]))
		py_state = hash(r.getstate())
		print(f'{np_state=:20}, {py_state=:20}')
		yes = False
	# ...
```

As expected, the `numpy.random` initial state is equivalent across all workers. Unexpectedly, the `random` initial states do not, and they remain different even when I extend the sleep duration to infinity, indicating a new `random._inst` state is initialised per worker:

```python
$ python3 scratch.py
np_state= 8542083115030040073, py_state= -509295089981894936
np_state= 8542083115030040073, py_state= 7542824820053631765
np_state= 8542083115030040073, py_state=-7860580465393152951
np_state= 8542083115030040073, py_state=-9195119932724983682
```

---

At this point I'm pretty confused. The `numpy` module was getting pickled, but the `random`  module was not. Could I force a pickling of the `random` state?

I tried extending the test script to cover more things:
```python
import time as t
import random as r
import multiprocessing as mp
import numpy as np

def f(r_pickled):
    s = np.random.get_state()
    np_rand_hash = hash(tuple((*s[1], *s[2:5])))
    py_rand_hash = hash(r.getstate())
    pi_rand_hash = hash(r_pickled.getstate())
    t.sleep(0.01)
    #
    return {
        'np_s': np_rand_hash,
        'py_s': py_rand_hash,
        'pi_s': pi_rand_hash,
        # all of these generate in range(10000):
        'np_v': np.random.randint(100000),
        'py_v': r.randint(0,99999),
        'pi_v': r_pickled.randint(0,99999)
    }

ITERS = 128
NUM_WORKERS = 4
with mp.Pool(NUM_WORKERS) as p:
    res = p.map(f, (r._inst for _ in range(ITERS)))
    print('Unique numpy.random states:', len(set(d['np_s'] for d in res)))
    print('Unique np.random.randint():', len(set(d['np_v'] for d in res)))
    print('Unique random states:', len(set(d['py_s'] for d in res)))
    print('Unique random.randint():', len(set(d['py_v'] for d in res)))
    print('Unique r._inst states:', len(set(d['pi_s'] for d in res)))
    print('Unique r._inst.randint():', len(set(d['pi_v'] for d in res)))
```

In short, I'm asking (over 128 iterations): how many unique PRNG states / random numbers are there across all processes?

And the answer is weird:
```python
$ python3 t.py
Unique numpy.random states: 32
Unique np.random.randint(): 32
Unique random states: 128
Unique random.randint(): 128
Unique r._inst states: 8
Unique r._inst.randint(): 8
```

1. if you use `numpy.random`, you get `ITERS/NUM_WORKERS` unique outcomes, (or `NUM_WORKERS` collisions of the same seed)
2. if you use `random`, the module, you get `ITERS` unique outcomes (regardless of number of processes)
3. if you _use the parent process's `random.Random` instance_, stored at `random._inst`, as an argument to a multiworker task, the `random.Random`  instance will get pickled, and the number of unique outcomes will be `ITERS/NUM_WORKERS/4`. This formula scales well with both varying `ITERS` and `NUM_WORKERS`.

So at this point, I'm doubly confused. I haven't figured out why `random` succeeds in re-initing state in new processes, and I now have a new question of what mechanism keeps the pickled `random._inst` PRNG state ticking forwards.

## Speculation
And so, I move into the fog of unverified ideas and poorly substantiated hypotheses.

---

### The `forkmethod` is important
I tried switching up the forkmethod from `fork` to `forkserver`/`spawn`

```python
if __name__ == '__main__':
	with mp.get_context("forkserver").Pool(4) as p:
		for i,(np_v,py_v,h_delta) in enumerate(p.map(f, range(50))):
		pass
```

In both cases, the random states of both `numpy` and `random` became different in all workers:

```sh
$ python3 scratch.py
np_state= -664522000653194140, py_state= 7612061459549224940
np_state= 4019719269418835448, py_state= -826723981683651496
np_state=-3004816782852048973, py_state=-3881656698949473802
np_state=  374882113652371457, py_state=  746934503431840062
```
Only the choice of `fork` causes the initial seeds of numpy to differ. Why? I'm not sure; you can start with [the source code here](https://github.com/python/cpython/tree/3.10/Lib/multiprocessing/), but it's not simple.

### `random` state is copied, but overwritten
The `pickle` representation of both  `numpy.random.randint` and `random.randint` are both large _and_ similar in size:
```python
>>> import pickle as p
>>> import numpy as np
>>> import random as r
>>> len(p.dumps(r.randint))
3804
>>> len(p.dumps(np.random.randint))
2825
>>> def f(): pass   # empty func
>>> len(p.dumps(f)) # as comparison
29
```

Plugging the outputs of **either**  `randint()` function into `pickletools.dis()` shows a large stored array of numbers, which I assume is representative of their PRNG states. If the multiprocessing library _does_ capture the state of `random` before execution, then something about the initialisation process of each worker must recreate the `random` state again.

Alternatively, the `random.randint()` function is not captured by the `multiprocessing` module entirely. But I find this difficult to believe, because it has to capture _something_ to run the `randint` function, and since `pickle` is unable to dump `module` types, I'm unable to come up with another idea here.

This also explains why explicitly capturing `r._inst` succeeds in creating duplicate seeds -- the `multiprocessing` library reinitiallises `random._inst`, but the function-provided `r_inst` object remains as a separate instance.

---

## In conclusion
1. Using `random.*` explicitly in a multiprocessed function will never go wrong. Speculation: the multiprocessing library recreates a `random.Random` object per worker.
2. Using `numpy.random` with multiprocessing using `fork` will cause predictable random number duplication. The PRNG state is captured by `pickle` and copied between processes.
3. Using _a copy of `random._inst`_ as an _argument_ for a multiprocessed task will cause even more duplication than the `numpy` option. Mechanism unknown.
