---
title: "DeepSeek Core Readings 0 - Coder"
date: 2024-06-30T00:00:00+08:00
author: "152334H"

lastmod: 2024-06-30T00:00:01+08:00
featuredImagePreview: ""
featuredImage: ""

draft: false
subtitle: "Part 0/? of a <a href=/deepseek-core-readings>series</a>"
description: ""
tags: [ machine learning, DeepSeek, llm, series, codegen ]
categories: ["tech"]

---

[Paper](https://arxiv.org/pdf/2401.14196) summary: 1.3B to 33B LLMs on 1/2T code tokens (87 langs) w/ FiM and 16K seqlen. Strong effort in constructing pretraining data from Github from scratch, with repository-level samples. Evals beat OSS code models solidly + GPT-3.5 a bit; Coder-7B > CodeLlama-33B often.

They don't spend much effort on Instruction tuning. They commit a continued pretrain of DeepSeek LLM -> Coder: I believe it underperforms; they don't.

<!--more-->

## Dataset
2T tokens: 87% source code, 10%/3% code-related natural English/Chinese -- English from github markdown / StackExchange, Chinese from selected articles.

To scrape github, they:
1. crawl all repositories created before Feb 2023, keeping only top87 langs. Inspired by StarCoder, they apply filters against
    * files with average length > 100chars or maxlen > 1000chars
    * files with <25% alphabetic chars
    * files with `<?xml version=` in first 100chars, except files tagged as XSLT
    * HTML files that have <20% visible text, or that are more "than 100 characters" (???)
    * JSON/YAML files that aren't in length $[50,5000]$.  
2. They attempt to **analyse dependencies between files** within the same repository,
    * by using a basic regexep to look for import keywords,
    * they create a **potentially circular** dependency graph between different files, 
    * use a modified toposort that selects nodes with the least in-degrees rather than none,
    * to create a linear sorted topology of files that are then concatentated (with added comment about filepath) to form a singlular training example.
3. They do **repo-level deduplication**, i.e. they compare concatentated repo examples for near-duplicates and prune repos when appropriate.
4. They use a compiler & quality model & heuristics to filter out garbage. This is supposed to get rid of code with syntax errors / poor readability/modularity.
5. They use an n-gram filter to get rid of test data from the train set. Specifically they look for 10-grams identical to any string in the test data, and also check for identical match on test examples that are shorter than 10-grams.

{{< figure src="Pasted%20image%2020240624022622.png" caption="" >}}

## Training

### Objectives
By default, models are assumed to be trained with basic CausalLM.

Then, they consider applying the [FIM](https://openai.com/index/efficient-training-of-language-models-to-fill-in-the-middle/) objective. DeepSeek considered:
* using T5-like masked-span prediction ("MSP") to infill 50% of the time
* using varying (0/50/100%) levels of FIM, using the traditional Prefix-Suffix-Middle (PSM) format.
  `<|fim_start|>{pre}<|fim_hole|>{suf}<|fim_end|>{mid}<|eos_token|>`

On 1.3B experiments, they observe that FIM 50% generally does better than MSP 50% on both infilling && code completion benchmarks.

They mention possibly using Suffix-Prefix-Middle (SPM) at the start of Section 3, but it is not clear to me whether they actually used it for their models or not.

### Arch
They use a 32k+ BPE vocab. This might sound like they just copied the LLaMA tokenizer, but their BOS/EOS token IDs are different (32013/32014), and they pad their embedding to 32256.

They do 1.3B/6.7B/33B. Only 33B is GQA ($R_\text{kv}=1/8$).

* 1.3B: `D=2048 R_ffn=2.6875 L=24 H=16 BS=1024 LR=5.3e-4`
* 6.7B: `D=4096 R_ffn=2.6875 L=32 H=32 BS=2304 LR=4.2e-4`
* 33B: `D=7168 R_ffn=19/28 L=64 H=56 BS=3840 LR=3.5e-4`

Optim/LR follows [Deepseek LLM](deepseek-1). RoPE $\theta=100000$ and 4x linear scaling, with 1k steps of 16k seqlen training. 64k extrapolation not reliable here.

I find their cluster description odd:

> ...utilize clusters outfitted with NVIDIA A100 and H800 GPUs. In the A100 cluster, each node is configured with 8 GPUs, interconnected in pairs using NVLink bridges. The H800 cluster is similarly arranged, with each node containing 8 GPUs. These GPUs are interconnected using a combination of NVLink and NVSwitch technologies, ensuring efficient data transfer within nodes. To facilitate seamless communication between nodes in both A100 and H800 clusters, we employ InfiniBand interconnects, known for their high throughput and low latency.

I don't get "interconnected in pairs." An SXM A100 node should have 8 GPUs connected all-to-all over an NVSwitch. 

![](https://docs.nvidia.com/dgx/dgxa100-user-guide/_images/dgxa100-system-topology.png)

Direct pairing should only apply for PCIe A100s. It is technically possible that they had NVL bridges across PCIe pairs, and used some CX-6 PCIe connectors, and had a smart parallelism strategy to reduce cross-pair comms maximally. But my bet is "typing/translation error".

Anyway,

They do a _lot less_ for post-training alignment here than they do for Deepseek LLM. They have only a single small section for SFT, where they use 100 step warmup cosine over 2B tokens on 1e-5 lr with 4M batch size. Their dataset is unknown alpaca-like with `<|EOT|>` at chat turn ends.

## Evaluation
They compare against **CodeGeeX2**, **StarCoder**, **CodeLlama**, **code-cushman-001**, and **GPT-3.5/4** (of course). God these names bring back memories.

### Code generation
On **HumanEval/MBPP** (and, note: both of these are normally Python only), 1.3B-instruct trounces all non-GPT models, and 33B beats 3.5 Turbo by about 3%.

Deepseek kindly makes the effort to use the [MultiPL-E](https://arxiv.org/pdf/2208.08227) strategy to convert the Python problems in HumanEval to C++/Java/PHP/TS/JS/C#/Bash, and finds similar results for the rest of the languages.

Because HumanEval/MBPP is too simple (basically no libraries), they also test with DS-1000. They do not compare with GPT3.5/4 here, so deepseek-coder wins by default.

Like Deepseek-LLM, they use **LeetCode contests** as a benchmark, where 33B achieves a Pass@1 of 27.8%, better than 3.5 again. They notice that their model improves on Medium/Hard problems with CoT, but worsens slightly on Easy problems. They also notice evidence of data contamination, as their model (and GPT-4) performs better on problems from July/August.

### Others

On SantaCoder's **Single-Line Infilling** benchmark, Codellama-13B-base beats Deepseek-33B-base (!) for Python (but not for java/javascript). Despite bolding this information in the table, they do not describe this phenomeon at all in the prose, stating:

> Despite being the smallest model with a capacity of 1.3 billion parameters, DeepSeek-Coder outperforms its larger counterparts, StarCoder and CodeLlama, in these benchmarks.

This is a bit weird. Usually Deepseek is more dignified than this.

Anyway, on **CrossCodeEval** -- [a benchmark](https://crosscodeeval.github.io/) where every code completion problem requires context from extra files, that was also created with data _after_ DeepSeek's pretrianing cutoff -- they once again note that their model performs good compared to non-OpenAI models (with the Retrieval strategy)

Finally, Deepseek groups GSM8k/MATH/GSM-Hard/SVAMP/TabMWP/ASDiv/MAWPS under the general category of "**Program-based Math Reasoning**" benchmarks (and yeah they win against `$baseline`). For these benchmarks,

> "the model is prompted to alternately describe a solution step in natural language and then execute that step with code".

I'm not sure what this means. Do they do step-by-step reasoning? Do they _actually_ execute the code, ala Code Interpreter, or just tell the model to hallucinate an execution? I'd guess the latter, since code environments aren't that simple to setup.

## Continued Pre-training
Deepseek attempts training DeekSeek-LLM-7B-Base on 2 Trillion more code tokens to create **DeepSeek-Coder-v1.5 7B**. Their training process is completely different from the prior DeepSeek models described -- they purely use CausalLM, and they use different data sources: 

{{< figure src="Pasted%20image%2020240629215736.png" caption="" >}}

They want to compare pure pretrained Coder vs continual pretrained Coder-v1.5, and test HumanEval/MBPP + GSM8K/MATH + standard GPT4All bench mix. They find that...

> the DeepSeek-Coder-Base-v1.5 model, despite a slight decrease in coding performance,
shows marked improvements across most tasks when compared to the DeepSeek-Coder-Base
model.

Yes, you read that right. Despite being worse at coding, they state that DeepSeek-Coder-v1.5 is better. Because it performs better than Coder v1 && LLM v1 at NLP / Math benchmarks. After having 2T more tokens than both.

...

## Conclusion
Other non-openai code models at the time sucked compared to DeepSeek-Coder on the tested regime (basic problems, library usage, leetcode, infilling, small cross-context, math reasoning), and especially suck to their basic instruct FT.

They use motivated reasoning to conclude that continued pretraining on general LLMs -> code LLMs is good.

Their appendix contains example usages, benchmark curves over pretraining tokens (their 1.3B was only trained to 1T!)
