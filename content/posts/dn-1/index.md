---
title: "Disco Narrator - Data Scraping"
date: 2022-09-11T00:00:00+01:00
author: "152334H"

lastmod: 2022-09-11T00:00:00+01:00

subtitle: "Part 1/4 of a <a href=/disco-narrator>series</a>"
description: ""
tags: [ Disco Elysium, reverse engineering, part of a series, WIP ]
categories: [ "tech" ]
toc: true

---

To train an AI Text-To-Speech (TTS) model, we'll need to obtain a Labelled Dataset with two things:
1. Clean audio files (`.wav`s), containing only the voice we're cloning
2. The dialogue transcript (text) for each audio file

<!--more-->

I refer to these points as `(1)` and `(2)` in the next section.

## Asset extraction
Now, I've never Reversed Engineered a Unity game before. But I knew, at bare minimum, that getting (2) done was definitely possible, because of [Disco Reader](https://disco-reader.gitlab.io/disco-reader) -- a third-party app that renders dialogue trees from the game.

A bit of googling later, and I find an informative [reddit thread](https://www.reddit.com/r/DiscoElysium/comments/kz66p3/_):
> I needed a structured export of all the conversations in the game [...] **I now use [AssetStudio](https://github.com/Perfare/AssetStudio) to extract the dialoguebundle myself.**

While googling, I also found another project, [Disco Courier](https://github.com/htmlbanjo/disco-courier), which suggested much of the same:
> place a copy of your [exported data](https://github.com/Perfare/AssetStudio) in /data/dialog.json

So, I'm looking to get a `.json` file from AssetStudio somehow. Let's try that.

---

{{< admonition info "Using AssetStudio" false >}}
I [download](https://github.com/Perfare/AssetStudio/releases/) and open the program. It looks like this:

<p align="center">
<img src="Pasted%20image%2020220817194836.png">
</p>

Okay, "Load file/folder". The data has to be somewhere in the game files, so... I could try loading `Steam\steamapps\common\Disco Elysium\`, and look from there?

{{< image src="Pasted%20image%2020220817195617.png" caption="**uh oh**" >}}

A crash and a reboot later, and I realise I should've read the `README` in closer detail:
> When AssetStudio loads AssetBundles, it decompresses and reads it directly in memory, which may cause a large amount of memory to be used. You can use **File-Extract file** or **File-Extract folder** to extract AssetBundles to another folder, and then read.

So I extracted the folder elsewhere, and _then_ I tried `Load Folder`:

<p align="center">
<img src="Pasted%20image%2020220817195953.png">
</p>

The first thing I see are the audio files for the game. This is good -- we've solved for (1) -- but I still haven't found the dialogue data yet.

After clicking around in the UI a bit, I luck out again: sorting the assets by size, I find an asset named `Disco Elysium`, stored at `/Assets/Dialogue Databases/Disco Elysium.asset`. Loading the _preview_ for this asset nearly crashes my computer again because of how large it is -- **Every** piece of writing (dialogue, thoughts, item descriptions, etc.) in the **entire game** is bundled in this one asset. Exported, this amounts to a 266MB `.json` file, as we expected earlier.

{{< /admonition >}}

---

After extracting the audio files as well, I have the following data:
```bash
~/DiscoAudioSources$ tree 
.
├── AudioClip
│   ├── Abandoned Lorry-JAM  INSTIGATOR CABIN-118.wav
│   ├── Abandoned Lorry-JAM  INSTIGATOR CABIN-120.wav
│   ├── Abandoned Lorry-JAM  INSTIGATOR CABIN-123.wav
.....
│   └── Yellow Man Mug-INVENTORY  MUG-9.wav
└── MonoBehaviour
    └── Disco Elysium.json
```
`AudioClip/` contains the files for (1), `MonoBehaviour/` contains the lines for (2).

## Linking audio to dialogue
I had _hoped_ the `AudioClip` assets would contain some labelled metadata, but to my consternation, the audio files and dialogue text were bundled separately. So the `.wav` files themselves aren't very useful, and I need to gather more information.

Specifically, for each audio file, I need to know
1. Which voice actor is speaking, and
2. What lines are being read

### Dead-end: `disco-courier`
The first thing I tried was to locate the information in `dialog.json`. 

The [Disco Courier](https://github.com/htmlbanjo/disco-courier) project I mentioned earlier was made to work with it, so I started with an `npm install`:

{{< image src="Pasted%20image%2020220815203719.png" caption="only 3 critical!" scale=60 >}}

The app was a little bit old -- last commit in 2021. 1 year is about two centuries long in the NodeJS ecosystem, so the app obviously crashed on first try:

<p align="center">
<img src="Pasted%20image%2020220815203747.png">
</p>

So, I did a little bit of debugging and added the fixes to [my fork](https://github.com/152334H/disco-courier).

{{< admonition bug "Details of the bug, if you care" false >}}

To start, I grep the source for `the version of your data`, and quickly find the problem:

<p align="center">
<img src="Pasted%20image%2020220815203916.png">
</p>

The version is hardcoded... along with some metadata about the json file. I extracted these with a oneliner:
```bash
$ fx src/data/dialog.json 'd => ({version: d.version, rowCounts: { locations: d.locations.length, actors: d.actors.length, items: d.items.length, variables: d.variables.length, conversations: d.conversations.length }})'
{
  "version": "5/20/2022 12:05:57 PM",
  "rowCounts": {
    "locations": 0,
    "actors": 424,
    "items": 259,
    "variables": 10645,
    "conversations": 1501
  }
}
```
And patched that output into the source. After that, I tried running the project with one of the suggested example commands:
```bash
$ courier -- --export=json --actor=3 --OR=true --conversant=6 conversations.dialog
# "Creates a detailed json export where the speaker is Kim, OR the conversant is Garte."
```
Naturally, this didn't do what it said it would do. After fixing another minor bug (the output directory, `./src/data/json`, did not exist), I found a list of conversations from The Player to Cunoesse at `./src/data/json/conversations/conversations.dialog.json`.

{{< /admonition >}}

After accounting for bugs, I get a json file of dialogue entries from `courier`. A single dialog entry looks like this:
```json
{
  "parentId": 1039,
  "dialogId": 222,
  "isRoot": 0,
  "isGroup": 0,
  "refId": "0x0100005A0000E5F6",
  "isHub": false,
  "dialogShort": "Little Lily: \"\"Ll... Luby... Rr... R-luuby.\" Sudd...\"",
  "dialogLong": "\"Ll... Luby... Rr... R-luuby.\" Suddenly the girl gets all serious and leans in, as if she's about to tell you a secret.",
  "actorId": 101,
  "actorName": "Little Lily",
  "conversantId": 0,
  "modifiers": [],
  "conditionPriority": 2,
  "userScript": "",
  "inputId": "0x0100002100000B63",
  "outputId": "0x0100002100000B70"
}
```
And although there's a lot of information in that entry, there's nothing that tells me what audio file (if any) this line is linked to. Dead end.

### Further sleuthing
So, a third-party CLI app failed to give me the information I wanted. Maybe I should've been a little less credulous, and checked the game files myself?

Answer: no, `disco-courier` is doing about the best it can. The only thing I really notice is that what the CLI app calls `refId`s [are named `articyId`](http://pixelcrushers.com/dialogue_system/manual/html/articy_voice_over_plugin_details.html)s in the game files.

In desperation, I try to run `grep` on the raw assets to see if I'd get anything.
```bash
~/disco_Data$ grep 'Ruud Hoenkloewen-PLAZA  KORTENAER-74' *
Binary file sharedassets1.assets matches
```
To my surprise, it _did_ find something. `sharedassets1`; what's in there?
<p align="center">
<img src="Pasted%20image%2020220817202059.png">
</p>
A lot of things, unfortunately. Let's try harder.

---

{{< admonition info "Shared Asset Extraction" false >}}
As with before, let's start by sorting by size:
<p align="center">
<img src="Pasted%20image%2020220817202143.png">
</p>
That won't work; textures are big. Filter by <code>MonoBehaviour</code>?
<p align="center">
<img src="Pasted%20image%2020220817202224.png">
<br>
<i>Hello.</i>
</p>

I click on the asset, hit `Esc` after a file browser pop-up appears inexplicably, and I see:
<p align="center">
<img src="Pasted%20image%2020220817202335.png">
<br>
An empty object?
</p>
{{< image src="Pasted%20image%2020220817202349.png" >}}
No, wait. An error. I search through <a href="https://github.com/Perfare/AssetStudio/issues/986">Github Issues</a> and...

> please check https://github.com/Perfare/AssetStudio#export-monobehaviour

Ah. I failed to the documentation.

Again.

One installation of `Il2CppDumper.exe` later, and I create the fake `.dll` files AssetStudio is looking for:
```bat
C:\Program Files\Steam\steamapps\common\Disco Elysium\disco_Data>C:\Program Files\Il2CppDumper-net6-v6.7.25\Il2CppDumper.exe ..\GameAssembly.dll .\il2cpp_data\Metadata\global-metadata.dat C:\il2cpp_out
Initializing metadata...
Metadata Version: 27
Initializing il2cpp file...
Il2Cpp Version: 27
Searching...
Change il2cpp version to: 27.1
CodeRegistration : 18216e9c0
MetadataRegistration : 182173350
Dumping...
Done!
Generate struct...
Done!
Generate dummy dll...
Done!
Press any key to exit...

C:\Program Files\Steam\steamapps\common\Disco Elysium\disco_Data>
```
{{< /admonition >}}

---

Re-opening AssetStudio, I follow the README, and...

<p align="center">
<img src="Pasted%20image%2020220817202812.png">
</p>

It's there! `AssetName`, `ArticyID`. The `AssetName`s match up with the `.wav` filenames we extracted, and the `ArticyID`s match exactly to the expected dialogue for each audio file. Taking the first example in the image, `0x010000060001BDCA` refers to this part of the massive Dialogue `json`:
{{< figure src="Pasted%20image%2020220817203121.png" >}}
With all the information we need (theoretically) in tow, we can move on to [Data Formatting](/todo).
