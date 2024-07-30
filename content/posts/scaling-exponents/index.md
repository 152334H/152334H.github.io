---
title: "Calculating the Cost of a Google Deepmind Paper"
date: 2024-07-30T00:00:00+08:00
author: "152334H"

lastmod: 2024-07-30T00:00:01+08:00
featuredImagePreview: ""
featuredImage: ""

draft: false
subtitle: "How to burn US$10,000,000 on an arXiv preprint"
tags: [ machine learning, paper, llm, series ]
categories: ["tech"]

---
Recently, GDM released a great paper titled, [*Scaling Exponents Across Parameterizations and Optimizers*](https://arxiv.org/pdf/2407.05872), in which they conduct over 10,000 LLM training runs to obtain optimal hyperparameters under different regimes.

After reading it (it was great), I wanted to test my understanding of the paper by tallying up all experiments conducted within, calculating **the total compute cost it would take to replicate the paper**.

<!--more-->

## Headline result

| Subset                | Sources of uncertainty  | FLOPs   | Costs @ $3/H100/hr |
| --------------------- | ----------------------- | ------- | ------------------ |
| Alignment             | N/A                     | 3.7e20  | $888               |
| LR variants (+default)| LR-sweeps, bayes search | 7.99e23 | $1.90M             |
| LR variants (+optimal)| LR-sweeps               | 1.35e24 | $3.22M             |
| Epslion (Heatmaps)    | LR-sweeps, $D$          | 1.34e24 | $3.19M             |
| Epslion (Full Sweeps) | LR-sweeps               | 7.99e23 | $1.90M             |
| Weight Decay          | LR-sweeps               | 1.33e23 | $317K              |
| Adafactor vs Adam+PS  | LR-sweeps, $D$          | 7.92e22 | $188.5K            |
| Compute Optimals      | LR-sweeps, $D$          | 7.52e23 | $1.79M             |
|||||
| **Total**             | too much                |**5.42e24**| **$12.9M**        |

**Any corrections on the numbers here will be appreciated.**

Although I have made significant efforts to vet these claims, _if I have made significant mistakes in mathematics, these results could be off by magnitudes._

{{< admonition type=question title="Sidenote: What's an H100 worth?" open=false >}}

Although it's never stated, all experiments in the paper were almost certainly conducted with TPUs (because it's from Google Deepmind). Furthermore, as there is no mention of int8 usage in their paper, it is most likely that all experiments were conducted with bfloat16 compute precision, per the [nanodo default](https://github.com/google-deepmind/nanodo/blob/10aefdeed40a63293daf112b91a5538cd24fa3a4/nanodo/configs/default.py#L41).

However, as a GPU user, I prefer to calculate compute in terms of H100 hours. Some basic facts:
* The H100-SXM is [reported](https://resources.nvidia.com/en-us-tensor-core/nvidia-tensor-core-gpu-datasheet) as having 989.40TFLOP/s of 16-bit tensor core operations.
	* Also, 66.9TFLOP/s fp32 non-tensor, but I won't consider non-tensor operations (such as softmax or hadamard products) in my analysis.
* Recent pytorch [blogs](https://pytorch.org/blog/maximizing-training/) and [torchtitan](https://github.com/pytorch/torchtitan/pull/165) both report single-node FSDP'd bf16 H100 MFU for reasonably mid sized models at (optimistically) 40%.
	* the smaller models ($D<1024$) in the paper are unlikely to have MFU that high.
	* Although this is not hard to push higher with some manual tuning, the time spent tuning performance & engineering required to heuristically adjust for efficiency depending on setting is unlikely to be worth it.
* The cost of a H100 node (at the time of writing) is $3.5/hr/gpu on [lambdalabs](https://cloud.lambdalabs.com/instances), $2.85/hr/gpu from [sfcompute](https://sfcompute.com/), and ballpark $2/hr/gpu if you get a [long term bulk contract](https://gpulist.ai/).

If we pessimistically estimate the true average tensor FLOP/s provided by a H100 GPU on an average run as 3.5e14 (aka slightly above 35% MFU), and the cost of a H100 GPU as $3/hr, we get:

```python
def cost_of_run(flops: float, pergpu_flops=3.5e14):
  gpu_hours = flops / 3600 / pergpu_flops
  rental_cost = 3 * gpu_hours
  single_node_duration = gpu_hours / 8
  return rental_cost, single_node_duration
```

**These numbers are fungible** and you can choose to mentally halve (or double) them if you find it appropriate.

{{< /admonition >}}

---
## A summary of all experiments tried

{{< figure src="Pasted%20image%2020240722025146.png" caption="" >}}

There are a few different types of experiments done in the paper:

{{< admonition type=info title="Experiment types" open=true >}}
* **Alignment experiments**, which use a single global close-to-optimal LR, while varying
	* $D \in \{1024, 2048, 4096\}$
	* 4x paramterizations
	* 3x optimizers (Adam, SGD+momentum, Adafactor)
* **Learning rate** experiments, which vary:
	* 3x optimizers (Adam, SGD+momentum, Adam+PS)
	* 4x paramterizations
	* **14x model widths** $D \in [128, 16384]$. but this is really best described as scaling numheads $H \in \{1,2,4,6,8,12,16,20,24,32,48,64,96,128\}$
	* Global LR vs Per-layer Beta LR vs Per-layer $\beta$ Gamma LR + Per-layer $\beta\ \gamma$ No align LR
		* The $\gamma$ experiements are particularly complex to calculate, see [point 3](#Problems)
	* LR by an **indeterminate range** -- they sweep in intervals of $2^{0.25}\text{ or }2^{0.5}$ and terminate rightwards when
	  1.  the LR leads to NaNs OR
	  2. the eval loss for a given LR $\mathcal{L}^\eta \gt 1.2\times \text{argmin}_\eta(\mathcal{L^\eta})$

      i.e. the first (larger than optimal) LR to show either of those conditions is **not plotted**, and the LR $\sqrt{2}$ or $\surd\surd2$ is.

      ...or at least, that is what the paper says is supposed to be the case. [I explain my contentions later](#Problems).
* **Adam Epslion** experiments, which vary
	* over 4x parameterizations,
		* _at least_ $D\in \{3072, 4096, 6144, 8192, 12288, 16384\}$ over Adam, where
			* _at least_ 6x eps is tried
			* _at least_ constant vs per-layer $\epsilon$ is compared.
			* _at least_ 13x LR is tried. Appendix F: "[learning rate sweep at each model dim for each value of epsilon or base epsilon](https://arxiv.org/pdf/2407.05872#page=43)"
		* according to Appendix J/K, over **all 14 model dims**,
			* For Adam, 4x (base eps, small const, good per-layer, atan2)
				* technically, we double-count base EPS from the LR experiments, but we also neglect the extra no-align per-layer eps experiments, so this cancels out
			* For Adam+PS, 2x (base eps, good per-layer)
				* the double-neglect accounting argument applies here too
* extra **weight decay** experiments
	* static: adam, per-layer, full alignment, decoupled 1e-4
	* 4x parameterizations
	* LR experiment-like sweep across **all 14 model widths**
* extra **adafactor** experiments
	* 2x optim (Adafactor vs adam+ps)
	* 2x setting (globalLR+default vs perlayer+optimal)
	* 4x parameterizations
	* LR experiment-like sweep across **only 11x** model widths up to $H=48$ due to FSDP.
		* actually [implemented as 12x](https://arxiv.org/pdf/2407.05872#page=52) but [final results are 11x](https://arxiv.org/pdf/2407.05872#page=47) and I follow the latter.
* extra fixed step vs **compute optimal**
	* the 50k fixed step experiments are **not the same** as any of the above; they use "default constant learning rate multipliers" and have different power laws.
	* 3x optim (SGD+moment, adam, adafactor)
	* 4x parameterizations
	* LR experiment-like sweep across model width && LR.
		* width **only goes up to 11x**, last 3 are missing on Compute Optimal.
	* compute-optimal experiments use 20x tokens of non-embedding P as a heuristic.
{{< /admonition >}}

However, there are many problems with the experimental summary as given above.

{{< admonition type=warning title="Problems" open=false >}}

1. It is **not clear whether they re-executed** the per-layerLR experiments for the two edge cases where per-layer constants lead to identical behavior to globalLR (where $c_1 = c_l = c_{L+1}$):
	* muP + SGD + full alignment, or
	* Adafactor + any parameterization + no alignment

   My expectation is that **their experiments were repeated**, because if you look at Table E1, you'll see that the muP+SGD+full columns actually have a single diverging value (presumably caused by precision differences):

   {{< figure src="VXjZe27k.jpeg" caption="" >}}

   **However**, I was also given (private) notice that **in some cases, the experiments with theoretically equivalent settings were merely executed once**, with the eval losses copied twice. This makes the true extent of compute unknowable from the paper.
2. The LR experiments have indeterminate bounds, so I can't directly figure out how many experiments were executed.
   
   You can't "just read the graphs" to figure out what the range of LRs used are either; they cut off the y/x axis:
   {{< figure src="Pasted%20image%2020240722134014.png" caption="" >}}
   Frankly, it doesn't even look like the steps here are guaranteed to be split in intervals of $2^{0.25}\text{ or }2^{0.5}$.
   {{< figure src="Pasted%20image%2020240730045534.png" caption="" >}}
   After further inspection, it looks an awful lot like the runs have **arbitrary LR ranges even for the same $D$, optim, parameterization, and alignment**. Or I just don't understand the selection process (what are the unshaded shapes?).
3. In [C.4](https://arxiv.org/pdf/2407.05872#page=34)., they state:
   > When tuning the per-layer constant multiplicative factors defined in Section 4.2, we use [vizier](https://github.com/google/vizier) to perform 3D hparam search for $(γ_1, γ_h, γ_{L+1})$ at $b = 1024$. Recall that we define the learning rate in layer $l$ as $η_l = β_n·γ_l·\frac{n}{b}^{−cl}$ and sweep one dimension at all model sizes to determine $β_n$, so these values of $(γ_1, γ_h, γ_{L+1})$ define two ratios where any common factor can be absorbed by $β_n$.
   
   To be clear, that last segment means: "you can divide $(γ_1, γ_h, γ_{L+1})$ by any of the 3 values to obtain some $(\gamma_x, \gamma_y, 1)$ tuple, the sweep will bring $\beta_n$ back to the correct value". And so they say:
   > For each optimizer × parameterization, we run 800 trials with at most 100 trials in parallel with a range set to $[1\text{e−}2, 1e2]$ for each constant. If the optimal value for any of the constants is at or near the edge of the range after this first search, we extend the range of the sweep for that constant to 0.01 and 100x the optimal value found in the original sweep and repeat the same tuning procedure.
   
   Upside: this gives 800 experiments as a lower bound for the $\gamma$ experiments.
   Downside: We otherwise have no plotted information about the 3D experiments that were conducted. The actual plotted graphs just show final eval loss against base LR, under the assumption that the $b=1024$ base line on the Optimal Constants graphs actually hide the extra work done to sweep $\gamma$ values.
4. It is deeply unclear to me what is actually implemented for the fixed-step vs compute optimal runs. If we look at the [50k steps graph](https://arxiv.org/pdf/2407.05872#page=51):
   {{< figure src="Pasted%20image%2020240730044030.png" caption="" >}}
   It looks _extremely similar_, but **not identical** to the [original Adam+GlobalLR+default graphs](https://arxiv.org/pdf/2407.05872#page=56):
   {{< figure src="Pasted%20image%2020240730044047.png" caption="" >}}
   I have no idea what the differences are supposed to be here. However, in the interest of sticking with the paper's behaviour, I attempt to include the compute used for these psuedo-repeated experiments.

{{< /admonition >}}

For each of these issues, I do my best to pick an approximation that makes sense to me in the later sections. 

---
## Transformer information

In Appendix C, the model is described as:
* decoder-only
* no bias on weights (including layernorm, which only has learnable scale)
* LPE, pre-LN, GeLU, no tied emb
* T5 Sentencepiece 32k + 1BOS + 100extra, i.e. $V=32101$. This is never stated to be padded.
* "Training inputs are sequence-packed, while evaluation inputs are padded"
* $\text{batch size}=256$, $l_\text{seq}=512$, $L=8$, $D_\text{head}=128$
* $D_\text{head}*H = D$, $R_\text{ffn} = 4$.

with some extra details for later:
* no dropout
* mostly FSDP
* $P \approx L12D^2 + 2VD$ (this excludes the layernorm params ($2LD$) and the LPE ($Vl_\text{seq}$))
* "The compute optimal experiments include models up to $H = 32$ or $H = 48$, and the fixed (50,000) step experiments include models up to $H = 128$."

### FLOPs per token

To start, we want to find $M$, the number of FLOPs required per token for a training run.

{{< admonition type=info title="Basic transformer math" open=false >}}
As a reminder for any noam-like transformer, the tensor FLOPs required per token $M$ is approx: 

$$V - \text{vocab size}$$
$$D - \text{hidden dim}$$
$$L - \text{xf layer count}$$

$$R_{\text{ffn}} - \text{[ffn dim : outer dim] ratio, assuming no GLU}$$
$$R_{kv} - \text{[num k or v heads : num att heads] ratio}$$
$$l_{seq} - \text{assumed average sequence length}$$

$$M = 12D^2L(1 + R_{kv} + R_{\text{ffn}}) + 6DL\cdot l_{seq} + 6DV$$

In particular, $6DL\cdot l_\text{seq}$ assumes a causal mask halves the computation required (I assume flash-attn does this)

{{< /admonition >}}

The paper does not describe the usage of any GQA/MQA, so I assume $R_\text{kv} = 1$. This gives us

> $M=72D^2L + 6DLl_\text{seq} + 6DV = 6D(12DL + Ll_\text{seq} + V) = 6D(L(12D+l_\text{seq}) + V)$

We have additional constants of $L=8$, $l_\text{seq} = 512$, and $V=32101$, so we write:

```python
def M(d: int, L=8, l_seq=512, V=32101) -> int:
    return 6*d * (L*(12*d + l_seq) + V)
TPE = 50000 * 256 * 512
```

For all experiments _except the compute-optimal series in Appendix I_, we also have a hardcoded number of $steps=50000$ and global $BS=256$, making the total number of tokens seen per experiment $TPE=6.5536\text{e}9$ by default.

---

## Subproblem: Alignment experiments
I *assume* the alignment experiments got their optimal LRs from the later experiments, and didn't do their own sweeps, so that would make the cost simply,
$$
\sum_{d\in \{1024,2048,4096\}} 4\times\text{tokens per experiment}\times M(d)
$$
```python
def alignment() -> int:
    return 4 * TPE * sum(M(d) for d in [1024,2048,4096])
# >>> f'{alignment():.3E}'
# '3.733E+20'
# >>> cost_of_run(alignment())[0]
# 888.81395400704
```

These experiments would take <US$1k to execute.
## Subproblem: Table E1 experiments

[Table E1](https://arxiv.org/pdf/2407.05872#page=40) has a neat collection of many of the runs done for obtaining the _best_ eval losses under any given parameterization/optimizer/setting (some combination of global vs per-layer vs $\gamma$-optimal vs $\epsilon$-optimal).

This is an easier subproblem to tackle than the general issue of _all LR sweeps_, as the requirements are better known -- though still not entirely determined, per the repetition ambiguity mentioned earlier. For that issue, I assume that all experiments were conducted, with no copied results, making the estimate here an upper bound.

We have the following schedule:
* $D\in \{3072, 4096, 6144, 8192, 12288, 16384\}$
* 4x parameterizations
* 3x optimizers, where
	* SGD only receives 5 experimental settings
	* Adam & Adam+PS receives 7

$$
\sum_{d\in \{3072,4096,6144,8192,12288,16384\}} 4\times(5+7*2)\times\text{tokens per experiment}\times M(d)
$$

```python
H = [1,2,4,6,8,12,16,20,24,32,48,64,96,128]
D = [h * 128 for h in H]
def table_e1() -> int:
  sets_x_optims = 5 + 7 + 7
  return 4 * sets_x_optims * TPE * sum(M(d) for d in D[-6:])
# >>> f'{table_e1():.3E}';cost_of_run(table_e1())
# '1.634E+23'
# (388955.9991064986, 16206.499962770775)
```

These would've taken slightly below $400k in H100 compute to execute. Reasonably speaking, this is within the bounds of SWE life savings / big academic budgets / TPU Research Cloud upper-class. Technically replicable, albeit not cheap.

But the bulk of the compute used in the paper comes from the LR sweeps, so we have to start working on that.

## Estimating LR sweep damage

So, here's a a graph:
{{< image src="Pasted%20image%2020240730050758.png" caption="" >}}
Here's another graph:
{{< image src="Pasted%20image%2020240730050857.png" caption="" >}}
And here's a third one:
{{< image src="Pasted%20image%2020240730050917.png" caption="" >}}
Guess what?

1. There isn't a constant num. of LRs sweeped for a given $D$, or optim/parameterization/setting.
	* Especially notable: number of runs seems inversely correlated with $D$; there are almost always less runs for the highest dim than the lowest.
2. Neither is there an observable cutoff for when the runs stop -- runs will spike up to 2x the optimal no problem.
3. You can't get the *exact* correct number of runs by graph-reading; in many cases the points are out-of-bounds.

The consistencies I _do_ spot are that:
* there is typically a "starting LR" (smallest base) for any given line.
* the hollowed points are typically to the right -- but sometimes left -- of the optimal point.

so **I _think_ the mechanism worked this way**:
1. start a sweep with a starting LR and some expected jumpsizes of $\sqrt{2}$ or $\sqrt{\surd 2}$.
2. terminate it by the 20% / NaN heuristic.
3. if the graph looks weird (optimal point somewhere odd), rerun to fill many $2^{0.25}$ intervals around the current optimal. These result in the plotted hollow points

I have no means of confirming this as the experimental procedure, as the authors of the paper stopped replying to me.
### An arbitrary decision

Due to my desire to finish this blog post in a reasonable amount of time, I made the unprincipled decision of approximating the number of experiments-per-line in any given Eval Loss vs Base Learning Rate graph as **15**. 

Why 15? By eyeballing, the range of runs-per-line for the highest $D=16384$ hovers around 10~15. Although the lines with smaller D tend to have far more points on average, the amount of compute spent per run scales by $O(D^2)$, so I think this is fair enough.

Feel free to suggest a more principled approach if you have one.

---

## Main problem: Epslion

Much of the compute used up by the paper comes from [Section 4.3](https://arxiv.org/pdf/2407.05872#page=15), the Adam epslion experiments.

### Optimal eps runs

Now that we have an estimate of LRs-per-line as 15, we can estimate the compute spent on the actual [Adam epslion varying graphs](https://arxiv.org/pdf/2407.05872#page=44):

$$
\sum_{d} 4*(2+4) \times \text{points per line}\times\text{tokens per experiment}\times M(d)
$$

```python
PpL = 15 # unprincipled estimate
def eps_variants() -> int:
  return 4 * 6 * PpL * TPE * sum(M(d) for d in D)
'''
>>> f'{eps_variants():.3E}';cost_of_run(eps_variants())
'7.988E+23'
(1902022.3291813303, 79250.93038255542)
'''
```

Simple enough, right? Ignoring the ~$2M bill.

### Epslion Heatmaps

There are two ways you could approach the expected sweep range for this problem:
1. assume the LR experiment sweep code was reused. All 14x $D$, LR swept by arcane unknown ruleset.
2. Limit to the graphs. Only the last 6 values of $D$ were shown -- assume only those were used. Plus, if we look at [Figure 6](https://arxiv.org/pdf/2407.05872#page=16): 
   {{< figure src="Pasted%20image%2020240730044906.png" caption="" >}}
   Notice that the range of evaluated learning rates actually seems constant here, unlike in the normal Eval Loss vs Base LR plots. 

I'm picking the latter because it's simpler. Would be happy to be shown evidence that this is wrong.

$$ \sum_{d\in \{3072,4096,6144,8192,12288,16384\}} 4\cdot 2\cdot 6\cdot 13\times \text{tokens per experiment}\times M(d) $$

```python
def eps_heatmaps() -> int:
  # eps-type * eps-val * parameterizations * LR range * ...
  return 2 * 6 * 4 * 13 * TPE * sum(M(d) for d in D[-6:])
'''
>>> f'{eps_heatmaps():.3E}';cost_of_run(eps_heatmaps())
'1.341E+24'
(3193533.466348094, 133063.89443117057)
'''
```

#### These squares are worth US$3.2 Million

{{< figure src="Pasted%20image%2020240730060102.png" caption="" >}}

To be clear, this is supposed to be an **underestimate** of the budget required, because we model the average number of unique LRs used per heatmap square as a constant $13$ instead of the (typically higher) value used in variable LR sweeps.


## Main problem: LR Sweep Strategies

The other meat of the paper is in Section 4.2, the $\text{optimizer}\times\text{parameterization}\times D\times\text{LR setting}\times\text{alignment}\times\text{LR Sweeps}$ experiments.
### $\beta$-only experiments

"$\beta$" refers to the empirically obtained base LR constant under the equation $\eta_l = \beta_n\cdot\frac{n}{b}^{-c_l}$, also known as the `+default` experiments.

The paper sweeps this for 3x optimizers, 4x parameterizations, 14x widths, global vs per-layer $c_l$, and of course unknown LR sweep counts.

$$ \sum_{d} 3\*4\*2 \times \text{points per line}\times\text{tokens per experiment}\times M(d) $$

```python
def beta_only() -> int:
  return 3*4*2*PpL * TPE * sum(M(d) for d in D)
# 7.988E+23 (1902022.3291813303, 79250.93038255542)
```

Incidentally, this has an identical estimated cost to the [epslion variants](#optimal-eps-runs).

### $\gamma$ experiments

So, two issues.
1. These experiments are "like" the $\beta$-only experiments, but with 3x cases (GlobalLR, Perlayer-fullalign, Perlayer-nolign) instead of 2x (GlobalLR, Perlayer-fullalign).
	$$ \sum_{d} 3\*4\*3 \times \text{points per line}\times\text{tokens per experiment}\times M(d) $$
2. Specifically for $d=1024=b$, we have **at least** 800 extra runs, due to the 3D hparam search for $(\gamma_1, \gamma_h, \gamma_{L+1})$.
	$$ 3\*4\*3\*800 \times\text{tokens per experiment}\times M(1024) $$

We can combine those two as,
$$ 36\times\text{tokens per experiment}(800\*M(1024) + \text{points per line}\sum_{d}\times M(d)) $$

```python
def gamma_expts() -> int:
  return 36*TPE * (800*M(1024) + PpL*sum(M(d) for d in D))
# gamma_expts 	 1.354E+24 (3224397.534237257, 134349.8972598857)
```
This is, once again, exceedingly close to that of the Adam $\epslion$ heatmap experiments.

Sidenote: I may be understanding the per-layer aspect of the paper incorrectly; I expected the compute expenditure of this section to be larger.

## Extras
### Weight Decay
The WD experiments are simple enough. We repeat 4x parameterizations && do a single base-LR sweep on all $D$

$$
\sum_{d} 4*(2+4) \times \text{points per line}\times\text{tokens per experiment}\times M(d)
$$

```python
def weight_decay() -> int:
  return 4 * PpL * TPE * sum(M(d) for d in D)
'''
>>> f'{weight_decay():.3E}'; cost_of_run(weight_decay())
'1.331E+23'
(317003.7215302217, 13208.488397092571)
'''
```

Incredibly cheap, I could afford that in some years.

### Adafactor

As a reminder, I only count the first 11 $D$, even though the report actually has 12 in one graph.

$$
\sum_{d\in D[:11]} 2 * 2* 4\times \text{points per line}\times\text{tokens per experiment}\times M(d)
$$

```python
def adafactor() -> int:
  return 2*2*4*PpL*TPE*sum(M(d) for d in D[:11])
'''
>>> f'{adafactor():.3E}'; cost_of_run(adafactor())
'7.918E+22'
(188532.80765144504, 7855.533652143543)
'''
```

### Compute Optimal

The paper states that,
> The compute optimal experiments include models up to $H = 32$ or $H = 48$, and the fixed (50,000) step experiments include models up to $H = 128$.

If you read the graphs in [Appendix I](https://arxiv.org/pdf/2407.05872#page=50), this is slightly wrong, because
* 50k experiments go to $H=48$ on Adafactor, and $H=128$ otherwise
* all compute optimal experiments go up to $H=32$ only.

  Note that a 4B param run requires 80B tokens by chinchilla, and C4 is less than 200B tokens, so they couldn't have gone higher without changing the dataset.

This is honestly a bit complex, so let's forgo the latex and just describe it in python:

```python
def P(d: int, L=8, V=32101) -> int:
    return 2 * d * (6*L*d + V)

def compute_optimal():
  indices_50k = (14, 14, 12)
  return 4*PpL*sum([
    TPE * sum(sum( M(d) for d in D[:i] ) for i in indices_50k),
	20  * sum(P(d)*M(d) for d in D[:11]) *3,
  ])
# compute_optim 	 7.518E+23 (1790104.1799513847, 74587.67416464102)
```

## Code summary

Here is the full script to get the estimates I created:

```python
TPE = 50000 * 256 * 512
H = [1,2,4,6,8,12,16,20,24,32,48,64,96,128]
D = [h * 128 for h in H]
PpL = 15 # unprincipled estimate

def cost_of_run(flops: float, pergpu_flops=3.5e14):
  gpu_hours = flops / 3600 / pergpu_flops
  rental_cost = 3 * gpu_hours
  single_node_duration = gpu_hours / 8
  return rental_cost, single_node_duration

def M(d: int, L=8, l_seq=512, V=32101) -> int:
    return 6*d * (L*(12*d + l_seq) + V)

def P(d: int, L=8, V=32101) -> int:
    return 2 * d * (6*L*d + V)

def alignment() -> int:
    return 4 * TPE * sum(M(d) for d in [1024,2048,4096])

def table_e1() -> int:
  sets_x_optims = 5 + 7 + 7
  return 4 * sets_x_optims * TPE * sum(M(d) for d in D[-6:])

def eps_variants() -> int:
  return 4 * 6 * PpL * TPE * sum(M(d) for d in D)

def eps_heatmaps() -> int:
  return 2 * 6 * 4 * 13 * TPE * sum(M(d) for d in D[-6:])

def beta_only() -> int:
  return 3*4*2*PpL * TPE * sum(M(d) for d in D)

def gamma_expts() -> int:
  return 36*TPE * (800*M(1024) + PpL*sum(M(d) for d in D))

def weight_decay() -> int:
  return 4 * PpL * TPE * sum(M(d) for d in D)

def adafactor() -> int:
  return 2*2*4*PpL*TPE*sum(M(d) for d in D[:11])

def compute_optim():
  indices_50k = (14, 14, 12)
  return 4*PpL*sum([
    TPE * sum(sum( M(d) for d in D[:i] ) for i in indices_50k),
	20  * sum(P(d)*M(d) for d in D[:11]) *3,
  ])

total_flops, total_price, total_hours = 0,0,0
for f in (alignment,table_e1,eps_variants,eps_heatmaps,beta_only,gamma_expts,weight_decay,adafactor,compute_optim):
    flops = f()
    costs = cost_of_run(flops)
    print(f'{f.__name__:15}', f'{flops:.3E}',costs)
    total_flops += flops; total_price += costs[0]; total_hours += costs[1]

print(f'{total_flops=:.3E}')
print(f'rental price: US${total_price/1e6:.3}M')
print(f'h100 node months required: {total_hours/24/30}')
print()
print(f'(sanity check) {D=}')
print('(sanity check) model sizes:', [f'{P(d)/1e9:.3}B' for d in D])
print('(sanity check) M/6P:', [f'{100*M(d)/P(d)/6:.3}%' for d in D])

```

This gives the following:
```
alignment       3.733E+20 (888.81395400704, 37.033914750293334)
table_e1        1.634E+23 (388955.9991064986, 16206.499962770775)
eps_variants    7.988E+23 (1902022.3291813303, 79250.93038255542)
eps_heatmaps    1.341E+24 (3193533.466348094, 133063.89443117057)
beta_only       7.988E+23 (1902022.3291813303, 79250.93038255542)
gamma_expts     1.354E+24 (3224397.534237257, 134349.8972598857)
weight_decay    1.331E+23 (317003.7215302217, 13208.488397092571)
adafactor       7.918E+22 (188532.80765144504, 7855.533652143543)
compute_optim   7.518E+23 (1790104.1799513847, 74587.67416464102)
```
```
total_flops=5.421E+24
rental price: US$12.9M
h100 node months required: 746.9595590938408

(sanity check) D=[128, 256, 512, 768, 1024, 1536, 2048, 2560, 3072, 4096, 6144, 8192, 12288, 16384]
(sanity check) model sizes: ['0.00979B', '0.0227B', '0.058B', '0.106B', '0.166B', '0.325B', '0.534B', '0.794B', '1.1B', '1.87B', '4.02B', '6.97B', '15.3B', '26.8B']
(sanity check) M/6P: ['63.4%', '68.5%', '75.3%', '79.7%', '82.8%', '86.8%', '89.3%', '91.0%', '92.2%', '93.9%', '95.7%', '96.7%', '97.7%', '98.3%']
```

In the grand scheme of things, 5.42e24 is "not that big". After all, that's not even 15% of the [compute used for Llama 3](https://ai.meta.com/research/publications/the-llama-3-herd-of-models/); a [100k H100 cluster](https://www.semianalysis.com/p/100000-h100-clusters-power-network) could accomplish all of these experiments in just 2 days.

