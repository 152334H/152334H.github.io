---
title: "Append a newline to your shell prompt"
date: 2022-09-04T04:31:35+01:00
author: "152334H"

lastmod: 2022-09-04T04:31:35+01:00
featuredImagePreview: ""
featuredImage: ""

draft: false
subtitle: ""
description: ""
tags: ["my setups"]
categories: ["tech"]
toc: false

---
The default shell prompt (for Ubuntu 20) looks like this:
```bash
username@hostname:/path/to/cwd$ █
```
<!--more-->
The format of the prompt is defined in `.bashrc`, as the `PS1` variable:
{{< image src="Screenshot_20220802_152338.jpg" scale=70 >}}

If you add a newline around the end of the `PS1` defintion, your shell will start looking like this (after re-running `bash`):
```bash
username@hostname:/path/to/cwd
$ █
```

---

#### Why do this?
1. More screen width for a one-liner. You can write longer commands before you get disrupted by line-wrapping. Especially useful if you're [using Termux](/todo).
2. Easier to read the prompt. Your eyes naturally capture text above&below the line you're working on, so you get to see more without shifting attention.
3. It's an extremely simple thing to do. It's cumbersome and potentially disruptive to install `$your_favourite_shell_theme` every time you touch a remote machine, whereas adding `\n` to `.bashrc` takes about ten seconds in a text editor.
