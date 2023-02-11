---
title: "Fast (5x) Inference with TorToiSe-TTS"
date: 2023-02-05T20:28:55+08:00
author: "152334H"

lastmod: 2023-02-05T20:28:55+08:00
featuredImagePreview: ""
featuredImage: ""

draft: false
subtitle: "ignore this if you've already seen the repo before"
description: ""
tags: ["machine learning", "TTS"]
categories: ["tech"]

---

I made a fork of TorToiSe with much faster inference speed. Here are the summarised results:

<!--more-->

{{< admonition title="Example texts used" open=false >}}
A (70 characters):
> I'm looking for contributors who can do optimizations better than me.

B (188 characters):
> Then took the other, as just as fair,
> And having perhaps the better claim,
> Because it was grassy and wanted wear;
> Though as for that the passing there
> Had worn them really about the same,
{{</ admonition >}}

Original TorToiSe [repo](https://github.com/neonbjb/tortoise-tts):
| speed (B) | speed (A) | preset |
|-|-|-|
| 112.81s | 14.94s | ultra_fast |

New [repo](https://github.com/152334H/tortoise-tts), with `--preset ultra_fast`:
| speed (B) | speed (A) | GPT kv-cache | sampler | cond-free diffusion | autocast to fp16 |
|-|-|-|-|-|-|
|  118.61   |   11.20   | ❌           | DDIM    | ❌                  | ❌               |
|  115.51   |   10.67   | ❌           | DPM++2M | ✅                  | ❌               |
|  114.58   |   10.24   | ❌           | DPM++2M | ❌                  | ❌               |
|   55.76   |    7.25   | ❌           | DDIM    | ❌                  | ✅               |
|   53.59   |    6.77   | ❌           | DPM++2M | ✅                  | ✅               |
|   51.98   |    6.29   | ❌           | DPM++2M | ❌                  | ✅               |
|    9.86   |    4.24   | ✅           | DDIM    | ❌                  | ❌               |
|    8.51   |    3.77   | ✅           | DPM++2M | ✅                  | ❌               |
|    8.12   |    3.82   | ✅           | DPM++2M | ✅                  | ✅               |
|    6.78   |    3.35   | ✅           | DPM++2M | ❌                  | ✅               |

All results listed were generated with a slightly undervolted RTX 3090 on Ubuntu 22.04, with the following base command:

```sh
python tortoise/do_tts.py --voice emma --seed 42 --text "$TEXT"
```
