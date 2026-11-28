# TSDF Block Transform Coding

Also called: 3D DCT coding, Intra-frame coding, Intra-frame compression

## Overview
TSDF Block Transform Coding is a compression method for Truncated Signed Distance Function (TSDF) volumes that decomposes the input into fixed-size voxel blocks, applies a 3D Discrete Cosine Transform (DCT) to map spatial values to frequency coefficients, and entropy codes the retained low-frequency components [S1]. It distinguishes itself from Learned Transform Coding by avoiding the need to compute and transmit scene-specific transform parameters, thereby targeting real-time streaming across arbitrary scenes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sparse TSDF volume represented as a set of active voxel blocks [S1].
*   **Output:** A compressed bitstream containing entropy-coded coefficients for each active block [S1].

**Data Structures:**
*   **Voxel Block:** A fixed-size 3D array of voxels [S1].
*   **Coefficient Stream:** A list of quantized DCT coefficients [S1].

**Parameters:**
*   **Block Size:** $8 \times 8 \times 8$ voxels [S1].
*   **$N$:** The number of retained low-frequency DCT coefficients [S1].
*   **$\Delta_{AC}$:** Tunable quantization step for AC coefficients [S1].
*   **$\Delta_{DC}$:** Fixed quantization step for the DC term [S1].

**Algorithm Steps:**
1.  **Identify Active Blocks:** Iterate through the TSDF volume grid and mark a block as active if it contains any voxel with a valid signed distance value within the truncation band [S1].
2.  **Transform:** For each active block, apply the 3D DCT to map the spatial voxel values $v(x,y,z)$ into frequency coefficients $C(u,v,w)$ [S1].
3.  **Select Coefficients:** Sort the DCT coefficients by increasing index sum ($u+v+w$), which corresponds to the Manhattan distance from the DC component $(0,0,0)$ [S1]. Retain the first $N$ coefficients in this order and set all remaining coefficients to zero [S1].
4.  **Quantize:** Quantize the retained coefficients using a fixed step for the DC term and a tunable step for AC coefficients [S1].
5.  **Group Streams:** Organize the quantized coefficients into $N$ streams, where stream $k$ contains the $k$-th retained coefficient from every active block [S1].
6.  **Entropy Encode:** Apply lossless range coding to each of the $N$ streams independently [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def tsdf_block_transform_coding(active_blocks, N, delta_ac, delta_dc):
    """
    Compresses a set of active TSDF blocks using 3D DCT and range coding.
    
    Args:
        active_blocks: List of 3D arrays (list of lists of lists), each of shape (8, 8, 8).
        N: Number of low-frequency coefficients to retain.
        delta_ac: Quantization step for AC coefficients.
        delta_dc: Quantization step for DC coefficient.
        
    Returns:
        streams: List of N lists, where streams[k] contains the quantized 
                 k-th coefficient from each block.
    """
    # Initialize N streams
    streams = [[] for _ in range(N)]
    
    # Precompute sorted indices for 8x8x8 block by Manhattan distance from DC
    indices = []
    for u in range(8):
        for v in range(8):
            for w in range(8):
                indices.append((u + v + w, u, v, w))
    indices.sort(key=lambda x: x[0])
    
    for block in active_blocks:
        # Step 1: Apply 3D DCT (separable)
        coeffs = apply_3d_dct(block)
        
        # Step 2: Select N lowest-frequency coefficients
        selected_coeffs = []
        for _, u, v, w in indices[:N]:
            selected_coeffs.append(coeffs[u][v][w])
            
        # Step 3: Quantize
        quantized = []
        for i, c in enumerate(selected_coeffs):
            if i == 0:
                # DC term
                q = round(c / delta_dc)
            else:
                # AC terms
                q = round(c / delta_ac)
            quantized.append(q)
            
        # Step 4: Group by index across blocks
        for k in range(N):
            streams[k].append(quantized[k])
            
    # Step 5: Entropy encode each stream using adaptive range coding [S1]
    # encoded_bitstreams = [range_encode(stream) for stream in streams]
    return streams

def apply_3d_dct(block):
    """Applies separable 3D DCT-II to an 8x8x8 block [G]."""
    # Apply 1D DCT along axis 0
    temp1 = [apply_1d_dct_8(row) for row in block]
    # Apply 1D DCT along axis 1
    temp2 = [[apply_1d_dct_8(col) for col in row] for row in temp1]
    # Apply 1D DCT along axis 2
    result = [[[apply_1d_dct_8(slice_val) for slice_val in row] for row in plane] for plane in temp2]
    return result

def apply_1d_dct_8(data):
    """Applies 1D DCT-II to an 8-element list [G]."""
    N = 8
    result = [0.0] * N
    for k in range(N):
        s = 0.0
        for n in range(N):
            s += data[n] * math.cos(math.pi * k * (2 * n + 1) / (2 * N))
        if k == 0:
            result[k] = math.sqrt(1.0 / N) * s
        else:
            result[k] = math.sqrt(2.0 / N) * s
    return result
```

**Edge Cases and Pitfalls:**
*   **Non-watertight Surfaces:** For meshes that are non-watertight, thin, or overlapping, a global inside/outside sign is ill-defined [S1].
*   **Precision:** The choice of $\Delta_{AC}$ directly impacts the dynamic range of the quantized integers [S1].
*   **Streaming Overhead:** Explicit probability models are avoided to prevent per-stream metadata overhead, which is critical for real-time streaming of many short blocks [S1].

## Subtopics

### [DCT Transform Method](5741-dct-transform-method.md)
The DCT Transform Method is a volume data compression technique that partitions input data into fixed-size blocks, subdivides them into smaller subblocks, and applies the 3D Discrete Cosine Transform (DCT) to convert spatial voxel values into frequency coefficients. It shares a similar encoding pipeline with the Discrete Fourier Transform (DFT) approach.

### Adaptive Range Coding
Adaptive Range Coding is a lossless entropy coding technique using range coding with adaptive probability models [S1]. The encoder and decoder start from the same fixed initial model and update symbol frequencies online as the stream is encoded or decoded, avoiding explicit histogram transmission [S1]. This approach avoids the per-stream metadata overhead and complexity associated with explicit model approaches when many short streams are produced [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Measured Performance:** In the practical operating range $N \in [27, 64]$, the method achieves good reconstruction quality with low bitrate [S1]. The cross-scene KLT closely tracks the fixed DCT in this range, validating the efficiency of the fixed transform [S1].
*   **Strengths:** Real-time encoding capability due to fixed transforms and parallel block processing; no need to transmit transform parameters; effective energy compaction for locally smooth TSDF fields [S1].
*   **Limitations:** Requires full block reconstruction for rendering; less optimal than scene-specific learned transforms for very small $N$, though the gain is modest in practical ranges [S1].
*   **When to use:** Real-time streaming of dynamic implicit surfaces where computational predictability and low latency are prioritized over maximum compression efficiency [S1].
*   **When not to use:** Not given in the sources [S1].

## Background
The method originates from the work on implicit surface compression using "Good Old Discrete Cosine Transform and Motion Compensation" [S1]. It was developed to enable real-time streaming of dynamic implicit surfaces, addressing the limitations of intra-only coding by incorporating temporal prediction and motion compensation in subsequent frames [S1]. The choice of DCT over learned transforms like KLT was motivated by the need to avoid computing and transmitting transform parameters in advance, targeting arbitrary scenes [S1].

## Sources
[S1] Implicit Surface Compression -- with Good Old Discrete Cosine Transform and Motion Compensation, https://doi.org/10.1145/3811298, fetched 2026-10-30
