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
2. **Hierarchical Vector Quantization:** Apply HVQ to the delta residuals $D_t$. HVQ decomposes the data into multiple resolution levels. At each level, blocks of voxels are quantized using codebooks learned from the training data (the delta residuals). The bit count per voxel is adjusted based on the range of data within each brick/block [S1].
3. **Entropy Coding:** Encode the resulting quantization indices and codebook information. The source notes mention that blocks are encoded with a LZO algorithm after quantization [S1].

**Parameters:**
- **Bit count per voxel:** Determined by the range of data inside the brick [S1].
- **Resolution levels:** Determined by the hierarchical structure of the HVQ. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Temporal Discontinuity:** Large changes between frames (e.g., object appearance/disappearance) result in large delta values, which may require higher bit counts or specialized handling. Not given in the sources.
- **Precision:** Delta encoding can amplify noise if the original data is noisy. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_time_varying_volume(frames):
    """
    Compress a sequence of volumetric frames using HVQ on Delta.
    frames: List of 3D numpy arrays, frames[t] is the volume at time t.
    """
    compressed_data = []
    base_frame = frames[0]
    compressed_data.append(("base", base_frame))
    for t in range(1, len(frames)):
        delta_residuals = frames[t] - frames[t-1]
        # HVQ quantization is defined in the parent article [827-hierarchical-vector-quantization.md]
        indices, codebooks = hvq_quantize(delta_residuals)
        # LZO is a standard lossless compression algorithm [G]
        encoded_indices = lzo_compress(indices)
        encoded_codebooks = lzo_compress(codebooks)
        compressed_data.append(("delta_hvq", t, encoded_indices, encoded_codebooks))
    return compressed_data

def decompress_time_varying_volume(compressed_data):
    """
    Decompress the data back into frames.
    """
    frames = []
    base_type, base_frame = compressed_data[0]
    frames.append(base_frame)
    prev_frame = base_frame
    for item in compressed_data[1:]:
        type_tag, t, encoded_indices, encoded_codebooks = item
        indices = lzo_decompress(encoded_indices)
        codebooks = lzo_decompress(encoded_codebooks)
        delta_residuals = hvq_dequantize(indices, codebooks)
        current_frame = prev_frame + delta_residuals
        frames.append(current_frame)
        prev_frame = current_frame
    return frames

def hvq_quantize(data):
    # Defined in parent article [827-hierarchical-vector-quantization.md]
    pass

def hvq_dequantize(indices, codebooks):
    # Defined in parent article [827-hierarchical-vector-quantization.md]
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
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Exploits both temporal and spatial redundancy [S1].
- **Limitations:** Requires temporal coherence for effective delta encoding. Not given in the sources.
- **When to use:** Time-varying volumetric data with high temporal coherence.
- **When not to use:** Data with large temporal discontinuities or no temporal coherence.

## Background
The method was proposed by She et al. [SBN11] as a way to compress time-varying volume data [S1]. It builds on the HVQ method [827-hierarchical-vector-quantization.md] and adds a temporal delta encoding step. The approach is part of a broader effort to compress GPU-based direct volume rendering data, as surveyed in [S1]. Other related works include Jang et al. [JEG12] and various wavelet-based approaches [Wes95, GS01, WS04, WGLS05, She06, WS09, MA09, WYM10] [S1].

## Sources
[S1] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
