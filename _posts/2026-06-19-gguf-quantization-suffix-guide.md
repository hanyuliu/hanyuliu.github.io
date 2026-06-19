---
title: "Decoding GGUF Quantization Suffixes"
date: 2026-06-19 00:00:00 -0800
categories: [AI, Local LLMs]
tags: [gguf, llama.cpp, quantization, local-llm, ai]
lang: en
---

You browse HuggingFace looking for a model to run locally, and the file list looks like this:

```
DeepSeek-R1-Distill-Qwen-7B-Q4_K_M.gguf
DeepSeek-R1-Distill-Qwen-7B-Q5_K_L.gguf
DeepSeek-R1-Distill-Qwen-7B-IQ4_XS.gguf
DeepSeek-R1-Distill-Qwen-7B-Q6_K_L.gguf
```

What do you pick? The suffix is a compact encoding of four independent decisions: quantization method, bit depth, internal precision variant, and which layers get special treatment. Once you know the grammar, the right file jumps out immediately.

## Q vs IQ: the quantization method

The first letter is either `Q` (K-Quant) or `IQ` (I-Quant, short for Importance Matrix Quant).

**K-Quants (`Q`)** are the safe default. They are fast on everything — CPU, GPU, Apple Metal, and AMD Vulkan. If you are not sure what backend you are running, use a `Q` model.

**I-Quants (`IQ`)** use a calibration dataset (the "imatrix") to figure out which weights matter most and quantize each part of the model accordingly. For the same file size you get meaningfully better output quality. The catch: they are slower on CPU and Apple Metal, and they do not work with Vulkan at all. If you are running on Nvidia (cuBLAS) or AMD ROCm (rocBLAS), `IQ` is worth reaching for.

## The number: bits per weight

The number after the prefix is how many bits each weight gets. Higher is better quality, bigger is larger file.

| Bits | Quality | Typical use |
|------|---------|-------------|
| 2 | Very low | Extreme RAM constraints only |
| 3 | Low | Low RAM, acceptable output |
| 4 | Good | Most common sweet spot |
| 5 | High | Recommended when RAM allows |
| 6 | Very high | Near-perfect quality |
| 8 | Near-lossless | Maximum quantized quality |
| F16 | Lossless (16-bit float) | Full precision, large file |
| F32 | Lossless (32-bit float) | Original model weights |

For most people `Q4` is where the sweet spot lives: small enough to fit in consumer GPU VRAM or a laptop's unified memory, good enough that the output quality degradation is hard to notice.

## The method letter: `_K`, `_0`, `_1`

After the bit depth comes the quantization method variant.

- **`_K`** — K-Quant, mixed precision. Different model layers use slightly different internal precisions to maximize overall quality. This is the modern default; choose it unless you have a specific reason not to.
- **`_0`** — Legacy uniform quantization per block. Simpler arithmetic, and it supports "online repacking" so ARM and AVX CPU inference can squeeze out more speed.
- **`_1`** — Legacy with a per-block scale factor added on top of `_0`. Marginally better quality; notably good tokens-per-watt on Apple Silicon.

In practice you will almost always be choosing between `_K` variants.

## The size variant: `_S`, `_M`, `_L`, `_XL`, `_XS`

K-Quants support a size variant that controls how aggressively the internal mixed precision is applied.

| Suffix | Name | What it means |
|--------|------|----------------|
| `_S` | Small | More aggressive compression; smaller file, slightly lower quality |
| `_M` | Medium | Balanced. The recommended default for general use |
| `_L` | Large | Less compression on important layers; better quality, slightly bigger |
| `_XL` | Extra Large | Minimal internal compression; best quality at this bit depth |
| `_XS` | Extra Small | Used with IQ-quants; more compressed than `_S` but with a good quality-to-size ratio |

When in doubt, `_M`.

## The `_L` suffix on the full name: upgraded embed/output layers

Some names look like `Q4_K_L` or `Q6_K_L`. The `_L` here is on the entire name, not just the size variant — it means the **embedding and output weight layers** have been bumped to Q8_0 (near-lossless), while the rest of the model stays at the base bit depth.

Those layers are the ones that translate between tokens and internal representations. They are disproportionately sensitive to quantization error. Upgrading just them costs a modest file size increase but noticeably improves response quality at the boundaries — especially for tokenization-heavy tasks like code, math, or non-English text.

| Example | Base quant | Embed/output quant |
|---------|------------|-------------------|
| `Q4_K_M` | Q4 K-quant | Default (Q4/Q5 range) |
| `Q4_K_L` | Q4 K-quant | **Q8_0** |
| `Q6_K_L` | Q6 K-quant | **Q8_0** |
| `Q2_K_L` | Q2 K-quant | **Q8_0** |

If you are on the fence between `Q4_K_M` and `Q5_K_M`, try `Q4_K_L` — it is often the better trade.

## The `_NL` suffix: non-linear IQ variants

`IQ4_NL` is the one name you will see with `_NL` (non-linear). It uses non-linear quantization levels, puts it roughly on par with `IQ4_XS` quality-wise, is slightly larger, but supports online repacking for ARM CPU inference — making it a good pick if you are running on an ARM CPU and want I-Quant quality without the full IQ speed penalty.

## How to choose

**Step 1 — figure out your memory budget.**
- GPU only (fastest): pick a file about 1–2 GB smaller than your VRAM.
- GPU + system RAM offloading: add VRAM and RAM, subtract 1–2 GB.

**Step 2 — Q or IQ?**
- Default to `Q` (K-Quants). Works everywhere.
- Use `IQ` only on Nvidia CUDA or AMD ROCm where it actually runs fast.

**Step 3 — pick the size variant.**
- Start with `_M`. Go to `_L` if you can afford slightly more memory and want better quality. Go to `_S` only if you are tight on space.

In a decision tree:

```
Limited RAM?
├── Yes → Q3_K_M or Q3_K_XL (or IQ3_M on Nvidia/AMD)
└── No → Can you fit Q4?
    ├── Yes → Q4_K_M (default) or IQ4_XS (Nvidia/AMD)
    └── Can you fit Q5 or Q6?
        ├── Q5 → Q5_K_M or Q5_K_L
        └── Q6 → Q6_K or Q6_K_L (near perfect)
```

## Backend compatibility

Not all quants run on all backends:

| Backend | K-quants (Q) | I-quants (IQ) |
|---------|:---:|:---:|
| CPU (x86/ARM) | ✅ | ✅ (slower) |
| Apple Metal | ✅ | ✅ (slower) |
| Nvidia CUDA | ✅ | ✅ |
| AMD ROCm | ✅ | ✅ |
| AMD Vulkan | ✅ | ❌ |

If you are using Vulkan, stick to K-Quants.

## Quick reference

| Format | Bits | Notes |
|--------|------|-------|
| `Q4_K_M` | 4 | Default recommendation for most users |
| `Q4_K_L` | 4 | Q8_0 embed/output; often better than Q5_K_M |
| `Q5_K_M` | 5 | High quality, recommended when RAM allows |
| `Q6_K` | 6 | Very high quality, near perfect |
| `Q6_K_L` | 6 | Q8_0 embed/output; as close to lossless as quantized gets |
| `Q8_0` | 8 | Near-lossless; maximum quantized quality |
| `IQ4_XS` | 4 | Smaller than Q4_K_S, similar quality (Nvidia/AMD only) |
| `IQ4_NL` | 4 | ARM-friendly repacking, IQ quality |

The next time you see a wall of `.gguf` files, read from left to right: method prefix, bit depth, precision variant, size variant, and whether the embed/output layers got the Q8_0 upgrade. Four pieces, one informed download.

---

*Based on llama.cpp GGUF quantization formats as documented in bartowski's HuggingFace releases.*
