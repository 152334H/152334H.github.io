---
title: "An Informal Reminder"
date: 2022-12-02T05:15:20+08:00
author: "152334H"

lastmod: 2022-12-02T11:45:20+08:00
featuredImagePreview: ""
featuredImage: ""

draft: false
subtitle: "Tapping the sign in futility"
description: ""
tags: ["opinion", "language models"]
categories: ["opinion"]

---

{{< figure src="/blog/an-informal-reminder/meme.png" >}}

<!--more-->

## No one is immune to the consequences of an Artifical General Intelligence.

Programmers are not special. They are knowledge workers. [The Pile](https://pile.eleuther.ai/) contains everything, including an enormous amount of code. The idea that all other careers would sink to AI, while programmers would flourish, was myopic, and hopefully now sufficiently discredited.

Also, the [litigation](https://githubcopilotlitigation.com/) will not save you:

{{< figure src="2211.results.png" >}}
<p align="center">
<a href=https://arxiv.org/abs/2211.15533><b>The Stack</b>: 3 TB of permissively licensed source code</a>
</p>

For reference, these are the [HumanEval results](https://github.com/VHellendoorn/Code-LMs) for the rest of Codex:

|Model|Pass@1|Pass@10|Pass@100|
|------|-----|-----|-------|
| Codex (300M) | 13.17% | 20.37% | 36.27% | 
| Codex (2.5B) | 21.36% | 35.42% | 59.50% | 
| Codex (12B) | 28.81% | 46.81% | 72.31% |

---

ChatGPT is currently [free for public use](https://chat.openai.com/chat). There are [known cases](https://twitter.com/kurumuz/status/1598380913993211904) where it fails as a programmer. There are legitimate limitations to the current approach of throwing LLMs at everything. But regardless of their limitations, these models [will look an awful lot](https://jmcdonnell.substack.com/p/the-near-future-of-ai-is-action-driven) like AGI.

#### Some fun examples

* [Designing a pizza REST API](https://cdn.discordapp.com/attachments/730095596861521970/1047949631109218464/Screenshot_20221201-135703.png)
* [Building a Tokio API with MongoDB](https://cdn.discordapp.com/attachments/730095596861521970/1047595621923688538/Screenshot_2022-11-30_at_20.30.55.png)
* [Exploiting a buffer overflow](https://twitter.com/moyix/status/1598081204846489600)
* [Explaining the attention mechanism in transformers](https://cdn.discordapp.com/attachments/730095596861521970/1047677207771881473/imagen.png)
* [Creating shapes with js](https://cdn.discordapp.com/attachments/730095596861521970/1047631121363517470/image.png) ([source](https://cdn.discordapp.com/attachments/730095596861521970/1047631043949232228/image.png))
* [A comparison with Google](https://twitter.com/jdjkelly/status/1598021488795586561)
* [`longwest_incweasing_subseqence(aww)`](https://cdn.discordapp.com/attachments/730095596861521970/1047854144817479711/Screenshot_2022-12-01_at_13-37-45_ChatGPT.png)
* The samples from [OpenAI](https://openai.com/blog/chatgpt/#samples)
* [Decompiling assembly](https://twitter.com/sagitz_/status/1598426274157961229)
* [Fixing and Diagnosing problems in AWS IAM Policies](https://twitter.com/iangcarroll/status/1598171507062022148)
* [Making a React login page](https://cln.sh/X4p01n)
* [Bubblesort explained by a 40's gangsta](https://twitter.com/goodside/status/1598129631609380864)

