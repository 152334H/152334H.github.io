---
title: "Disco Narrator - Data Formatting"
date: 2022-09-25T00:00:00+01:00
author: "152334H"

lastmod: 2022-09-25T00:00:00+01:00

subtitle: "Part 2/4 of a <a href=/disco-narrator>series</a>"
description: ""
tags: [ Disco Elysium, pandas, part of a series, WIP ]
categories: [ "tech" ]
toc: true

---

With the raw data in tow, we can construct a proper TTS Dataset with the use of a few Python scripts.

<!--more-->

I currently have:
1. named audio files (`AudioClip/`)
2. dialogue lines with IDs (`dialog.json`)
3. IDs that are linked to the names of audio files from (1) (`VoiceOverClipsLibrary.json`)

I want to squash (1) and (2) together -- to create dialog lines linked to audio files -- and we need to use (3) to get there.

## Preprocessing
My idea was simple:
1. Each line of dialog from `dialog.json` has an `articyID`.
2. Each `articyID` is linked to an `assetName`
3. Each `assetName` represents a unique `.wav` file
4. Run through steps 1-3 to obtain `(dialog, wav_file)` pairs.

Or at least, it was supposed to be simple. In reality, each and every one of these steps were **assumptions** -- expected conditions that weren't strictly followed by the raw data I had obtained in the first post.

Here's what went wrong:

### Useless `dialogueEntries`
Each [DialogueEntry](https://www.pixelcrushers.com/dialogue_system/manual2x/html/class_pixel_crushers_1_1_dialogue_system_1_1_dialogue_entry.html) in `dialog.json` contained an `articyID`, which was great. What was less great was the presence of useless/meta dialogue entries like this:
```json
{
  "id": 0,
  "fields": [
    {
      "title": "Title",
      "value": "START",
      "type": 0,
      "typeString": ""                                            
    },
    {
      "title": "Articy Id",
      "value": "0x0000000000000000",
      "type": 0,
      "typeString": ""                                            
    },
    {
      "title": "Sequence",
      "value": "Continue()",
      "type": 0,
      "typeString": ""
	}
  ],
  ...
}
```
So, instead of looping through every dialogue entry in the game, I threw every entry into a [pandas](https://pandas.pydata.org/) dataframe and ran `articyID` queries on that:
```python
with open(JSON_DIALOG) as f:
  loaded = json.load(f)
  df_actors_init = pd.json_normalize(loaded,record_path=['actors'])
  df_actors = pd.concat([df_actors_init, df_actors_init.pop('fields').apply(iterate)], axis=1)
  df_convos_init = pd.json_normalize(loaded,record_path=['conversations','dialogueEntries'])
  df_convos = pd.concat([df_convos_init, df_convos_init.pop('fields').apply(iterate)], axis=1)

# To access a dialogueEntry with a given `ArticyID`,
# access `df_convos[df_convos["Articy Id"] == ArticyID]`
```
This was a somewhat inefficient solution and I am not very proud of it.  More on this later, in the final script.

### One-to-many mapping: `ArticyID` --> `AssetName`
Problem: in `VoiceOverClipsLibrary.json`, some `clipInformation[]` entries contain `alternativeVoiceClips[]`. Example:
```json
{
  "AssetName": "Empathy-KINEEMA  SYLVIE-198",
  "ArticyID": "0x0100002B00060B58",
  "AssetBundle": "kineema_empathy",
  "PathToClipInProject": "Assets\\Sounds/Dialogue/VOImports\\kineema\\empathy\\Empathy-KINEEMA  SYLVIE-198.wav",
  "DoesNotNeedVO": false,
  "alternativeVoiceClips": [
    {
      "AlternativeID": 0,
      "AlternativeAssetName": "alternative-0-Empathy-KINEEMA  SYLVIE-198-0",
      "AlternativeClipPath": "Assets\\Sounds/Dialogue/VOImports\\kineema\\empathy\\Alternative\\alternative-0-Empathy-KINEEMA  SYLVIE-198-0.wav",
      "DoesNotNeedVO": false
	},
	{
      "AlternativeID": 1,
      "AlternativeAssetName": "alternative-1-Empathy-KINEEMA  SYLVIE-198-1",
      "AlternativeClipPath": "Assets\\Sounds/Dialogue/VOImports\\kineema\\empathy\\Alternative\\alternative-1-Empathy-KINEEMA  SYLVIE-198-1.wav",
      "DoesNotNeedVO": false
	},
	{
      "AlternativeID": 2,
      "AlternativeAssetName": "alternative-2-Empathy-KINEEMA  SYLVIE-198-2",
      "AlternativeClipPath": "Assets\\Sounds/Dialogue/VOImports\\kineema\\empathy\\Alternative\\alternative-2-Empathy-KINEEMA  SYLVIE-198-2.wav",
      "DoesNotNeedVO": false
	},
	{
      "AlternativeID": 3,
      "AlternativeAssetName": "alternative-3-Empathy-KINEEMA  SYLVIE-198-3",
      "AlternativeClipPath": "Assets\\Sounds/Dialogue/VOImports\\kineema\\empathy\\Alternative\\alternative-3-Empathy-KINEEMA  SYLVIE-198-3.wav",
      "DoesNotNeedVO": false
	}
  ]
}
```
The full object has only 1 `articyID`, but many `AssetNames`. And all of these AssetNames exist as `.wav` files, and correspond to their own separate dialogue lines:
<img src="Pasted%20image%2020220817205406.png">
So, that's not great. I have to create additional code to handle this edge case.

### 404 not found: `AssetName` -/-> `.wav` file
Some `AssetName`s pointed to `.wav` files that simply didn't exist.
```
WARNING: Unable to find .wav file for: 'Titus Hardie-WHIRLING F1  TITUS HARDIE barks-2'
WARNING: Unable to find .wav file for: 'Titus Hardie-WHIRLING F1  TITUS HARDIE barks-3'
... (<1000 lines omitted) ...
WARNING: Unable to find .wav file for: 'Echo Maker-APT  ECHO MAKER barks-11'
WARNING: Unable to find .wav file for: 'Echo Maker-APT  ECHO MAKER barks-12'
```
The vast majority of these were labelled as `barks`, which from what I gather refers to [this](https://www.pixelcrushers.com/dialogue_system/manual2x/html/tutorial_bark.html). Generally speaking, they aren't actual voice lines and it doesn't hurt the dataset to simply ignore their presence (or lackthereof).

### One-to-many mapping: `AssetName` --> `DialogueEntry`
A small number of `clipInformation[]` entries contained duplicate `AssetName`s, despite having different `articyID`s:
<img src="Pasted%20image%2020220828102321.png">
These appear mostly linked to extra dialogue lines added from _The Final Cut_, and are distinguished by the presence of a `fixed-` prefix in their `PathToCLipInProject`:
```json
{
  "AssetName": "Kim Kitsuragi-WHIRLING  KIM MAIN-55",
  "ArticyID": "0x0100000400008D23",
  "AssetBundle": "whirling_kim-kitsuragi",
  "PathToClipInProject": "Assets\\Sounds/Dialogue/VOImports\\whirling\\kim kitsuragi\\fixed-Kim Kitsuragi-WHIRLING  KIM MAIN-55.wav",
  "DoesNotNeedVO": false,
  "alternativeVoiceClips": []
},
```
So I implemented extra corner case code for this too.
```python
def parseVoiceOver(vo):
  ID,AN,PATH = vo['ArticyID'], vo['AssetName'], vo['PathToClipInProject']
  expected_fname = PATH.split('\\').pop()
  if expected_fname[:-4] != AN: # INACCURATE AN DETECTED
    AN = expected_fname[:-4]
    assert AN[:6] == 'fixed-' # this is a very specific kind of file.
  ...
```
### 404 not found: `articyID` -/-> `DialogueEntry`
After fixing all of the above, and making my first attempt to squash all the data, I noticed there were also a few unused (either that or my code was incorrect) `.wav` files detected:
```python
Building final csv...:   0%|                                                                                                          | 1/45545 [00:00<2:38:26,  4.79it/s]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Inland Empire-WHIRLING F2  TEQUILA DOOR-5.wav')]
Building final csv...:   0%|▏                                                                                                         | 102/45545 [00:01<07:45, 97.59it/s]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Inland Empire-WHIRLING F2  DREAM 2 INTRO-40.wav')]
Building final csv...:   3%|███▌                                                                                                     | 1533/45545 [00:16<07:32, 97.27it/s]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Composure-JAM  TOMMY-776.wav')]
Building final csv...:  13%|█████████████▊                                                                                           | 5982/45545 [01:03<07:05, 92.98it/s]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Kortenaer-PLAZA  KORTENAER-525.wav')]
Building final csv...:  23%|████████████████████████                                                                                | 10519/45545 [01:53<06:26, 90.60it/s]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Garte, the Cafeteria Manager-WHIRLING F1  GARTE MAIN-120.wav')]
Building final csv...:  49%|███████████████████████████████████████████████████                                                     | 22388/45545 [04:11<04:40, 82.59it/s]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Siileng-JAM  HUMANITARIAN AID-482.wav')]
Building final csv...:  49%|███████████████████████████████████████████████████▍                                                    | 22505/45545 [04:12<04:35, 83.49it/s]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Siileng-JAM  HUMANITARIAN AID-479.wav')]
Building final csv...:  49%|███████████████████████████████████████████████████▍                                                    | 22532/45545 [04:13<04:35, 83.45it/s]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Siileng-JAM  faln sneakers on a pedestal of speakers-88.wav')]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Siileng-JAM  faln sneakers on a pedestal of speakers-94.wav')]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Siileng-JAM  faln sneakers on a pedestal of speakers-99.wav')]
Building final csv...:  50%|███████████████████████████████████████████████████▍                                                    | 22550/45545 [04:13<04:29, 85.40it/s]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Siileng-JAM  box of clothes-64.wav')]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Siileng-JAM  box of clothes-67.wav')]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Siileng-JAM  box of clothes-74.wav')]
Building final csv...:  50%|███████████████████████████████████████████████████▌                                                    | 22568/45545 [04:13<04:26, 86.27it/s]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Siileng-JAM  box of sun glasses-77.wav')]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Siileng-JAM  box of sun glasses-83.wav')]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Siileng-JAM  box of sun glasses-87.wav')]
Building final csv...:  57%|███████████████████████████████████████████████████████████                                             | 25844/45545 [04:53<04:07, 79.57it/s]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Kim Kitsuragi-WHIRLING F1  GARTE MAIN-553.wav')]
Building final csv...:  62%|████████████████████████████████████████████████████████████████▎                                       | 28138/45545 [05:22<03:45, 77.19it/s]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Kim Kitsuragi-VILLAGE  POSSE 2-74.wav')]
Building final csv...:  82%|█████████████████████████████████████████████████████████████████████████████████████▋                  | 37550/45545 [07:25<01:45, 75.45it/s]
WARNING: unused audio clip(s) [PosixPath('AudioClip/Kim Kitsuragi-WHIRLING  KIM MAIN-981.wav')]
Building final csv...: 100%|████████████████████████████████████████████████████████████████████████████████████████████████████████| 45545/45545 [09:16<00:00, 81.81it/s
```
Considering how few of these there were, I deliberately chose to move on instead of investigating further.

### Final result
After accounting for all of those intricacies, I end up with this final script:
```python
#!/usr/bin/python3
import os
import json
import pandas as pd
from pathlib import Path
from tqdm import tqdm

# this is for df.append()
import warnings
warnings.simplefilter(action='ignore', category=FutureWarning)


### filepaths
os.chdir('../0_original_data')
JSON_DIALOG = './dialog.json'
JSON_VOCLIPS = './VoiceOverClipsLibrary.json'
WAV_AUDIOCLIPS_FOLDER = Path('./AudioClip')


### List /AudioClip/ folder
audio_assets = {wav_file.stem:wav_file for wav_file in WAV_AUDIOCLIPS_FOLDER.iterdir()}
print('loaded audio clips folder')


### Load VoiceOverClipsLibrary into clips{}
clips = {}
def aasset(k):
    v = audio_assets.get(k,None)
    if v is None: print(f"WARNING: Missing '{k}'")
def parseVoiceOver(vo):
    ID,AN,PATH = vo['ArticyID'], vo['AssetName'], vo['PathToClipInProject']
    expected_fname = PATH.split('\\').pop()
    if expected_fname[:-4] != AN: # INACCURATE AN DETECTED
        AN = expected_fname[:-4]
        assert AN[:6] == 'fixed-' # this is a very specific kind of file.
    path = audio_assets.get(AN,None)
    if path is None: return print(f"WARNING: Unable to find .wav file for: '{AN}'")

    clips[ID] = [path]
    for alt in sorted(vo['alternativeVoiceClips'], key=lambda alt: alt['AlternativeID']):
        #assert alt['AlternativeClipPath'].split('\\').pop()[:-4] == alt['AlternativeAssetName']
        # Alt assets do not appear to have the "fixed-" problem.
        clips[ID].append(audio_assets[alt['AlternativeAssetName']])

with open(JSON_VOCLIPS) as f:
    loaded = json.load(f)
    df_vo = pd.json_normalize(loaded, record_path=['clipInformations'])
    filtered = df_vo.filter(['AssetName','ArticyID','alternativeVoiceClips','PathToClipInProject'])
    filtered.apply(parseVoiceOver,axis=1)
print('parsed VoiceOverClipsLibrary')


### Load dialog.json into DataFrames
def iterate(values):
    return pd.Series({x["title"]: x["value"] for x in values})

with open(JSON_DIALOG) as f:
    loaded = json.load(f)
    df_actors_init = pd.json_normalize(loaded,record_path=['actors'])
    df_actors = pd.concat([df_actors_init, df_actors_init.pop('fields').apply(iterate)], axis=1)
    df_convos_init = pd.json_normalize(loaded,record_path=['conversations','dialogueEntries'])
    df_convos = pd.concat([df_convos_init, df_convos_init.pop('fields').apply(iterate)], axis=1)
print('Read huge dialog json file')


### Create processed csv file
df_final = pd.DataFrame(columns=[
    'fname',
    'acticyID',
    'alternativeIdx',
    'text',
    'actorID',
    'actorName'
]) # shape of the output csv

for aID,clip_ls in tqdm(clips.items(),desc='Building final csv...'):
    dialogueEntries = df_convos[df_convos["Articy Id"] == aID]
    if dialogueEntries.shape[0] == 0:
        print(f'WARNING: unused audio clip(s) {clip_ls}')
        continue
    assert dialogueEntries.shape[0] == 1 # make sure there's only 1 entry
    dialogueEntry = dialogueEntries.iloc[0] # get the entry
    actor = df_actors[df_actors.id == int(dialogueEntry.Actor)].iloc[0]
    for i,path in enumerate(clip_ls):
        text = dialogueEntry['Alternate%d'%i] if i else dialogueEntry['Dialogue Text']
        df_final = df_final.append({
            'fname': path.name,
            'acticyID': aID,
            'alternativeIdx': i,
            'text': text,
            'actorID': actor.id,
            'actorName': actor.Name,
        }, ignore_index=True)

df_final.to_csv(r'AudioClipMetadata.csv', index=False)
```
The script is single-threaded, because I don't know how to properly operate `pandas`. After about fifteen minutes of waiting, I end up with a `.csv` file that looks like this:
<img src="Pasted%20image%2020220818175859.png">
Audio files, lines, and actor names. Great.

## Packaging
Now, there are a few problems with the audio transcript as-is:
* Unaccepted punctuation. `*asterisks*` for emphasis, dashes (`--`) and ellipses (`...`) for pauses, quotation marks (`""`) -- these are all readable by humans, but poorly handled by the models I'll be using.
* Fantasy words. Oranje, Revachol, Kineema, radiocomputer, Reál, Isola... Some people have the volition to sift through entire datasets, manually assigning the right pronunciations to each word. I am not one of those people.
* Lines that just shouldn't exist in any audio dataset. For example:
   > Half-Finished Paperwork: KK57-0803-0815 (THE HANGED MAN)
    
   > Pale Latitude Compressor: "236189281... If you're looking for a deal on mattresses... SUHSUHSUHSPEEDFRRRR... 23567... 32971047302819... Oh Rosaline, oh Rosaline..."

But I'm going to ignore these problems for now. We have close to 50k lines of speech, and given about half of those are by the Narrator, that's >20k lines of training data. I _assume_ that's enough to get a passable first model, even with problems in the dataset.

So, ignoring those issues for the future I want to format my raw data for an AI blackbox to eat. Which AI? From what little I know, [Tacotron 2](https://github.com/NVIDIA/tacotron2) is one of the simplest and cheapest models to train. Starting with it sounds like a good idea. Let's read the [tutorial](https://colab.research.google.com/github/NVIDIA/NeMo/blob/stable/tutorials/tts/Tacotron2_Training.ipynb#scrollTo=htbJiaJjYQAD):

{{< figure src="Pasted%20image%2020220818193200.png" >}}

Well, this is quite nice. I don't really know what a _phoneme_ is (yet), and much of the audio I have is significantly longer than 10 seconds, but the other 4 bullet points seem to be OK.

The data format shown above is also delightfully simplistic. The main problem for us is that it requires a `"duration"` field, which our current raw data does not declare. This isn't much of a problem; it's trivial to extract the info from each `.wav` file:
```python
import csv
import wave
import contextlib
from json import dumps
from multiprocessing import Pool
AUDIO_DIR = './AudioClip/'

# https://stackoverflow.com/a/7833963
def duration(fname: str) -> float:
    with contextlib.closing(wave.open(fname,'r')) as f:
        frames = f.getnframes()
        rate = f.getframerate()
        return frames / float(rate)
# 2.2565208333333335 == duration('./AudioClip/ANTI OBJECT TASK FORCE_TITLE.wav'))

# ['fname', 'acticyID', 'alternativeIdx', 'text', 'actorID', 'actorName']
def csvToJson(args):
    fname, text = args[0], args[3]
    t = duration(AUDIO_DIR+fname)
    return dumps({'audio_filepath': fname, 'text': text, 'duration': t})

CORES = 8
with open('./AudioClipMetadata.csv') as csvfile, Pool(CORES) as pool:
    reader = csv.reader(csvfile)
    next(reader) # ignore header
    for json_line in pool.imap(csvToJson, reader, 1<<10):
        print(json_line)
        exit()
```
Here, I've made the outlines of a script to create each expected `json` line with multithreading. I leave an `exit()` in the main loop so that I can test that the script approximately works without parsing everything yet:
```python
$ python3 duration.py
{"audio_filepath": "Inland Empire-BOOKSTORE  BIOGRAPHIES-29.wav", "text": "You feel like you should get this one. Definitely. It's *important* somehow. There is something personal inside...", "duration": 7.793041666666666}
```
Format looks good. But I'm not looking to use _all_ of the voices here; just the voices of the narrator. I need to figure out which voice actors are narrated and which aren't.

Step 1 is to extract a list of actors. I use `disco-courier` to export `actors.json`:
```json
$ fx disco-courier/src/data/json/actors/actors.json .actors[100]
{
  "actorId": 101,
  "refId": "0x0100002F0000F968",
  "name": "Little Lily",
  "shortDescription": "",
  "longDescription": "",
  "isPlayer": false,
  "isNPC": true,
  "isFemale": true
}
```
`isPlayer`/`isNPC` looks promising. Can we filter for those?
```json
$ fx '.actors.filter(d => !d.isNPC)[0]' < actors.json
{
  "actorId": 20,
  "refId": "0x01000014000003D8",
  "name": "Large Scab",
  "isPlayer": false,
  "isNPC": false,
  "isFemale": false,
  "voice": null
}
$ fx '.actors.filter(d => d.isNPC).slice(-1)' < actors.json
[
  {
    "actorId": 424,
    "refId": "0x0100000800000BBC",
    "name": "Perception (Sight)",
    "shortDescription": "",
    "longDescription": "",
    "isPlayer": false,
    "isNPC": true,
    "isFemale": false
  }
]
$ fx '.actors.filter(d => d.isPlayer)' < actors.json
[
  {
    "actorId": 396,
    "refId": "0x010000000000057C",
    "name": "You",
    "isPlayer": true,
    "isNPC": false,
    "isFemale": false
  }
]
```
Well, this is clearly useless. `isNPC` covers narrated actors like `Perception` _and_ other voice actors like `Kim Kitsuragi`, while `isPlayer` covers absolutely nothing of value.

Reading `actors.json` further, I realise that there are actors like `Ceiling Fan` or `"Smallest Church in Saint-Saëns"` that are actually voiced by the main Narrator too:
```bash
$ jq .actors[].name src/data/json/actors/actors.json | tail -100
"Door, Basement Apartment"
"The Great Doorgunner Megamix"
"Coffee Table"
"A Note from the Fridge"
"Shimmering Wall of Vices"
"Photo of Tattoos"
"\"Smallest Church in Saint-Saëns\""
"Karaoke Stand"
"A Folded Library Card"
```
So, `disco-courier` is ineffective here, and we have no idea which `actorID` corresponds to the Narrator of the game, so we'll need to...

... `*deep sigh*` ...

...manually label the data.

## Filtering for voice actors
The voice actors can be split into 5 categories for our purposes:
1. NPCs (like Kim Kitsuragi)
2. Narrator (obviously)
3. Not sure (I don't have an encyclopedic knowledge of the game; I'll ignore a few actors for now and check again later)
4. 404 Not Found. Interestingly, several voice actors listed in `actors.json` do not actually have any voice lines.
5. Mixed voices. Many of the voice lines labelled to be by an NPC are actually narrated. Consider these lines, labelled to be by `Cunoesse`:
    {{< figure src="Pasted%20image%2020220819155102.png" >}}
    The top 5 are narrated; the bottom five aren't. The full solution to this is _not_ as simple as, "look for quotation marks", either. Scare/rhetorical quotes are a thing.

I started work by editing the `.json` file in a text editor, but quickly realised it was _ridiculously slow_. Instead, I created a buggy script to minimise the time delay between each label:
```python
import pandas as pd
from prompt_toolkit import prompt
from prompt_toolkit.key_binding import KeyBindings
from termcolor import cprint
from json import load,dump,dumps

with open('./actors.json') as f: actors = load(f)['actors']
df = pd.read_csv('./AudioClipMetadata.csv')

res = []
def handle(opt: str):
    print()
    if not opt: print(dumps(res))
    elif opt == 'back': res.pop()
    elif len(actors) > len(res): res.append({**actors[len(res)], 'narratorType': opt})
    else: print('press q bro')
    #
    while 1:
        if len(actors) == len(res): break
        a = actors[len(res)]
        cprint(a['name'], 'magenta', None, ['bold', 'underline'])
        desc = a.get('longDescription','')
        if desc: cprint(desc, 'yellow')
        else: cprint('No description', 'grey')
        #
        lines = df[df['actorID'] == a['actorId']]
        if lines.shape[0]:
            print(lines)
            break
        else:
            cprint('No lines found for this Actor. Skipping...', 'grey')
            res.append({**actors[len(res)], 'narratorType': 'Absent'})
    print('> ', end='')

bindings = KeyBindings()

def bind(c: str, out: str):
    @bindings.add(c)
    def _(e): handle(out)
bind('enter', '')
bind('b', 'back')
bind('1', 'NPC')
bind('2', 'Narrator')
bind('3', 'Unknown')
bind('4', 'Mixed')

@bindings.add('q')
def finish(e):
    with open('actors_by_narrator.json', 'w') as f: dump(res,f)
    exit()

prompt("Press enter to begin: ", key_bindings=bindings)
```
This script breaks extremely often, but it was enough for me to create the labelled data within a few hours:
```bash
$ jq .[].narratorType actors_by_narrator.json | head
"Absent"
"Absent"
"Absent"
"Absent"
"NPC"
"Mixed"
"Mixed"
"NPC"
"NPC"
"NPC"
```

## The First Dataset
I finished the script from earlier to create the json file:
```python
import csv
import wave
import contextlib
import pandas as pd
from shutil import copy
from typing import List
from json import dumps,load
from multiprocessing import Pool
AUDIO_DIR = './AudioClip/'
AUDIO_DIR_NEW = './AudioClip_Narrator/'
with open('./actors_by_narrator.json') as f:
    df_actor = pd.json_normalize(load(f))

# https://stackoverflow.com/a/7833963
def duration(fname: str) -> float:
    with contextlib.closing(wave.open(fname,'r')) as f:
        frames = f.getnframes()
        rate = f.getframerate()
        return frames / float(rate)
# 2.2565208333333335 == duration('./AudioClip/ANTI OBJECT TASK FORCE_TITLE.wav'))

def csvToJson(fname: str, text: str) -> str:
    t = duration(AUDIO_DIR+fname)
    return dumps({'audio_filepath': fname, 'text': text, 'duration': t})
# ['fname', 'acticyID', 'alternativeIdx', 'text', 'actorID', 'actorName']
def consider(line: List[str]):
    fname, text, actorID = line[0], line[3], line[4]
    actor = df_actor.loc[int(actorID)-1]
    assert actor.actorId == int(actorID)
    if actor.narratorType == 'Narrator':
        copy(AUDIO_DIR+fname, AUDIO_DIR_NEW+fname) # very slow operation.
        return csvToJson(fname, text)

CORES = 8
with open('./AudioClipMetadata.csv') as csvfile, Pool(CORES) as pool, open('./AudioClips_Narrator.json', 'w') as json_file:
    reader = csv.reader(csvfile)
    next(reader) # ignore header
    for json_line in pool.imap(consider, reader, 1<<8):
        if json_line: json_file.write(json_line+'\n') # this isn't pooled, but this is ok since consider() is much slower.
```
Looks good:
{{< figure src="Pasted%20image%2020220819171057.png" >}}
But we're supposed to split this into test/train data. Let's copy a stackoverflow solution again:
```python
# https://stackoverflow.com/a/48347284
import random

def split_file(file,out1,out2,percentage=0.75,isShuffle=True,seed=123):
    """Splits a file in 2 given the `percentage` to go in the large file."""
    random.seed(seed)
    with open(file, 'r',encoding="utf-8") as fin, \
         open(out1, 'w') as foutBig, \
         open(out2, 'w') as foutSmall:

        nLines = sum(1 for line in fin) # if didn't count you could only approximate the percentage
        fin.seek(0)
        nTrain = int(nLines*percentage)
        nValid = nLines - nTrain

        i = 0
        for line in fin:
            r = random.random() if isShuffle else 0 # so that always evaluated to true when not isShuffle
            if (i < nTrain and r < percentage):# or (nLines - i > nValid): # <-- error in the SO solution.
                foutBig.write(line)
                i += 1
            else:
                foutSmall.write(line)
split_file('./AudioClips_Narrator.json', './AudioClips_Narrator_train.json', './AudioClips_Narrator_val.json')
```
I can verify that this doesn't accidentally delete any of the lines:
```bash
$ cat AudioClips_Narrator_* | sort | diff - <(sort AudioClips_Narrator.json) -s
Files - and /dev/fd/63 are identical
```
And now we just need to pack the data:
```sh
$ # this step takes a few minutes
$ tar zcf disco_dataset_NARRATOR.tar.gz AudioClip_Narrator/ AudioClips_Narrator_*
$ du disco_dataset_NARRATOR.tar.gz -h # check filesize
7.1G    disco_dataset_NARRATOR.tar.gz
```
7GB is big, but not too big. Google Drive provides 15GB of space for free, so I'm good to go for [Colab training](/todo).

