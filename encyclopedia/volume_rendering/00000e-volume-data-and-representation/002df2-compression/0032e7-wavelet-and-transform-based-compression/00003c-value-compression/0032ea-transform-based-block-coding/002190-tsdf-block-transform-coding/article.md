# TSDF Block Transform Coding

Also called: 3D DCT coding, Intra-frame coding, Intra-frame compression

## Overview
TSDF Block Transform Coding is a compression method for Truncated Signed Distance Function (TSDF) volumes that decomposes the input into fixed-size voxel blocks, applies a 3D Discrete Cosine Transform (DCT) to map spatial values to frequency coefficients, and entropy codes the retained low-frequency components [S1]. It distinguishes itself from sibling methods like JPEG2000 by using a fixed, data-independent 3D DCT kernel rather than wavelets, and from Learned Transform Coding by avoiding the need to compute and transmit scene-specific transform parameters, thereby targeting real-time streaming across arbitrary scenes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A sparse TSDF volume represented as a set of active voxel blocks. An active block contains at least one voxel within the truncation band of the observed surface [S1].
*   **Output:** A compressed bitstream containing entropy-coded coefficients for each active block [S1].

**Data Structures:**
*   **Voxel Block:** A fixed-size 3D array of voxels. The standard implementation uses a block size of $8 \times 8 \times 8$ voxels [S1].
*   **Coefficient Stream:** A list of quantized DCT coefficients. The method groups coefficients by DCT index across all active blocks, producing $N$ separate streams rather than encoding each block as a mixed vector [S1].

**Parameters:**
*   **Block Size:** $8 \times 8 \times 8$ voxels [S1].
*   **$N$:** The number of retained low-frequency DCT coefficients. Typical practical operating range is $N \in [27, 64]$ [S1].
*   **$\Delta_{AC}$:** Tunable quantization step for AC coefficients. Smaller $\Delta_{AC}$ reduces distortion but increases bitrate [S1].
*   **$\Delta_{DC}$:** Fixed quantization step for the DC term. Value used in experiments: $0.001$ [S1].

**Algorithm Steps:**
1.  **Identify Active Blocks:** Iterate through the TSDF volume grid. Mark a block as "active" if it contains any voxel with a valid signed distance value within the truncation band [S1].
2.  **Transform:** For each active block, apply the 3D DCT to map the spatial voxel values $v(x,y,z)$ into frequency coefficients $C(u,v,w)$ [S1]. The 3D DCT basis is separable, with the DC kernel capturing the block average and higher indices representing increasing spatial frequencies along the axes [S1].
3.  **Select Coefficients:** Sort the DCT coefficients by increasing index sum ($u+v+w$), which corresponds to the Manhattan distance from the DC component (0,0,0). Retain the first $N$ coefficients in this order. Set all remaining coefficients to zero [S1].
4.  **Quantize:** Quantize the retained coefficients:
    *   For the DC coefficient ($C_{0,0,0}$): $Q_{DC} = \text{round}(C_{0,0,0} / \Delta_{DC})$ [S1].
    *   For AC coefficients ($C_{i}$ where $i \neq 0$): $Q_{AC} = \text{round}(C_{i} / \Delta_{AC})$ [S1].
5.  **Group Streams:** Organize the quantized coefficients into $N$ streams. Stream $k$ contains the $k$-th retained coefficient from every active block [S1].
6.  **Entropy Encode:** Apply lossless range coding to each of the $N$ streams independently [S1]. The range coder uses an adaptive probability model that updates symbol frequencies online as symbols are processed [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def tsdf_block_transform_coding(active_blocks, N, delta_ac, delta_dc):
    """
    Compresses a set of active TSDF blocks using 3D DCT and range coding.
    
    Args:
        active_blocks: List of 3D arrays (numpy-like), each of shape (8, 8, 8).
        N: Number of low-frequency coefficients to retain.
        delta_ac: Quantization step for AC coefficients.
        delta_dc: Quantization step for DC coefficient.
        
    Returns:
        streams: List of N lists, where streams[k] contains the quantized 
                 k-th coefficient from each block.
    """
    # Initialize N streams
    streams = [[] for _ in range(N)]
    
    for block in active_blocks:
        # Step 1: Apply 3D DCT
        # The 3D DCT of an 8x8x8 block can be computed via separable 1D DCTs
        # DCT_3D = DCT_1D(DCT_1D(DCT_1D(block, axis=0), axis=1), axis=2)
        # For implementation, we assume a standard Type-II DCT function exists.
        coeffs = apply_3d_dct(block)
        
        # Step 2: Select N lowest-frequency coefficients
        # Sort by index sum (Manhattan distance from DC)
        indices = []
        for u in range(8):
            for v in range(8):
                for w in range(8):
                    indices.append((u + v + w, u, v, w))
        
        indices.sort(key=lambda x: x[0])
        
        selected_coeffs = []
        for _, u, v, w in indices[:N]:
            selected_coeffs.append(coeffs[u, v, w])
            
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
            
        # Step 4: Group by index
        for k in range(N):
            streams[k].append(quantized[k])
            
    return streams

def apply_3d_dct(block):
    """
    Applies separable 3D DCT to an 8x8x8 block.
    """
    # Apply 1D DCT along axis 0
    temp1 = apply_1d_dct_8x8x8(block, axis=0)
    # Apply 1D DCT along axis 1
    temp2 = apply_1d_dct_8x8x8(temp1, axis=1)
    # Apply 1D DCT along axis 2
    result = apply_1d_dct_8x8x8(temp2, axis=2)
    return result

def apply_1d_dct_8x8x8(data, axis):
    """
    Applies 1D DCT-II to the last dimension of the data, 
    assuming the data is transposed so the transform axis is last.
    Standard 8-point DCT-II formula.
    """
    # Transpose so the transform axis is the last one
    data = numpy.moveaxis(data, axis, -1)
    shape = data.shape
    # Reshape to 1D array of 8-element vectors
    data_flat = data.reshape(-1, 8)
    
    # Precompute DCT basis matrix for N=8
    # C[k, n] = sqrt(2/N) * cos(pi * (2n+1) * k / (2N)) for k>0
    # C[0, n] = sqrt(1/N) * cos(pi * (2n+1) * 0 / (2N)) = sqrt(1/N)
    N = 8
    n = numpy.arange(N)
    k = numpy.arange(N).reshape(-1, 1)
    
    basis = numpy.zeros((N, N))
    basis[0, :] = numpy.sqrt(1.0 / N) * numpy.ones(N)
    for i in range(1, N):
        basis[i, :] = numpy.sqrt(2.0 / N) * numpy.cos((2 * n + 1) * i * numpy.pi / (2 * N))
        
    # Transform
    transformed = numpy.dot(data_flat, basis.T)
    
    # Reshape back
    result = transformed.reshape(shape)
    # Move axis back to original position
    result = numpy.moveaxis(result, -1, axis)
    return result
```

**Edge Cases and Pitfalls:**
*   **Non-watertight Surfaces:** For meshes that are non-watertight, thin, or overlapping, a global inside/outside sign is ill-defined. The method uses an unsigned-distance offset fallback to define the signed distance field before TSDF conversion [S1].
*   **Precision:** The choice of $\Delta_{AC}$ directly impacts the dynamic range of the quantized integers. A smaller step increases the number of non-zero coefficients, increasing the bitrate [S1].
*   **Streaming Overhead:** Explicit probability models (histograms) are avoided to prevent per-stream metadata overhead, which is critical for real-time streaming of many short blocks [S1].

## Subtopics

### [DCT Transform Method](5741-dct-transform-method.md)
The DCT Transform Method is a volume data compression technique that partitions input data into fixed-size blocks, subdivides them into smaller subblocks, and applies the 3D Discrete Cosine Transform (DCT) to convert spatial voxel values into frequency coefficients. It distinguishes itself from the Discrete Fourier Transform (DFT) approach by offering faster compression and decompression speeds, though it shares the limitation of requiring full reconstruction for volume rendering. It builds on [TSDF Block Transform Coding](2548-tsdf-block-transform-coding.md) by specifying the DCT as the transformation kernel.

### Adaptive Range Coding
Adaptive Range Coding is a lossless entropy coding technique using range coding with adaptive probability models. The encoder and decoder start from the same fixed initial model and update symbol frequencies online as the stream is encoded/decoded, avoiding explicit histogram transmission [S1]. This approach avoids the per-stream metadata overhead and complexity associated with explicit model approaches (building histograms and transmitting tables) when many short streams are produced [S1].

## Operating characteristics
*   **Time Complexity:** The transform is $O(B \log B)$ per block where $B$ is the number of voxels in a block ($8^3=512$). Since blocks are processed independently, encoding is highly parallelizable [S1].
*   **Memory Complexity:** Memory usage is proportional to the number of active blocks. The method stores only the active blocks and their coefficients, making it suitable for sparse volumes [S1].
*   **Measured Performance:** In the practical operating range $N \in [27, 64]$, the method achieves good reconstruction quality with low bitrate [S1]. The cross-scene KLT (Karhunen-Loève Transform) closely tracks the fixed DCT in this range, validating the efficiency of the fixed transform [S1].
*   **Strengths:** Real-time encoding capability due to fixed transforms and parallel block processing; no need to transmit transform parameters; effective energy compaction for locally smooth TSDF fields [S1].
*   **Limitations:** Requires full block reconstruction for rendering; less optimal than scene-specific learned transforms (KLT) for very small $N$, though the gain is modest in practical ranges [S1].
*   **When to use:** Real-time streaming of dynamic implicit surfaces where computational predictability and low latency are prioritized over maximum compression efficiency [S1].
*   **When not to use:** Offline compression where scene-specific optimization is feasible and latency is not a constraint [S1].

## Background
The method originates from the work on implicit surface compression using "Good Old Discrete Cosine Transform and Motion Compensation" [S1]. It was developed to enable real-time streaming of dynamic implicit surfaces, addressing the limitations of intra-only coding by incorporating temporal prediction and motion compensation in subsequent frames [S1]. The choice of DCT over learned transforms like KLT was motivated by the need to avoid computing and transmitting transform parameters in advance, targeting arbitrary scenes [S1].

## Sources
[S1] Implicit Surface Compression -- with Good Old Discrete Cosine Transform and Motion Compensation, https://doi.org/10.1145/3811298, fetched 2026-10-30
