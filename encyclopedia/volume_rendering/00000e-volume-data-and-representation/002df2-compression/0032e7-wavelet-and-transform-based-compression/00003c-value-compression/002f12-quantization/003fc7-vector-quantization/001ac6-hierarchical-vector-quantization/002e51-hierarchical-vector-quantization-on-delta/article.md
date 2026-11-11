# Hierarchical Vector Quantization on Delta

Also called: HVQ on Delta

## Overview
Hierarchical Vector Quantization on Delta is a compression method for time-varying volumetric data that combines temporal delta encoding with hierarchical vector quantization (HVQ) to achieve high compression ratios while maintaining direct rendering capabilities on programmable graphics hardware. It sets itself apart from sibling methods like Temporal/Spatial Vector Quantization by explicitly decoupling temporal redundancy (via delta encoding) from spatial redundancy (via HVQ), allowing the quantization codebooks to adapt to the residual data distribution rather than the raw data [S1].

## How it works
The method operates on time-varying volume data represented as a sequence of 3D grids. The core innovation is the application of delta encoding across time steps before applying the spatial compression.

**Inputs and Outputs:**
- Input: A sequence of volumetric data frames $V_t$ for $t = 0, 1, \dots, T-1$.
- Output: A compressed bitstream containing the base frame, delta residuals, and HVQ codebooks/indices.

**Data Structures:**
1.  **Base Frame ($V_0$):** Stored uncompressed or with standard spatial compression.
2.  **Delta Residuals ($D_t$):** For $t > 0$, $D_t(x,y,z) = V_t(x,y,z) - V_{t-1}(x,y,z)$.
3.  **HVQ Codebooks:** Hierarchical sets of vectors learned from the delta residuals.
4.  **Indices:** Quantization indices mapping voxels to codebook entries.

**Steps:**
1.  **Temporal Delta Encoding:** Compute the difference between adjacent time steps. For each voxel $(x,y,z)$ at time $t$:
    $$ D_t(x,y,z) = V_t(x,y,z) - V_{t-1}(x,y,z) $$
    This step exploits temporal coherence, reducing the magnitude of values to be compressed [S1].
2.  **Hierarchical Vector Quantization:** Apply HVQ to the delta residuals $D_t$. HVQ decomposes the data into multiple resolution levels. At each level, blocks of voxels are quantized using codebooks learned from the training data (the delta residuals). The bit count per voxel is adjusted based on the range of data within each brick/block [S1].
3.  **Entropy Coding:** Encode the resulting quantization indices and codebook information. The source notes mention that blocks are encoded with a LZO algorithm after quantization [S1].

**Parameters:**
-   **Bit count per voxel:** Determined by the range of data inside the brick. Not given in the sources.
-   **Resolution levels:** Determined by the hierarchical structure of the HVQ. Not given in the sources.

**Edge Cases and Pitfalls:**
-   **Temporal Discontinuity:** Large changes between frames (e.g., object appearance/disappearance) result in large delta values, which may require higher bit counts or specialized handling. Not given in the sources.
-   **Precision:** Delta encoding can amplify noise if the original data is noisy. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compress_time_varying_volume(frames):
    """
    Compress a sequence of volumetric frames using HVQ on Delta.
    frames: List of 3D numpy arrays, frames[t] is the volume at time t.
    """
    compressed_data = []
    
    # Step 1: Store base frame
    base_frame = frames[0]
    compressed_data.append(("base", base_frame))
    
    # Step 2: Delta encoding and HVQ for subsequent frames
    for t in range(1, len(frames)):
        current_frame = frames[t]
        prev_frame = frames[t-1]
        
        # Compute delta residuals
        delta_residuals = current_frame - prev_frame
        
        # Apply Hierarchical Vector Quantization
        # Note: The exact HVQ algorithm is defined in the parent article [827-hierarchical-vector-quantization.md]
        # Here we assume a function `hvq_quantize` that returns indices and codebooks
        indices, codebooks = hvq_quantize(delta_residuals)
        
        # Encode indices and codebooks
        # The source mentions LZO encoding for blocks [S1]
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
        
        # Decode LZO
        indices = lzo_decompress(encoded_indices)
        codebooks = lzo_decompress(encoded_codebooks)
        
        # Reverse HVQ quantization
        # Note: The exact HVQ algorithm is defined in the parent article [827-hierarchical-vector-quantization.md]
        delta_residuals = hvq_dequantize(indices, codebooks)
        
        # Reconstruct current frame
        current_frame = prev_frame + delta_residuals
        frames.append(current_frame)
        prev_frame = current_frame
        
    return frames
```

## Subtopics

### [Hierarchical Progressive Encoding](12107-hierarchical-progressive-encoding.md)
Hierarchical Progressive Encoding (HPC) is a method for compressing dynamic volumetric video by exploiting temporal coherence and multi-resolution spatial redundancy. It segments video sequences into Groups of Features (GoFs) and represents each frame's feature grid as a residual relative to the previous frame's grid, decomposed across multiple resolution levels. Unlike methods that transmit full radiance fields for every frame, HPC allows for adaptive rendering by selecting a subset of resolution levels based on bandwidth or computational constraints, enabling scalable quality adjustments under fluctuating network conditions.

## Operating characteristics
-   **Time Complexity:** Not given in the sources.
-   **Memory Complexity:** Not given in the sources.
-   **Measured Performance:** Not given in the sources.
-   **Strengths:** Exploits both temporal and spatial redundancy. Allows direct rendering on GPU via HVQ structure [S1].
-   **Limitations:** Requires temporal coherence for effective delta encoding. Not given in the sources.
-   **When to use:** Time-varying volumetric data with high temporal coherence.
-   **When not to use:** Data with large temporal discontinuities or no temporal coherence.

## Background
The method was proposed by She et al. [SBN11] as a way to compress time-varying volume data. It builds on the HVQ method [827-hierarchical-vector-quantization.md] and adds a temporal delta encoding step. The approach is part of a broader effort to compress GPU-based direct volume rendering data, as surveyed in [S1]. Other related works include Jang et al. [JEG12] and various wavelet-based approaches [Wes95, GS01, WS04, WGLS05, She06, WS09, MA09, WYM10] [S1].

## Sources
[S1] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
