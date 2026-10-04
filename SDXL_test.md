# Practical SDXL Testing on a 16 GB AMD GPU
## Batch Scaling, VRAM Behavior, VAE Tiling, Sampling Steps and Native High-Resolution Limits

## Overview

This case study started as a practical attempt to find reliable production settings for SDXL-family image generation in ComfyUI on a 16 GB AMD GPU.

Instead of benchmarking a single checkpoint once, the same general workflow was exercised across **three independent SDXL-family checkpoints**, all approximately **6.9–7.0 GB in file size**, using multiple batch sizes, resolutions, VAE decoding modes and sampling-step counts.

The goal was to answer practical questions:

- How far can batch size be pushed on 16 GB VRAM?
- How does VRAM usage scale with batch size and resolution?
- When does VAE decoding become more memory-intensive than sampling?
- When does tiled VAE decoding become useful?
- How much shared GPU memory appears under memory pressure?
- Does higher native resolution actually improve generation quality?
- At what point does model behavior become the limiting factor before hardware does?
- How useful are high sampling-step counts compared with finding a better seed?

The testing eventually revealed three different practical limits:

> **Sampler memory limit**  
> **VAE decode memory limit**  
> **Model native-resolution limit**

These limits are not reached at the same point.

---

## Test System

**GPU:** AMD Radeon RX 9060 XT, 16 GB VRAM  
**CPU:** AMD Ryzen 9 9900X  
**System RAM:** 32 GB  
**OS:** Windows  
**Frontend:** ComfyUI  
**Compute stack:** PyTorch / ROCm  
**Memory management:** DynamicVRAM with asynchronous weight offloading

The cooling configuration is not completely stock.

The GPU uses a manually adjusted fan curve and the case runs relatively aggressive airflow. CPU cooling is handled by an Arctic Liquid Freezer II 280, including additional airflow around the VRM area.

This matters when interpreting thermal results. The temperatures below describe this specific system and should not be treated as guaranteed values for stock fan curves or different case airflow.

---

## Models

Three independent SDXL-family checkpoints were tested.

All three:

- belonged to the same general SDXL model family
- were approximately **6.9–7.0 GB**
- were separate fine-tunes
- were tested through the same general ComfyUI workflow

Despite being similar in size and architecture, they did not behave identically.

Differences appeared in:

- VRAM consumption
- execution speed
- power behavior
- response to additional sampling steps
- anatomy and composition consistency
- high-resolution behavior

This helped avoid drawing conclusions from a single checkpoint.

---

## Test Methodology

Testing was performed incrementally.

The baseline resolution was:

> **800 × 1200**

At this resolution, batch size was progressively increased:

> **4 → 5 → 6 → 9 → 12**

Multiple runs were performed with both:

- standard/non-tiled VAE decoding
- tiled VAE decoding

The larger batch configurations were then repeated across different checkpoints.

Higher-resolution testing used:

> **1200 × 1800 — batch 2**

and:

> **1600 × 2400 — batch 1**

Most performance-oriented tests used approximately:

> **40–45 sampling steps**

Separate image-quality tests used fixed seeds across much wider ranges:

> **20 → 30 → 40 → 50 → 60 → 70 → 85 → 100 → 120–130**

where useful.

This was not a one-pass benchmark. Batch sizes, decoding modes, resolutions and checkpoints were tested through multiple runs before practical conclusions were drawn.

---

## Measurements

Two sources of data were used.

### ComfyUI console

The console provided:

- sampler execution time
- seconds per iteration
- total prompt execution time
- model-loading transitions
- VAE decode transitions

### HWiNFO logging

Hardware telemetry included:

- dedicated GPU memory
- shared GPU memory
- system RAM utilization
- GPU utilization
- total board power
- GPU temperature
- hotspot temperature
- GPU memory temperature
- memory junction temperature
- CPU temperature
- motherboard MOS / VRM temperature

The HWiNFO timeline was compared against ComfyUI execution timing so that sampling and VAE decoding could be evaluated separately.

Because HWiNFO is sampled externally, instantaneous peaks and exact transition points should be treated as approximate.

---

## Master Table — Representative Runs

The table below consolidates representative runs from the repeated test series. Peak memory values are HWiNFO observations and should be treated as sampled telemetry rather than exact instantaneous maxima.

| Model | Resolution | Batch | Steps | VAE | Peak dedicated VRAM | Peak shared GPU memory | Sampler time | Total time |
|---|---:|---:|---:|---|---:|---:|---:|---:|
| Model A | 800×1200 | 9 | 45 | Tiled 800/96 | ~13.65 GB | ~2.14 GB | ~109 s | 124.03 s |
| Model A | 800×1200 | 9 | 45 | Standard | ~13.93 GB | ~2.85 GB | ~109 s | 119.34 s |
| Model B | 800×1200 | 9 | 45 | Tiled 800/96 | ~13.18 GB | ~0.74 GB | ~109 s | 119.10 s |
| Model B | 800×1200 | 9 | 45 | Standard | ~13.32 GB | ~2.85 GB | ~105 s | 136.59 s |
| Model C | 800×1200 | 9 | 45 | Tiled 800/96 | ~13.14 GB | ~0.74 GB | ~109 s | 119.37 s |
| Model C | 800×1200 | 9 | 45 | Standard | ~13.03 GB | ~0.74 GB | ~108 s | 109.18 s |
| Model A | 800×1200 | 12 | 45 | Tiled 800/96 | ~13.72 GB | ~0.74 GB | ~140 s | 185.95 s |
| Model A | 800×1200 | 12 | 45 | Standard | ~14.04 GB | ~0.74 GB | ~145 s | 146.27 s |
| Model B | 800×1200 | 12 | 45 | Tiled 800/96 | ~14.24 GB | ~0.75 GB | ~146 s | 159.23 s |
| Model B | 800×1200 | 12 | 45 | Standard | ~14.37 GB | ~2.85 GB | ~146 s | 156.51 s |
| Model A | 1200×1800 | 2 | 40 | Tiled 800/96 | ~9.84 GB | ~0.74 GB | ~55 s | 84.58 s |
| Model A | 1200×1800 | 2 | 40 | Standard | ~15.31 GB | ~6.75 GB | ~57 s | 83.70 s |
| Model B | 1200×1800 | 2 | 40 | Tiled 800/96 | ~11.85 GB | ~0.74 GB | ~57 s | 62.71 s |
| Model B | 1200×1800 | 2 | 40 | Standard | ~14.91 GB | ~0.74 GB | ~58 s | 58.50 s |
| Model A | 1600×2400 | 1 | 40 | Tiled 800/96 | ~11.86 GB | ~0.74 GB | ~60 s | 64.49 s |
| Model A | 1600×2400 | 1 | 40 | Standard | ~14.20 GB | ~4.98 GB | ~60 s | 66.40 s |
| Model B | 1600×2400 | 1 | 40 | Tiled 800/96 | ~13.40 GB | ~0.74 GB | ~57 s | 66.70 s |
| Model B | 1600×2400 | 1 | 40 | Standard | ~12.36 GB | ~6.44 GB | ~57 s | 71.57 s |

![800×1200 batch scaling VRAM](charts/01_batch_scaling_vram.png)

The baseline chart shows that moving from batch 9 to batch 12 increased dedicated VRAM use, but the tested 16 GB card still retained usable headroom during sampling.

![1200×1800 memory comparison](charts/02_1200x1800_memory.png)

At 1200 × 1800, tiled decoding could dramatically reduce total memory pressure. The effect varied by checkpoint, which is why multiple independent models were included.

![1600×2400 memory comparison](charts/03_1600x2400_memory.png)

At 1600 × 2400, tiled decoding consistently kept shared-memory use near baseline in the representative runs, while standard decoding could spill several gigabytes into shared GPU/system memory.

---
## 800 × 1200 — Batch Scaling

800 × 1200 contains:

> **960,000 pixels**

This became the baseline production resolution.

Batch size was progressively increased to:

> **Batch 12**

and remained practical.

Representative batch-12 measurements from two of the heavier checkpoints were:

| Resolution | Batch | VAE | Sampler VRAM | Shared during sampler | Sampler time | Total time |
|---|---:|---|---:|---:|---:|---:|
| 800×1200 | 12 | Tiled | ~13.7 GB | ~0.74 GB | ~140 s | ~186 s |
| 800×1200 | 12 | Standard | ~14.0 GB | ~0.74 GB | ~145 s | ~146 s |
| 800×1200 | 12 | Standard | ~14.36 GB | ~0.74 GB | ~146 s | ~157 s |
| 800×1200 | 12 | Tiled | ~14.24 GB | ~0.75 GB | ~146 s | ~159 s |

With approximately 16.3 GB of usable physical VRAM, the heavier sampler runs still retained roughly:

> **~1.9–2.3 GB of VRAM headroom**

Shared GPU memory during sampling remained close to baseline, so batch 12 was not being sustained through heavy system-memory spill.

The practical seed-search preset that emerged was:

> **800 × 1200**  
> **Batch 12**  
> **40 steps**  
> **Standard VAE**

---

## Batch Scaling Summary

Representative runs across the batch-size progression looked approximately like this:

| Resolution | Batch | VAE | Total time | Approx. time per image |
|---|---:|---|---:|---:|
| 800×1200 | 4 | Standard | ~57 s | ~14.3 s |
| 800×1200 | 5 | Standard | ~76 s | ~15.2 s |
| 800×1200 | 5 | Tiled | ~66 s | ~13.2 s |
| 800×1200 | 6 | Standard | ~74 s | ~12.3 s |
| 800×1200 | 9 | Standard | ~109–137 s | ~12–15 s |
| 800×1200 | 12 | Standard | ~146–157 s | ~12–13 s |

Exact performance differed between checkpoints, but the broader result was consistent:

> **800 × 1200 at batch 12 was a usable operating point rather than an unstable edge case.**

---

## VAE Decode as a Separate Memory Problem

One of the clearest findings was that sampler memory and VAE decode memory must be evaluated independently.

A run could show perfectly healthy sampler behavior and then create substantially higher memory pressure during decode.

This became especially visible at higher resolutions.

Standard VAE decode sometimes caused large shared-memory spikes even though shared memory had stayed nearly flat during sampling.

Tiled VAE decode frequently avoided that behavior.

This also showed why dedicated VRAM alone can be misleading.

For example:

> **12 GB dedicated + 6 GB shared**

can represent a worse memory situation than:

> **14 GB dedicated + ~0.7 GB shared**

even though the first dedicated-VRAM number appears lower.

---

## 1200 × 1800 — Full-HD-Class Pixel Area

1200 × 1800 contains:

> **2,160,000 pixels**

For comparison:

> **1920 × 1080 = 2,073,600 pixels**

So 1200 × 1800 contains only about:

> **4.2% more pixels than Full HD**

The aspect ratio is different, but the raw pixel workload is in essentially the same class.

The main test configuration was:

> **1200 × 1800**  
> **Batch 2**  
> **40 steps**

---

## Image Quality at 1200 × 1800

This resolution produced one of the most useful visual findings in the testing.

For multiple checkpoints, the improvement over 800 × 1200 was not simply the result of having more pixels.

The generation itself often appeared better.

The model produced:

- more coherent structure
- cleaner faces
- stronger local detail
- more precise object construction
- better separation of materials and shapes
- stronger overall rendering

The difference felt less like:

> “the same image rendered at higher resolution”

and more like:

> “the model is constructing the image more successfully at this canvas size.”

That makes 1200 × 1800 especially interesting as a quality-oriented SDXL resolution.

---

## 1200 × 1800 — Memory Behavior

Representative measurements:

| Model | Batch | VAE mode | Peak dedicated VRAM | Peak shared memory | Total time |
|---|---:|---|---:|---:|---:|
| Model A | 2 | Tiled 800/96 | ~9.8 GB | ~0.74 GB | ~84.6 s |
| Model A | 2 | Standard | ~15.3 GB | ~6.75 GB | ~83.7 s |
| Model B | 2 | Tiled 800/96 | ~11.9 GB | ~0.74 GB | ~62.7 s |
| Model B | 2 | Standard | ~14.9 GB | ~0.74 GB | ~58.5 s |

The exact behavior differed by checkpoint, but tiled VAE consistently provided much cleaner memory behavior.

The tiled configuration used was:

> **Tile size: 800**  
> **Overlap: 96**

At this resolution, tiled decoding greatly reduced VRAM and/or shared-memory pressure without introducing a major runtime penalty.

This became the practical high-quality preset:

> **1200 × 1800**  
> **Batch 2**  
> **40 steps**  
> **Tiled VAE 800/96**

---

## Tiled VAE Was Not Always Slower

Tiled decoding is often treated as a low-memory fallback that necessarily trades speed for lower VRAM use.

The tests did not support such a simple rule.

At lower resolutions, standard decoding often remained faster and simpler.

At higher resolutions, tiled decoding sometimes avoided enough memory pressure that total runtime became comparable or even lower.

In other words:

> tiled VAE can become a memory-management optimization rather than merely a survival mode.

For this system, **800 / 96** proved to be a useful general high-resolution preset.

---

## 1600 × 2400 — Native High-Resolution Testing

1600 × 2400 contains:

> **3,840,000 pixels**

which is exactly:

> **4× the pixel count of 800 × 1200**

The test configuration was:

> **Batch 1**  
> **40 steps**

Both tiled and standard VAE decoding were tested.

Representative sampler times were approximately:

> **57–60 seconds for 40 steps**

or roughly:

> **1.45–1.52 seconds per iteration**

Sampler VRAM remained around the rough:

> **11–12 GB range**

depending on checkpoint and run.

From a hardware perspective, the resolution was therefore entirely feasible.

---

## 1600 × 2400 — VAE Behavior

Representative behavior:

| Resolution | VAE | Dedicated VRAM | Shared GPU memory | Total time |
|---|---|---:|---:|---:|
| 1600×2400 | Tiled 800/96 | ~12–13.4 GB | ~0.74 GB | ~64–67 s |
| 1600×2400 | Standard | ~12–14.2 GB | ~5–6.4 GB | ~66–72 s |

Standard decoding could spill several gigabytes into shared GPU/system memory.

Tiled decoding largely avoided that.

At this resolution, tiled VAE was therefore clearly preferable from a memory-management perspective and was sometimes slightly faster overall.

---

## High-Resolution Model Failure

The most important finding at 1600 × 2400 was not related to VRAM.

The hardware still had room.

The model itself became unreliable.

Across repeated generations, the outputs showed failures involving:

- anatomy
- trees
- objects
- overall spatial structure
- repeated or duplicated scene elements

One particularly obvious failure mode looked like part of the scene had been generated twice and placed into the image again.

The result resembled a visual **stamping or duplication effect**.

Parts of the composition could appear repeated above, below or next to another instance of the same structure.

This was not based on a single bad generation.

Multiple seeds were tested, and prompt wording was adjusted in attempts to suppress duplication and improve object consistency.

The problem persisted.

That made it increasingly clear that the issue was related to the checkpoint operating outside a comfortable native generation range rather than simply a bad seed or poorly worded prompt.

---

## Model Limit Before Hardware Limit

This became one of the strongest conclusions from the test series:

> **A resolution fitting into VRAM does not mean the model is comfortable generating at that resolution.**

There are at least two separate high-resolution limits.

### Hardware limit

The point where:

- sampler VRAM
- VAE VRAM
- shared memory
- runtime
- stability

become unacceptable.

### Model limit

The point where:

- anatomy
- objects
- scene structure
- spatial consistency

begin to degrade even though the hardware can still execute the workload.

At 1600 × 2400, several tested checkpoints began hitting the second limit before the first.

---

## Sampling-Step Experiments

A separate test series evaluated the effect of increasing sampling steps while keeping the seed fixed.

Typical step points included:

> **20, 30, 40, 50, 60, 70, 85, 100 and occasionally 120–130**

The broad pattern across the tested checkpoints was:

| Step range | Typical behavior |
|---|---|
| 20 → 30 | Often a large improvement |
| 30 → 40 | Usually still useful |
| 40 → 50 | Frequently useful refinement |
| 50 → 60 | Often still productive |
| 60 → 70 | Diminishing returns become common |
| 85 → 100 | Can improve strong seeds selectively |
| 120–130 | Frequently very little meaningful change |

Higher step counts could improve:

- lighting
- local detail
- surface refinement
- background cohesion
- overall polish

But they were not a reliable method for repairing a fundamentally poor seed.

This was particularly obvious with eye and anatomy problems.

A bad structural result often remained bad even after very large increases in step count.

---

## Seed Quality vs Step Count

The step testing changed the preferred production strategy.

Rather than spending 70–100 steps on every image, it proved more useful to generate more candidates at around:

> **40 steps**

and then rerender only the strongest seeds at higher step counts.

This is why the high-batch 800 × 1200 configuration became so valuable.

It spends compute on:

> **search breadth**

before spending additional compute on:

> **refinement depth**

For selected seeds, approximately:

> **50–70 steps**

often made sense, with:

> **85–100 steps**

reserved for images that were still visibly improving.

---

## Thermal Behavior

Heavy sampler runs frequently maintained GPU utilization at or near:

> **100%**

Typical temperatures were approximately:

### GPU hotspot

> **70–77 °C**

### GPU memory / junction

> roughly the mid-50s to around 60 °C

### CPU

> **~48–52 °C**

### Motherboard MOS / VRM

> **~35–41 °C**

These temperatures were well controlled on the tested system.

Again, the GPU uses a custom fan curve and the case runs relatively aggressive airflow, so these numbers should be interpreted in that context.

---

## Power Behavior

GPU board power during heavy sampler workloads commonly reached approximately:

> **200–230 W**

depending on checkpoint and run.

An occasional measured peak reached approximately:

> **~262 W**

while temperatures remained within the ranges described above.

Thermal throttling did not appear to be a practical limitation during the tested workloads.

---

## Practical Presets That Emerged

| Purpose | Resolution | Batch | Steps | VAE |
|---|---:|---:|---:|---|
| Seed hunting | 800×1200 | 12 | 40 | Standard |
| Higher-quality generation | 1200×1800 | 2 | 40 | Tiled 800/96 |
| Native high-resolution testing | 1600×2400 | 1 | 40 | Tiled 800/96 |
| Selected-seed refinement | variable | 1 | ~50–70+, selectively 85–100 | resolution-dependent |

These are not universal SDXL settings.

They are the operating points that emerged from this specific hardware/software configuration after repeated testing.

---

## Main Findings

**1. 16 GB VRAM can support surprisingly large SDXL batches.**  
At 800 × 1200, batch 12 remained practical and stayed mostly inside physical VRAM during sampling.

**2. VAE decoding can become the memory bottleneck even when sampling is completely healthy.**

**3. Dedicated VRAM alone does not describe the complete memory situation.**  
Shared GPU memory needs to be monitored as well.

**4. Tiled VAE is not simply a low-VRAM emergency mode.**  
At higher resolutions it can reduce memory pressure substantially and sometimes match or improve total runtime.

**5. 1200 × 1800 frequently produced genuinely better generations than 800 × 1200.**  
The improvement was not limited to additional pixel count; the model often constructed scenes and details more accurately.

**6. 1600 × 2400 exposed model-level failure before hardware failure.**  
Repeated seeds and prompt adjustments still produced anatomy, object and scene-structure problems including stamping-like duplication.

**7. Fitting into VRAM does not mean a resolution is a useful native generation resolution for a checkpoint.**

**8. Seed quality mattered more than extreme sampling-step counts.**  
Additional steps refined good seeds much more reliably than they rescued poor ones.

**9. Approximately 50–60 steps often delivered most of the useful refinement.**  
Beyond roughly 60–70, diminishing returns became increasingly common.

---

## Limitations

This is a practical single-system case study, not a universal SDXL benchmark.

Results depend on:

- GPU architecture
- drivers
- ROCm / PyTorch version
- ComfyUI version
- memory-management configuration
- operating system
- checkpoint
- sampler
- scheduler
- prompt
- aspect ratio
- resolution
- system RAM
- background processes
- cooling

The three tested checkpoints showed meaningful differences despite belonging to the same SDXL family and having nearly identical file sizes.

The results should therefore be interpreted as:

> **measured scaling behavior on one real system**

rather than:

> **universal SDXL limits.**

---

## Conclusion

The largest practical lesson from the testing was that SDXL performance cannot be reduced to a single number such as:

> **VRAM used**

The actual operating limit depends on several different factors:

> sampler VRAM  
> VAE decode behavior  
> shared-memory spill  
> batch size  
> native resolution  
> checkpoint behavior  
> seed quality

On this 16 GB AMD system, **800 × 1200 at batch 12** worked very well for broad seed searching.

Moving to **1200 × 1800 at batch 2** often improved the generation itself, not merely its pixel count, while tiled VAE decoding kept memory behavior controlled.

Native **1600 × 2400** generation was technically feasible from the hardware side, but repeated testing across seeds and prompt variations showed that several checkpoints became structurally unreliable at that resolution.

That leads to the main conclusion:

> **The best native resolution is not necessarily the highest resolution that fits into VRAM.**

It is the highest resolution at which both:

> **the hardware can execute the workload efficiently**

and

> **the model can still construct the image reliably.**
