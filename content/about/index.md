---
title: "About"
date: 2022-08-12T13:09:50+01:00
author: "152334H"

lastmod: 2022-08-17T00:00:00+01:00

draft: false
subtitle: "So, tell me about yourself..."
description: ""
tags: [ personal ]
categories: [ "not a post" ]
toc: true

---

<!--more-->
{{< admonition info "Welcome">}}
This is the personal site of Sherman Chann (aka `152334H`) 
{{< /admonition >}}

## Stuff I've done
_Weakly sorted by chronology_

{{< admonition type=tip title="Psst..." open=false >}}
Each section starts with a few **logos** like these:

{{< icon name=neovim url="https://neovim.io/" title="This is my primary text/code editor." >}} 
{{< icon name=python url="https://python.org/" title="This is my main language of choice." >}}
{{< icon name=twitter url="https://twitter.com/" title="This is a social media platform I don't use." >}}

They indicate, roughly speaking, what technologies I used for a specific project. If you're confused about their relevance, try **hovering over them** (or long press on mobile): some of these icons have `title`s explaining what they're used for:

{{< image src="hover_example.png" >}}


{{< /admonition >}}

### This blog

{{< icon hugo "https://github.com/gohugoio/hugo" >}}
{{< icon name=github url="https://pages.github.com" title="The Github Pages logo looks really bad compared to this one." >}}
{{< icon name=obsidian url="https://obsidian.md" title="For first drafts" >}}
{{< icon name=neovim url="https://neovim.io/" title="For final posts, as well as future changes." >}}

You're looking at it!

This blog is hosted by [Github Pages](pages.github.com). I upload the source files for this blog (developed in [Hugo](https://gohugo.io/documentation/)) to [a repository](https://github.com/152334H/blog), triggering a [Github Action](/todo) that compiles everything into a bundle minified HTML/CSS/JS. This produces the static page you're reading right now.

Previously, I used [Jekyll](https://jekyllrb.com/docs/github-pages/) (instead of Hugo), but there were many unfortunate problems with that setup. [More on that here](/blog/Blog-refurbishment).

I use [Obsidian](https://obsidian.md) to draft out posts, before settling the post metadata && final checks in neovim while running `hugo server -D`. This isn't the greatest workflow, and I'm working on a file sync system to smoothen the process.

### Disco Narrator
{{< icon name=pandas url="https://pandas.pydata.org/" title="Dataset creation" >}}
{{< icon name=googlecolab url="https://colab.research.google.com" title="Used for GPU resources in training models" >}}
{{< icon name=pytorchlightning url="https://github.com/NVIDIA/NeMo" title="NeMo's TalkNet implementation." >}}
{{< icon name=flask url="https://palletsprojects.com/p/flask/" title="Backend routing. Also: uwsgi" >}}
{{< icon python >}}
{{< icon name=javascript title="I used vanilla HTML/CSS/JS for the frontend as an exercise" >}}
{{< icon name=html5 title="I used vanilla HTML/CSS/JS for the frontend as an exercise" >}}
{{< icon name=github url="https://github.com/SortAnon/ControllableTalkNet" title="Fork of this linked project" >}}

A simple web app ([demo](https://test.disco-narrator.xyz)) that uses AI to emulate the voices of a few video game characters. Forked from the [Controllable TalkNet](https://github.com/SortAnon/ControllableTalkNet) project. More information available in the following series:
{{< single_summary "/disco-narrator" >}}

### `react-viewer-viewer`
{{< icon typescript "https://www.typescriptlang.org/" >}}
{{< icon react "https://reactjs.org/" >}}
{{< icon mui "https://mui.com/" >}}
{{< icon mongodb "https://www.mongodb.com/" >}}
{{< icon express "https://expressjs.com/" >}}
{{< icon nodedotjs "https://nodejs.org/" >}}
{{< icon name=docker url="https://docker.com/" title="This runs the backend API." >}}

A small React.js Tauri [App](https://github.com/152334H/react-viewer-viewer/) (full demo [here](/react-viewer-viewer/), if the one below doesn't work) I developed to learn the basics of modern Full Stack development. <span style="color:grey">Keywords: typescript, material-ui, create-react-app.</span>

<iframe src="https://152334H.github.io/react-viewer-viewer" width="100%" height="500">
  <p>Your browser does not support iframes.</p>
</iframe>

Some screenshots:

{{< image src="Screenshot_2022_0817_102526.jpg" caption="Not very complex" scale=50 >}}
{{< image src="Screenshot_2022_0817_102654.jpg" caption="<i>Matryoshka screenshot</i>" >}}

I started the project while I was midway through the [Full Stack Open](https://fullstackopen.com/) course online, so some of the design decisions in the app are regrettable. I've also developed a small backend API for the app (<span style="color:grey">keywords: NodeJS, Express, MongoDB, docker-compose</span>), but I've yet to publish its source.

### Sieberrsec CTF 3.0 (2021)
{{< icon name=docker url="https://docker.com/" title="Used to run challenges / CTF platform" >}}
<a href="https://aws.amazon.com/lightsail/" title="We used a Lightsail instance to host challenges / CTF Platform.">
        <svg xmlns="http://www.w3.org/2000/svg" width="2em" height="2em" preserveAspectRatio="xMidYMid meet" viewBox="0 0 256 221"><path fill="#ffffff" d="M145.726 0c29.308 0 56.876 11.364 77.702 32.027c20.88 20.771 32.462 48.394 32.571 77.865c.108 29.472-11.256 57.203-32.028 78.083c-20.77 20.88-48.393 32.462-77.865 32.57h-.435c-53.994 0-99.67-38.497-108.696-91.676a5.843 5.843 0 0 1 4.785-6.742c3.208-.49 6.199 1.577 6.743 4.785c8.102 47.524 48.937 81.943 97.168 81.943h.38c26.318-.109 51.06-10.44 69.601-29.145c18.542-18.705 28.71-43.5 28.601-69.818c-.108-26.317-10.44-51.058-29.145-69.6c-18.596-18.488-43.228-28.601-69.437-28.601h-.38c-48.177.163-88.958 34.745-96.897 82.215a5.814 5.814 0 0 1-6.743 4.785a5.814 5.814 0 0 1-4.785-6.743C45.838 38.878 91.405.218 145.291 0h.435Zm6.416 30.396l8.972 8.156c1.25 1.142 31.048 28.656 32.516 69.165c.924 25.829-9.95 50.732-32.244 74.114l-9.407 9.842l.38-13.649c.055-2.066.598-49.535-46.871-61.552c-2.828-.707-4.785-3.263-4.84-6.145c0-2.936 1.904-5.492 4.731-6.253c45.676-12.342 46.893-58.195 46.925-61.382v-.17l-.162-12.126ZM94.504 142.3c0 10.93 8.863 19.738 19.738 19.738c-10.875 0-19.684 8.809-19.738 19.738c0-10.93-8.863-19.738-19.738-19.738c10.93 0 19.738-8.863 19.738-19.738Zm67.045-87.762c-2.882 15.77-12.126 42.522-41.815 55.572c29.96 12.615 39.15 39.476 41.923 55.462c14.954-18.541 22.186-37.79 21.479-57.474c-.816-23.87-13.213-43.174-21.587-53.56ZM217.5 115.55l.004.316c.168 8.335 6.954 15.016 15.33 15.016c-8.482 0-15.333 6.851-15.333 15.334a15.316 15.316 0 0 0-15.334-15.334a15.316 15.316 0 0 0 15.334-15.334l-.001.002ZM40.781 105c2.882 0 5.275 2.338 5.275 5.274c0 2.882-2.338 5.274-5.275 5.274h-9.298c-2.882 0-5.274-2.338-5.274-5.274c0-2.882 2.338-5.274 5.274-5.274h9.298Zm-26.154 0c2.882 0 5.274 2.338 5.274 5.274c-.054 2.882-2.392 5.274-5.274 5.274H5.274c-2.881 0-5.274-2.338-5.274-5.274C0 107.392 2.338 105 5.274 105h9.353Zm51.33 0c2.882 0 5.275 2.338 5.275 5.274c-.055 2.882-2.393 5.274-5.275 5.274H54.81c-2.882 0-5.274-2.338-5.274-5.274c0-2.882 2.338-5.274 5.274-5.274h11.147Zm23.273 0c2.882 0 5.274 2.338 5.274 5.274c0 2.882-2.338 5.274-5.274 5.274h-9.353c-2.882 0-5.274-2.338-5.274-5.274c0-2.882 2.338-5.274 5.274-5.274h9.353Zm25.012-68.078a14.225 14.225 0 0 0 14.247 14.246a14.225 14.225 0 0 0-14.247 14.246v-.022l-.003-.292a14.224 14.224 0 0 0-13.929-13.93l-.315-.003a14.225 14.225 0 0 0 14.246-14.246Z"/></svg>
</a>
{{< icon name=amazons3 url="https://aws.amazon.com/s3/" title="Used for challenge file distribution" >}}
{{< icon name=discord url="https://this.url/was_omitted_to_prevent_botting" title="Used to run the CTF" >}}
{{< icon name=python title="Used to develop/test/solve challenges">}}
{{< icon name=rust title="Used to develop challenges">}}
{{< icon name=c title="Used to develop challenges">}}

Together with my friends at [IRS Cybersec](https://github.com/IRS-Cybersec), we hosted _Sieberrsec CTF 3.0_<sup>1</sup>, an online competition with >200 players, targeting Singaporeans in Secondary/Pre-Teritary Education.

{{< image src="Sieberrsec_CTF_3.0.png" caption="<b>Promotional image from 2021</b>.<br>The link works as of 08/2022, but won't stay up forever." >}}

I was personally responsible for:
* Challenge development: I made the most number of challenges, tested even more of them, and did my best to simplify/balance them in favour of the participants. I also handled the hosting/testing for most remote challenge microservices.
* QA/Admin details: CTF Platform testing, Discord announcements/organisation, post-event cleanup && writeup collection.

It takes a lot more than that to run a CTF, but I'll leave that information to [a future post](/todo).

<sup>1</sup> - 1.0 was a school event, 2.0 was nominally public but primarily catered to a few schools.

### CVEs (at [Star Labs](https://starlabs.sg/))
{{< icon name=python title="Used to develop exploits">}}
{{< icon name=c title="Binary source code reading. I would've put the IDA Pro logo here if it wasn't copyrighted.">}}

[CVE-2021-34978](https://www.zerodayinitiative.com/advisories/ZDI-21-1240/)

[CVE-2021-34979](https://www.zerodayinitiative.com/advisories/ZDI-21-1241/)

### `pwnscripts` (2020-2021)
{{< icon python >}}
{{< icon git >}}
{{< icon name=githubactions url="https://github.com/features/actions" >}}
{{< icon name=pypi url="https://pypi.org/" title="Python packages." >}}

[`pwnscripts`](https://github.com/152334H/pwnscripts) is a deprecated Python [package](https://pypi.org/project/pwnscripts/) I developed as a personal extension of [pwntools](https://github.com/Gallopsled/pwntools). 

[![Tests](https://github.com/152334H/pwnscripts/workflows/Python%20package/badge.svg)](https://github.com/152334H/pwnscripts/actions) [![PyPI package](https://badge.fury.io/py/pwnscripts.svg)](https://pypi.org/project/pwnscripts/) [![Python](https://img.shields.io/pypi/pyversions/pwnscripts)](https://www.python.org/downloads/)

This was the first **serious** project I created. It's not big -- a few thousand Lines of Code in total -- but it got me working with technologies that I had felt were for Real™ Developers Only. To summarise:
* I experienced the process of creating, developing, and publishing Python libraries to PyPi
* I went through Test Driven Development with the [pytest](https://docs.pytest.org) framework.
* I made use of [Github Actions](https://github.com/features/actions) to do the two things above automatically after pushing commits.
* I tried switching to [vscode](https://code.visualstudio.com/) because I realised plain old `vim` wasn't enough for modern development (although later I switched back to neovim after understanding its plugin ecosystem better).
* I got to understand first-hand why people don't recommend using Python for <span title="I'm talking about pwntools, not my comparatively tiny project">large projects</span> (spoiler: `__magic_methods__` are exceedingly bad for code readability!)

And I learned a lot about programming in general, along the way. But I'm not working on this project anymore, because...

### CTF player (2019-2021)
{{< icon name=gnubash title="Specifically: hundreds of CLI tools that I would've never learned to use otherwise.">}}
{{< icon name=python title="CTF scripts are notoriously poorly written from an SWE perspective. I did my best to write clean exploit code for the public." >}}
{{< icon name=c title="Too much knowledge about glibc and the C standard library functions." >}}
{{< icon name=markdown title="For writeups">}}

I used to be _very_ active in the CTF scene. My old [CTFTime profile](https://ctftime.org/user/75701) and my team's [writeup repository](https://github.com/IRS-Cybersec/ctfdump) is full of **hundreds** of challenges I've covered over the years. I won prizes for several local competitions in that time period, including:
* 1st in CDDC 2020 (Junior Category)
* 1st in Cyberthon 2020
* 2nd in Whitehacks 2020
* 1st in STACK The Flags 2020 (Junior Category)
* 1st in Whitehacks 2021
* 3rd in DSO-NUS CTF 2021
* 1st in CTF.SG CTF 2021

[I am no longer a CTF player](/todo). I'm happy to assist juniors/newcomers in learning the fundamentals, but I'm effectively retired from the local cadre of CTF professionals.

### Advent of Code
{{< icon name=gnubash title="For a CLI utility to interact with Advent of Code's API." >}}
{{< icon python >}}
{{< icon javascript >}}
{{< icon c >}}

[Advent of Code](https://adventofcode.com) was my introduction to the world of programming puzzles. My secondary school didn't offer much in the way of programming (beyond the [regrettable](/todo) _Hello World_ tutorials), and I might not have become the programmer I am today without Eric Wastl's work.

| Year | Points (global) | Languages |
|---|---|---|
| [2018](https://github.com/152334H/advent) | 0 :( | Python, C |
| [2019](https://github.com/152334H/advent2) | 37 | Python, JS, C |
| [2020](https://github.com/152334H/aoc_2020) | 28 | Python |
| [2021](https://github.com/152334H/aoc) | 372 | Python |

Although I started playing Advent of Code to learn programming, I mostly do it for the fun of it, nowadays.

### Other programming problems
{{< icon python >}}
{{< icon cplusplus >}}
{{< icon rust >}}

[![Badge](https://cp-logo.vercel.app/codeforces/152334H)](https://codeforces.com/profile/152334H)
[![Leetcode Badge](https://img.shields.io/badge/Leetcode-2181-blue.svg)](https://leetcode.com/152334H/)
[Kattis](https://open.kattis.com/users/sczs2)

While I'm not a Real™ Competitive Programmer, I play contests every now and then for fun, as well as to keep myself qualified for basic coding interviews.

### Open source contributions
{{< icon gnubash >}}
{{< icon python >}}
{{< icon rust >}}

{{< image src="Screenshot_2022_0817_104846.jpg" scale=40 >}}

I've made minor contributions to other people's repositories. See my [GitHub profile](https://github.com/152334H) for more info.

## Academics 
I'm an alumnus of [Hwa Chong Institution](https://www.hci.edu.sg/) (College), and spent most of my free time there helping out the [Cybersecurity Section](https://irscybersec.ml/) of the Infocomm and Robotics Society ([IRS](https://infocommsociety.com/)).

Some info that will only make sense to Singaporean eggheads: PCMH+H3 Chem, AAAA/AD+Dist, SSEF, SChO.
I include this for completion's sake; most readers probably don't care about this.

## My religious beliefs
Ubuntu, neovim, tabs > splits, tmux, docker, python3 (4 spaces per indent), Rust > C*, typescript, printf-debugging, `Casing_doesNotMatter`, iterators > indexing, Copilot, industry collapse by 2030.
