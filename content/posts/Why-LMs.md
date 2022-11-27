---
title: "Why Language Models?"
date: 2022-11-27T12:00:00+08:00
author: "152334H"

lastmod: 2022-11-27T12:00:00+08:00
featuredImagePreview: ""
featuredImage: ""

draft: false
subtitle: "Part 0/X"
description: "A preface to the upcoming series on my attempts to use language models, locally"
tags: ["language models", "personal", "machine learning"]
categories: ["tech"]

---

<!--more-->

Over the course of the last month or so, I've been working on a [webapp text editor](https://github.com/152334H/gpt-j-editor) that uses [GPT-J](https://huggingface.co/EleutherAI/gpt-j-6B), a language model, to perform autocomplete. If you don't know what that is, then [I hope you'll enjoy](https://xkcd.com/1053/) some of the links in this blogpost.

But for the majority that _does_ know what language models are, it's safe to say that I've done nothing complex. Technologically, I made a [React app](https://create-react-app.dev/) with a text editor derived from [Slate.js](https://docs.slatejs.org/), and connected that to a [FastAPI backend](https://fastapi.tiangolo.com/) which throws requests to huggingface's [`transformers`](https://huggingface.co/docs/transformers/index).

None of this is revolutionary. There are many solutions online that do way better. EleutherAI hosts their own free [demo page](https://20b.eleuther.ai/) that runs a lot more elegantly than my webapp. OpenAI's [GPT3 models](https://beta.openai.com/playground/) are a lot better than anything open source can provide. And companies like [NovelAI](https://novelai.net/) corner the _submarket_ of people who want to do more specific tasks like writing certain kinds of fiction novels.

So, why am I even working on any of this?

---

## Models need less vram now

Back in August, someone published a paper titled [LLM.int8(): 8-bit Matrix Multiplication for Transformers at Scale](https://arxiv.org/abs/2208.07339). I recommend reading the [huggingface article](https://huggingface.co/blog/hf-bitsandbytes-integration) on it if you're interested, but in short, 

> With our method, a 175B parameter 16/32-bit checkpoint can be loaded, converted to Int8, and used immediately without performance degradation.

Now, 175B parameters is still pretty big. With Int8, it'd be _175 gigabytes of memory_, which is still well in the category of "not for personal use".

But the improvements apply for _any_ language model reliant on the [transformer architecture](https://medium.com/@adityathiruvengadam/transformer-architecture-attention-is-all-you-need-aeccd9f50d09). And there are many great models that are now accessible to larger sections of the general population because of this.

|     Model     | 3050 (4GB) | 2080 TI (11GB) | Tesla T4 (16GB) | 3090 (24GB) |
|---------------|------------|----------------|-----------------|-------------|
| Codegen-2B    | :check_mark_button: | :black_large_square: | :black_large_square: | :black_large_square: |
| GPT-J-6B      | :cross_mark: | :check_mark_button: | :black_large_square: | :black_large_square: |
| CodeGeeX-13B  | :cross_mark: | :cross_mark: | :check_mark_button: | :check_mark_button: |
| GPT-NeoX-20B  | :cross_mark: | :cross_mark: | :cross_mark: | :check_mark_button: |

<p align="center">
:check_mark_button: - int8 improvement | :black_large_square: - no change | :cross_mark: - int8 insufficient
</p>

If you have an [RTX 3090](https://www.techpowerup.com/gpu-specs/geforce-rtx-3090.c3622), you can run [GPT-NeoX-20B](https://nn.labml.ai/neox/utils/llm_int8.html) or [CodeGeeX-13B](https://github.com/THUDM/CodeGeeX). If you have a 2080, you can run GPT-J-6B or [Incoder-6B](https://huggingface.co/facebook/incoder-6B). And if you have enough memory to run Stable Diffusion, you can run [Codegen-2B](https://github.com/salesforce/CodeGen).

That last example is particularly motivating, because of the next section.

## Advances in sampling strategies

Earlier this month, huggingface implemented [Contrastive Search](https://huggingface.co/blog/introducing-csearch) into their `transformers` library. While I'm not at all qualified to describe what it does (and whether it is 'novel' or 'obvious'), I find their [results](https://arxiv.org/pdf/2210.14140) rather encouraging.

![](https://user-images.githubusercontent.com/54623771/201234331-7ca646f7-a3f4-448c-94e2-707a1eea235c.jpg)

A 3% jump might not sound like much, but **it puts CodeGen-2B at the same level as [Codex-2.5B](https://github.com/VHellendoorn/Code-LMs#results---humaneval)**. This puts open source replacements for Copilot (like [fauxpilot](https://github.com/moyix/fauxpilot)) at the same level of code completion competency.

Contrastive search also does a lot better at long-form writing than other sampling strategies, which is great because:

## I wanted to write blogposts again

I'm not very good at writing. While I don't think the things I publish are _terrible_, I often feel that I take way too long to get from 'idea' to 'written essay'. And I'm sure that's not a unique problem, but it's the kind of problem that a lot of people seem to shrug at and say, 

> Guess I have to try harder.

Or,

> Guess I can't do much of that

I don't like either of these options. The third option, "Make an computer do it for you," is what language models are. But I also don't really like sending my drafted blogposts to a remote SaaS, so I wanted a solution that could run locally on my own hardware.

And that was surprisingly difficult to find online. I did some googling, asked a few communities, double checked a laundry list of github tags to make sure I didn't miss anything, and somehow I _just found nothing_. I'm still 90% certain someone has already done, "Open source webapp editor that uses GPT-J," but for the life of me, I couldn't find it. The searches I got were [polluted](https://github.com/nhaouari/obsidian-textgenerator-plugin) with [solutions](https://github.com/jameshiew/nvim-magic) that, while open source, were only designed to send requests to OpenAI's GPT3 API. Great for most people; not what I'm looking for.

So, I got to work on a simple tool that would help me to run GPT-J locally, thinking it would take me less than a weekend to finish.

---

The next few blogposts in this series will cover how I ended up spending a month doing just that.

