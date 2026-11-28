# Block Truncation Coding
Also called: BTC, ETC2

## Overview
Block Truncation Coding (BTC) is referenced in the context of adaptations that support hardware-accelerated random access for volume rendering [S1]. These adaptations include VTC (a 3D extension of S3TC), ETC2 (texture compression using invalid combinations), and ASTC [S1]. The method addresses storage and bandwidth constraints by quantizing small blocks to a limited number of intensity components [S1].

## How it works
**Inputs and Outputs:** Not given in the sources. [S1]
**Data Structures:** Not given in the sources. [S1]
**Steps:** Not given in the sources. [S1]
**Formulas:** Not given in the sources. [S1]
**Parameters:** Not given in the sources. [S1]
**Edge Cases and Pitfalls:** Not given in the sources. [S1]
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Note: The sources do not provide the core algorithm. A standard implementation follows [G].
def btc_compress(block_values, k=2):
    N = len(block_values)
    if k == 2:
        # Standard heuristic: levels at min and max [G]
        L0 = min(block_values)
        L1 = max(block_values)
        levels = [L0, L1]
        bitplane = []
        for v in block_values:
            bitplane.append(1 if v >= (L0 + L1) / 2 else 0)
        return levels, bitplane
    else:
        raise ValueError("k must be 2 for standard BTC")

def btc_decompress(levels, bitplane):
    return [levels[idx] for idx in bitplane]
```

## Subtopics

### [Neural Texture Compression (BCF1 Variants)](16542-neural-texture-compression-bcf1-variants.md)
Neural Texture Compression (BCF1) is a method for compressing physically-based rendering (PBR) texture sets by storing latent features in BC1 format and decoding them using a Multi-Layer Perceptron (MLP) accelerated by hardware cooperative vectors. It solves the storage and memory bandwidth bottleneck of high-resolution PBR assets by leveraging existing block compression hardware for random access while using neural inference to reconstruct high-dimensional material properties. Unlike its sibling [Block Truncation Coding](6187-block-truncation-coding.md) which supports general hardware accelerated random access, BCF1 specifically targets PBR texture sets and uses low dynamic range (LDR) BC1 compression instead of high dynamic range (HDR) BC6, allowing for higher compression ratios or improved quality at the same footprint.

## Operating characteristics
**Time and Memory Complexity:** Not given in the sources. [S1]
**Measured Performance:** Not given in the sources. [S1]
**Strengths and Limitations:** Not given in the sources. [S1]
**When to Use:** Not given in the sources. [S1]
**When Not to Use:** Not given in the sources. [S1]

## Background
Block Truncation Coding was adapted for hardware-accelerated random access in volume rendering [S1]. These adaptations include VTC (a 3D extension of S3TC), ETC2 (texture compression using invalid combinations), and ASTC [S1]. Additionally, Agus et al. proposed a method for segmented datasets where small blocks are quantized to one or two components, encoded as a block type and a splitting plane [S1].

## Sources
[S1] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
