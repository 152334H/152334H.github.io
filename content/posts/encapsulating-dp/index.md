---
title: "@cache without @cache"
date: 2022-08-28T08:02:23+01:00
author: "152334H"

lastmod: 2022-08-28T08:02:23+01:00
featuredImagePreview: ""
featuredImage: ""

draft: false
subtitle: "Encapsulating dp[] within functions"
description: ""
tags: ["code", "python"]
categories: ["tech"]

---

[Memoization](https://cs.stackexchange.com/a/99517) is a part of the standard toolkit for, "things I can use to solve the algorithm question in my next job interview". Most of the time, I like to use [`functools.cache`](https://docs.python.org/3/library/functools.html#functools.cache) for this:

{{< image src="cts_lru_cache.png" caption="From @gf_256" scale=50 >}}

<!--more-->

Using the [fibonacci](https://en.wikipedia.org/wiki/Fibonacci_number) function as an example, a simple Python implementation might look like this:

```python
# With @cache
from functools import cache # python3.9
@cache
def fib(n: int) -> int:
  return n if n < 2 else fib(n-1) + fib(n-2)
# Assume that `fib(n)` only receives non-negative integers.
```

However, [some interviewers](https://leetcode.com/discuss/general-discussion/1561340/has-anyone-used-python-cache-in-an-interview) might reject that One Weird Trick, given that the point of most coding interviews is to check for one's _understanding_ of the algorithm, rather than its existence.

So, instead of importing a solution, you'd manually create/check a hashtable, typically named `dp[]`:

```python
# Without @cache
dp = {0:0, 1:1} # base cases
def fib(n: int) -> int:
  res = dp.get(n,None) # avoid using `n in dp` to
  if res is None:      # minimise calls to dp.__getitem__
    dp[n] = res = fib(n-1) + fib(n-2)
  return res
```

And this works just as well as the `@cache` solution:

```python
>>> [fib(i) for i in range(10)]
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

Job well done? Probably.

But personally, don't like it. The `@cache`d solution is simple, readable, and intuitive. It's practically identical to how you'd write it mathematically:

$$ \forall\ n > 1, F_n = F_{n-1} + F_{n-2}$$

The manual `dp[]` solution is not as nice. The base cases are a bit more obvious, but the meat of the function is a lot more different. Plus, the `dp[]` variable gets leaked to the global namespace, when it should really only be accessible to the `fib()` function itself. Can we do better?

## Alternatives
I want to bind a stateful variable (`dp[]`) to a function. The natural tool for that in python is an _object_, so let's do that:

```python
# Without @cache, using a class
class Fib:
    dp = {0:0, 1:1}
    def fib(n: int) -> int:
      res = Fib.dp.get(n,None)
      if res is None:
        Fib.dp[n] = res = Fib.fib(n-1) + Fib.fib(n-2)
      return res
fib = Fib.fib
```

This _works_ (as in, `dp{}` is no longer a global variable), but it's much uglier. Can we do better?

Did you know that in Python, functions are objects too?

```python
# Without @cache, using the function itself as a class
def fib(n: int) -> int:
  res = fib.dp.get(n,None)
  if res is None:
    fib.dp[n] = res = fib(n-1) + fib(n-2)
  return res
fib.dp = {0:0, 1:1}
```

You can assign any attribute to a function, and it'll _just work_. Albeit with potential performance issues.

Still, both of these options leaves `.dp` available to the public namespace. Can we hide it entirely?

In theory, this is what you'd use a closure for:
```python
# Without @cache, using a closure
def _make_fib():
  dp = {0:0, 1:1}
  def fib(n: int) -> int:
    res = dp.get(n,None)
    if res is None:
      dp[n] = res = fib(n-1) + fib(n-2)
    return res
  return fib
fib = _make_fib()
```

But this is really long and ugly. We can separate the business logic with the _decorator_ pattern:

```python
# Without @cache, using a decorator
def dpcache(f):
  dp = {}
  def inner(n):
    res = dp.get(n,None)
    if res is None:
      dp[n] = res = f(n)
    return res
  return inner

@dpcache
def fib(n: int) -> int:
  return n if n < 2 else fib(n-1) + fib(n-2)
```
And, in doing so, return to the original `@cache` design for the `fib()` function. This is even somewhat similar to what [@cache itself](https://stackoverflow.com/a/49883466) does, the _Least Recently Used_ part aside.

---

The bootleg `@dpcache` solution might be the cleanest, but it also has the largest number of lines of all the solutions suggested. If we're comfortable with making our code look ugly, we can use a hack with default variables instead:
```python
# Without @cache, using a default variable
def fib(n: int, *, dp={0:0,1:1}) -> int:
  res = dp.get(n,None)
  if res is None:
    dp[n] = res = fib(n-1) + fib(n-2)
  return res
```

This is very close to the essence of our initial design for manual memoization, while also keeping the `dp[]` dictionary encapsulated. The positional glob (`*`) limits the potential for an accidental `fib(n, ?)` call, but it's possible someone else might unwittingly add the `dp=` argument after reading `fib()`'s type signature.

The use of default arguments in a singleton fashion this way is also, practically speaking, too smart for its own good -- it's a Code Smell, albeit in a different manner than having global variables.
