# Hierarchical Vector Quantization on Delta

Also called: HVQ on Delta

## Overview
Hierarchical Vector Quantization on Delta is a compression method for time-varying volumetric data that applies delta encoding to adjacent time steps before learning the resulting residuals with hierarchical vector quantization [S1].

## How it works
The method operates on time-varying volume data represented as a sequence of 3D grids [S1]. The core innovation is the application of delta encoding across time steps before applying the spatial compression [S1].

**Inputs and Outputs:**
- Input: A sequence of volumetric data frames $V_t$ for $t = 0, 1, \dots, T-1$ [S1].
- Output: A compressed bitstream containing the base frame, delta residuals, and HVQ codebooks/indices [S1].

**Data Structures:**
1. **Base Frame ($V_0$):** Stored uncompressed or with standard spatial compression [G].
2. **Delta Residuals ($D_t$):** For $t > 0$, $D_t(x,y,z) = V_t(x,y,z) - V_{t-1}(x,y,z)$ [S1].
3. **HVQ Codebooks:** Hierarchical sets of vectors learned from the delta residuals [S1].
4. **Indices:** Quantization indices mapping voxels to codebook entries [S1].

**Steps:**
1. **Temporal Delta Encoding:** Compute the difference between adjacent time steps. For each voxel $(x,y,z)$ at time $t$:
   $$ D_t(x,y,z) = V_t(x,y,z) - V_{t-1}(x,y,z) $$
   This step exploits temporal coherence [S1].
2. **Hierarchical Vector Quantization:** Apply HVQ to the delta residuals $D_t$. Each block is quantized with a proper bit count per voxel depending on the range of data inside the brick [S1].
3. **Entropy Coding:** Encode the resulting quantization indices and codebook information. All blocks are encoded with a LZO algorithm [S1].

**Parameters:**
- **Bit count per voxel:** Determined by the range of data inside the brick [S1].
- **Resolution levels:** Determined by the hierarchical structure of the HVQ [G].

**Edge Cases and Pitfalls:**
Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress(frames):
    compressed = []
    compressed.append(frames[0])
    for t in range(1, len(frames)):
        delta = frames[t] - frames[t-1]
        # HVQ encoding is defined in [827-hierarchical-vector-quantization.md]
        indices, codebooks = hvq_encode(delta)
        compressed.append((t, lzo_compress(indices), lzo_compress(codebooks)))
    return compressed

def decompress(compressed):
    frames = []
    frames.append(compressed[0])
    for item in compressed[1:]:
        t, enc_indices, enc_codebooks = item
        indices = lzo_decompress(enc_indices)
        codebooks = lzo_decompress(enc_codebooks)
        # HVQ decoding is defined in [827-hierarchical-vector-quantization.md]
        delta = hvq_decode(indices, codebooks)
        frames.append(frames[-1] + delta)
    return frames

def hvq_encode(data):
    # Defined in [827-hierarchical-vector-quantization.md]
    pass

def hvq_decode(indices, codebooks):
    # Defined in [827-hierarchical-vector-quantization.md]
    pass

def lzo_compress(data):
    # Standard LZO lossless compression algorithm [G]
    pass

def lzo_decompress(data):
    # Standard LZO lossless compression algorithm [G]
    pass
```

## Subtopics

### [Hierarchical Progressive Encoding](12107-hierarchical-progressive-encoding.md)
Hierarchical Progressive Encoding (HPC) is a compression method for dynamic volumetric video that exploits temporal coherence by encoding feature grids as residuals relative to the previous frame across multiple spatial resolutions. It differs from static volume compression by segmenting sequences into Groups of Features (GoFs) and using forward-referenced residuals, enabling adaptive streaming where lower-resolution details are transmitted first to satisfy varying bandwidth conditions.

## Operating characteristics
- **Strengths:** Exploits both temporal and spatial redundancy [S1].
- **Limitations:** Not given in the sources.
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
The method was proposed by She et al. [SBN11] as a way to compress time-varying volume data [S1]. It builds on the HVQ method [827-hierarchical-vector-quantization.md] and adds a temporal delta encoding step [S1]. The approach is part of a broader effort to compress GPU-based direct volume rendering data, as surveyed in [S1]. Other related works include Jang et al. [JEG12] and various wavelet-based approaches [Wes95, GS01, WS04, WGLS05, She06, WS09, MA09, WYM10] [S1].

## Sources
[S1] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
