---
title: "Blog Refurbishment"
date: 2022-08-13T05:30:32+01:00
author: "152334H"

lastmod: 2022-08-13T05:30:32+01:00
featuredImagePreview: ""
featuredImage: ""

weight: 1
draft: false
subtitle: "Under new management"
description: "Under new management"
tags: [ "blog", "hugo", "my setups" ]
categories: [ "tech" ]

---
<!--more-->
{{< image src="Screenshot_2022_0816_111551.jpg" scale=50 caption="The site used to look like this. Ugly, right?">}}

To my grand audience of absolutely nobody, I present: this blog, but better!

I've been putting this off for a while. I don't like to think of myself as a web designer, and I didn't have compelling reasons to work on setting up anything more complex than the [default recommended Jekyll+Pages]() setup.

This changes today. I'll be out of the military soon, and that means I need a :glowing_star: stunning :glowing_star: website for employers to see. Plus, I've gotten the writer's bug as of recent, and I want a place to throw my words on that doesn't look as bad as v1 of this site.

## Finding Hugo
Like many other Rust developers, I read [Faster Than Lime](https://fasterthanli.me). Their blog looks _really nice_, and I wanted to copy it, so I looked for their [blog post on how they developed their site](https://fasterthanli.me/articles/a-new-website-for-2020).

Answer: really complex stuff! But then I read this part:
> I definitely remember using [nanoc](https://nanoc.ws/) (Ruby), and switching to [jekyll](https://jekyllrb.com/) (Ruby) at some point. When I launched the Patreon integration, I was using [hugo](https://gohugo.io/).
>
> Now, I really don't want to be spending any time saying negative things about projects in this post - not when there's so much cool stuff to show off! So all I'll say is this: hugo was easy to get started with, and it stayed relatively out of my way.
>
> I forked one of the more minimal themes, and added the pieces I wanted: cool bear's hot tip, some navigation, etc.

Well, that's not exactly an endorsement, but I'll take it over the less trodden path.

Let's google, "[Hugo themes](https://themes.gohugo.io)":

{{< image src="Pasted%20image%2020220820141438.png" caption="Cropped for brevity; there are hundreds of these themes on the page." >}}

That one on the right looks pretty good! Let's try it.

### Papermod
{{< image src="Pasted%20image%2020220820142042.png" scale=80 >}}

Fast and pretty good! Setup worked entirely as expected, except...

1. I wanted to mix the centered "profile mode" (pictured above) with a post list, but that wasn't a default option
2. Their TOCs uses the default Hugo style. It doesn't look pretty enough:
   ![](Pasted%20image%2020220820142107.png)
3. Syntax highlighting was somewhat wrong. See the unhighlighted second `L`s here:
   ![](Pasted%20image%2020220820142221.png)

So I decided to look for another theme. I eventually settled with [LoveIt](https://hugoloveit.com/).

### Installing LoveIt
I faced many, many technical problems trying to get this theme to run. In order:
* After [installing the theme](https://hugoloveit.com/theme-documentation-basics/#22-install-the-theme) as a submodule, I got an error running `hugo server -D`:
    > `Module "LoveIt" is not compatible with this Hugo version;`
    
    This was confusing because I had the latest Hugo version (v0.101), and the stated required version for the _LoveIt_ theme was v0.64
* A round of googling later, and I figure out the error message actually happens because my `hugo` installation [wasn't compiled with the SCSS extension](https://gohugo.io/troubleshooting/faq/#i-get--this-feature-is-not-available-in-your-current-hugo-version). I try to download an `-extended` release, but I find out that the  `hugo` main repository does not publish `-extended` releases for `arm64` CPUs, which I need because I'm using a Raspberry Pi for this.
* I attempt to compile the extended version of hugo directly on my RPi. This fails, because `go install` requires more than 0.5 GB of memory, which my little tiny machine cannot offer.
* I attempt to cross-compile Hugo from a better machine. This fails with an abstruse error message.
* I eventually figure out that it failed because compilation requires  `go>=1.18`. After updating, the cross-compilation fails again, [because](https://github.com/gohugoio/hugo/issues/8257) Hugo depends on a C/C++ library that is more difficult to cross compile (this is the same reason that the main repository does not give `-extended` releases)
* I luckily find a repository (https://github.com/hugoguru/dist-hugo/releases) that has compiled `-extended` versions of Hugo for ARM. This ends the torment, mostly.

After that mess, it runs:
![](Pasted%20image%2020220820144145.png)
Hurray. Let's move on.

### Changes
So, what's new about the site? Is it just a CSS refresh?

Not exactly. Although I was mostly motivated by the idea of getting the site to look :sparkles: prettier :sparkles:, the new framework comes with several added features:
* Searching. The screenshot above shows this.
* Emojis, like the ones above.
* Post reactions! With the power of [giscus](https://giscus.app):
   ![](Pasted%20image%2020220820144933.png)
* [Shortcode](https://hugoloveit.com/theme-documentation-built-in-shortcodes/). I make good use of them in my [about](/about) page.
* Better development experience. You won't see this on the site, but the file/post structure for Hugo is much more pleasant than Jekyll was.

I also made a few small changes to the LoveIt theme for this site, mostly to embed images more easily. See [here](https://github.com/152334H/LoveIt/commits/master)

## Moving forward

I didn't do all of this designer work _just_ to let it rot like the last one. Moving forward, **I'll be pushing posts to this site weekly**, whether anyone reads them or not. It'll be somewhat like [Matt Rickard](https://matt-rickard.com/archive)'s blog: a regular stream of short posts on tech things. I have enough of a backlog to keep this up for several months, and I plan on going on ad infinitum.

