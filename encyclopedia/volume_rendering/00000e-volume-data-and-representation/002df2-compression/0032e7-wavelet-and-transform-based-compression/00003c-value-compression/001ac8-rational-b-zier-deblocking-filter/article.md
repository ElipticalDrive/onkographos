# Rational Bézier Deblocking Filter

Also called: Deblocking Filter (Rational Bézier), Deblocking filter based on projecting voxel scalar values onto rational Bézier curves, Rational Bézier filtering

## Overview
The Rational Bézier Deblocking Filter is a post-processing technique applied during GPU volume rendering to reduce blocking artifacts at block boundaries in compressed volumetric data [S1, unofficial][S2, unofficial]. It solves the problem of signal discontinuities introduced by block-based compression (e.g., K-SVD, HVQ) by projecting voxel scalar values onto rational Bézier curves [S1, unofficial][S2, unofficial]. Unlike standard Bézier filtering, it uses adjustable weights based on local gradient magnitude to preserve high-frequency features while smoothing homogeneous regions [S1, unofficial][S2, unofficial]. It is distinct from its siblings in [Value compression](6-value-compression.md) because it is a data-dependent, block-aware filter that requires no knowledge of the specific compression algorithm used and modifies no stored data [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A decompressed volumetric scalar field $V$ subdivided into blocks $B$. The field is represented as $V = f(x)$ where $x \in \mathbb{R}^3$.
*   **Output:** A filtered volumetric scalar field with reduced blocking artifacts.

**Data Structures:**
*   **Volumetric Field $V$:** A 3D array of scalar values.
*   **Blocks $B$:** The volume is subdivided into cubic blocks. The source specifies a block size of $B=8$ voxels in the evaluation [S1, unofficial].
*   **Gradient Field:** A 3D array of gradient magnitudes computed from $V$.

**Parameters:**
*   $\lambda$: A user-defined parameter controlling filtering intensity. It balances artifact removal and feature preservation. The source identifies $\lambda = 3$ as a good compromise value through parameter space search [S1, unofficial][S2, unofficial].

**Algorithm Steps:**

1.  **Feature Detection:** Compute the gradient magnitude $|\nabla v|$ for every voxel $v$ in the decompressed volume $V$ using the central differences method [S1, unofficial][S2, unofficial].
2.  **Sequential Axis Filtering:** Apply the rational Bézier filter sequentially along the three main axis directions (X, Y, Z) of the volume [S1, unofficial][S2, unofficial].
    *   For a given axis direction (e.g., X), iterate over all pairs of adjacent blocks $b_i$ and $b_{i+1}$ [S1, unofficial][S2, unofficial].
    *   For each boundary between $b_i$ and $b_{i+1}$, identify the set of axis-aligned voxels $S$ that traverse the boundary. Each set $S$ consists of voxels from both blocks aligned along the filtering axis [S1, unofficial][S2, unofficial].
    *   Compute the rational Bézier curve defined by the samples in $S$ (treated as control points) [S1, unofficial][S2, unofficial].
    *   Modify the voxels closer to the border (specifically, half the voxels of each block) by projecting them onto the computed curve [S1, unofficial][S2, unofficial].
3.  **Final Intensity Calculation:** The final intensity $I_f(v)$ of a given voxel $v$ is the average of its three filtered values, one computed for each axis direction (X, Y, Z) [S1, unofficial][S2, unofficial].

**Formulas:**

The filtered intensity $I_f(t_S)$ for a sample $s$ within a set $S$ is computed as [S1, unofficial][S2, unofficial]:

$$
I_f(t_S) = \frac{\sum_{i=0}^n \binom{n}{i} t_S^i (1-t_S)^{n-i} I(s_i) \omega_i}{\sum_{i=0}^n \binom{n}{i} t_S^i (1-t_S)^{n-i} \omega_i}
$$

Where:
*   $n$ is the number of samples in the set $S$ (i.e., the number of control points minus one, or the degree of the curve if $n$ is the degree and there are $n+1$ points; the source defines $n$ as the number of samples of $S$ in the summation limit, implying $n+1$ control points if indices go $0$ to $n$, but standard Bézier degree is $n$ with $n+1$ points. The source text says "n is the number of samples of S" and sums $i=0$ to $n$. This implies $n+1$ samples. We follow the source's index range $0 \dots n$).
*   $t_S \in [0, 1]$ is the position of sample $s$ within $S$. $t_S = 0$ corresponds to the first control point, and $t_S = 1$ corresponds to the last [S1, unofficial][S2, unofficial].
*   $I(s_i)$ is the compressed intensity of the $i$-th sample (control point) $s_i$ [S1, unofficial][S2, unofficial].
*   $\omega_i$ is the weight associated with the $i$-th control point [S1, unofficial][S2, unofficial].

The weight $\omega_i$ is computed as [S1, unofficial][S2, unofficial]:

$$
\omega_i = e^{\lambda(1 - \|\nabla s_i\|)}
$$

Where:
*   $\lambda$ is the filtering intensity parameter [S1, unofficial][S2, unofficial].
*   $\|\nabla s_i\|$ is the gradient magnitude at sample position $s_i$ [S1, unofficial][S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def compute_gradient_magnitude(volume):
    """
    Computes gradient magnitude for each voxel using central differences.
    volume: 3D numpy array of shape (Dx, Dy, Dz)
    returns: 3D numpy array of gradient magnitudes
    """
    # Pad volume to handle boundaries with zeros or replication
    # Central differences:
    # grad_x[i] = (volume[i+1] - volume[i-1]) / 2
    # grad_y[i] = (volume[:, i+1, :] - volume[:, i-1, :]) / 2
    # grad_z[i] = (volume[:, :, i+1] - volume[:, :, i-1]) / 2
    
    # Note: Exact boundary handling is not specified in sources [G]. 
    # Standard central differences require padding or clamping.
    
    gx = np.zeros_like(volume)
    gy = np.zeros_like(volume)
    gz = np.zeros_like(volume)
    
    # Interior points
    gx[1:-1, :, :] = (volume[2:, :, :] - volume[:-2, :, :]) / 2.0
    gy[:, 1:-1, :] = (volume[:, 2:, :] - volume[:, :-2, :]) / 2.0
    gz[:, :, 1:-1] = (volume[:, :, 2:] - volume[:, :, :-2]) / 2.0
    
    # Boundary handling [G]: Set to 0 or copy nearest neighbor. 
    # Sources do not specify. We assume 0 for simplicity or that 
    # filtering is only applied to interior block voxels.
    
    magnitude = np.sqrt(gx**2 + gy**2 + gz**2)
    return magnitude

def rational_bezier_filter_1d(samples, weights, positions):
    """
    Computes the filtered value for a single sample using Rational Bézier formula.
    samples: list of intensity values I(s_i)
    weights: list of weights omega_i
    positions: list of t_S values for each sample in the set S
    n: degree of the curve (len(samples) - 1)
    """
    n = len(samples) - 1
    filtered_values = []
    
    for t in positions:
        numerator = 0.0
        denominator = 0.0
        
        for i in range(n + 1):
            # Binomial coefficient
            binom = math.comb(n, i)
            # Bernstein basis function
            basis = binom * (t ** i) * ((1 - t) ** (n - i))
            
            numerator += basis * samples[i] * weights[i]
            denominator += basis * weights[i]
            
        if denominator == 0:
            filtered_values.append(samples[0]) # Fallback [G]
        else:
            filtered_values.append(numerator / denominator)
            
    return filtered_values

def deblock_volume(volume, block_size=8, lambda_param=3):
    """
    Applies the Rational Bézier Deblocking Filter to a volume.
    volume: 3D numpy array
    block_size: size of each block in voxels
    lambda_param: filtering intensity parameter
    """
    Dx, Dy, Dz = volume.shape
    
    # Step 1: Feature Detection
    grad_mag = compute_gradient_magnitude(volume)
    
    # Initialize filtered volume with original values
    filtered_volume = volume.copy()
    
    # Step 2: Sequential Axis Filtering
    # We need to store intermediate filtered values for each axis
    filtered_x = volume.copy()
    filtered_y = volume.copy()
    filtered_z = volume.copy()
    
    # Helper to process one axis
    def process_axis(vol, grad, axis, filtered_vol):
        # Determine iteration ranges based on axis
        if axis == 0: # X-axis
            dim_range = range(Dx)
            other_dims = [(y, z) for y in range(Dy) for z in range(Dz)]
        elif axis == 1: # Y-axis
            dim_range = range(Dy)
            other_dims = [(x, z) for x in range(Dx) for z in range(Dz)]
        else: # Z-axis
            dim_range = range(Dz)
            other_dims = [(x, y) for x in range(Dx) for y in range(Dy)]
            
        for d in dim_range:
            for other in other_dims:
                # Extract the line of voxels along the axis for this position
                # This forms the set S
                if axis == 0:
                    line = vol[other[0]:other[0]+block_size*2, other[1], other[2]] # Assuming 2 blocks involved
                    # Note: The source says "half the voxels of each block" are modified.
                    # The set S includes voxels from both adjacent blocks.
                    # We need to identify which block boundary this line crosses.
                    # Blocks are indexed by block_id = d // block_size
                    pass 
                
                # Simplified logic for extracting sets S across boundaries
                # Iterate over block boundaries
                num_blocks = (vol.shape[axis] + block_size - 1) // block_size
                for b in range(num_blocks - 1):
                    start_idx = b * block_size
                    end_idx = (b + 1) * block_size
                    
                    # Extract voxels from block b and b+1 along the axis
                    # For axis 0:
                    if axis == 0:
                        s_voxels = vol[start_idx:end_idx, other[0], other[1]]
                        s2_voxels = vol[end_idx:end_idx+block_size, other[0], other[1]]
                    elif axis == 1:
                        s_voxels = vol[other[0], start_idx:end_idx, other[1]]
                        s2_voxels = vol[other[0], end_idx:end_idx+block_size, other[1]]
                    else:
                        s_voxels = vol[other[0], other[1], start_idx:end_idx]
                        s2_voxels = vol[other[0], other[1], end_idx:end_idx+block_size]
                        
                    # Combine into set S
                    samples = np.concatenate([s_voxels, s2_voxels])
                    grad_samples = np.concatenate([grad[start_idx:end_idx, other[0], other[1]], 
                                                   grad[end_idx:end_idx+block_size, other[0], other[1]]])
                    
                    # Compute weights
                    weights = np.exp(lambda_param * (1 - grad_samples))
                    
                    # Compute positions t_S
                    # t_S ranges from 0 to 1 over the entire set S
                    n = len(samples) - 1
                    positions = np.linspace(0, 1, len(samples))
                    
                    # Apply filter
                    filtered_vals = rational_bezier_filter_1d(samples, weights, positions)
                    
                    # Modify voxels closer to the border (half of each block)
                    # Half of block b (the last half) and half of block b+1 (the first half)
                    half_block = block_size // 2
                    
                    # Indices in the original volume to update
                    if axis == 0:
                        # Block b: indices [end_idx - half_block : end_idx]
                        # Block b+1: indices [end_idx : end_idx + half_block]
                        idx_b = slice(end_idx - half_block, end_idx)
                        idx_b1 = slice(end_idx, end_idx + half_block)
                        
                        # Map filtered values to these indices
                        # The set S has length 2*block_size.
                        # The first half_block values correspond to the end of block b?
                        # Or does the curve fit the whole line and we sample back?
                        # Source: "voxels closer to the border (half the voxels of each block) are modified according to the computed curves"
                        # This implies we replace the values of these specific voxels with the filtered values.
                        # We need to map the filtered_vals array to the specific voxels.
                        # Assuming the order in 'samples' matches the spatial order.
                        
                        # Last half of block b
                        filtered_vol[idx_b, other[0], other[1]] = filtered_vals[:half_block] # [G] Assumption on mapping
                        # First half of block b+1
                        filtered_vol[idx_b1, other[0], other[1]] = filtered_vals[half_block:2*half_block] # [G] Assumption
                        
                    elif axis == 1:
                        idx_b = slice(None), slice(end_idx - half_block, end_idx), slice(None)
                        idx_b1 = slice(None), slice(end_idx, end_idx + half_block), slice(None)
                        filtered_vol[idx_b] = filtered_vals[:half_block] # [G]
                        filtered_vol[idx_b1] = filtered_vals[half_block:2*half_block] # [G]
                        
                    else:
                        idx_b = slice(None), slice(None), slice(end_idx - half_block, end_idx)
                        idx_b1 = slice(None), slice(None), slice(end_idx, end_idx + half_block)
                        filtered_vol[idx_b] = filtered_vals[:half_block] # [G]
                        filtered_vol[idx_b1] = filtered_vals[half_block:2*half_block] # [G]

    process_axis(volume, grad_mag, 0, filtered_x)
    process_axis(filtered_x, grad_mag, 1, filtered_y)
    process_axis(filtered_y, grad_mag, 2, filtered_z)
    
    # Step 3: Final Intensity
    final_volume = (filtered_x + filtered_y + filtered_z) / 3.0
    
    return final_volume
```

**Edge Cases and Pitfalls:**
*   **Homogeneous Regions:** Artifacts are barely perceivable in homogeneous regions. The filter applies lower strength here to preserve details [S2, unofficial].
*   **High Gradient Regions:** Artifacts are clearly visible due to shading. The filter applies higher strength but must not be excessive to avoid blurring features [S2, unofficial].
*   **Compression Technique Dependency:** The filter is independent of the compression technique and does not require storing additional information in the compressed data [S1, unofficial][S2, unofficial].
*   **Computational Cost:** The filter is computationally intensive. While it achieves interactive frame rates, optimization for speed is noted as future work [S1, unofficial].

## Subtopics

### Comparison with Standard Bézier Filtering
Standard Bézier filtering uses uniform weights for control points. Rational Bézier filtering introduces adjustable weights ($\omega_i$) based on gradient magnitude, allowing for more accurate approximations to the uncompressed signal and better preservation of local features [S1, unofficial][S2, unofficial].

### Comparison with Simple Averaging
Simple averaging (e.g., downsampling) reduces memory footprint but loses high-frequency details. Rational Bézier filtering preserves main features better than simple averaging while removing blocking artifacts [S1, unofficial].

### Integration with Deferred Rendering
The filter is integrated into a decompress-filter-and-render architecture. It supports deferred filtering, adaptive multi-resolution data loading, levels of detail, and visibility culling [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The filter is described as "computationally intensive" [S1, unofficial]. It involves computing gradients, evaluating rational Bézier curves for every block boundary along three axes, and averaging results.
*   **Memory Complexity:** No additional storage is required in the compressed data format [S1, unofficial][S2, unofficial]. The filter operates on decompressed data in memory/GPU memory.
*   **Performance:** Achieves interactive frame rates on standard graphics PCs for massive static and dynamic datasets (e.g., 512^4 time-varying simulation) [S1, unofficial].
*   **Quality Metrics:**
    *   SSIM gain: 5–7% for K-SVD, 6–11% for HVQ [S1, unofficial].
    *   MS-SSIM gain: 2–8% for K-SVD, 4–6% for HVQ [S1, unofficial].
*   **Strengths:**
    *   Works with any block-based compression method without modification [S1, unofficial][S2, unofficial].
    *   Preserves local features better than non-adaptive filters [S1, unofficial][S2, unofficial].
    *   Pluggable into existing GPU rendering pipelines [S1, unofficial].
*   **Limitations:**
    *   Computationally expensive [S1, unofficial].
    *   Requires decompression before filtering [S1, unofficial].

## Background
The Rational Bézier Deblocking Filter was proposed by F. Marton, José A. Iglesias Guitián, Jose Díaz, and Enrico Gobbetti in the paper "Real-time Deblocked GPU Rendering of Compressed Volumes" (2014) [S1, unofficial][S2, unofficial]. It extends previous work on deferred filtering for GPU volume rendering [S1, unofficial]. The method addresses the visual quality degradation caused by aggressive compression of massive volumetric datasets, enabling real-time exploration of animated sequences by storing data on the graphics board [S1, unofficial].

## Sources
[S1] Real-time Deblocked GPU Rendering of Compressed Volumes, https://doi.org/10.2312/vmv.20141290, fetched 2026-10-26 (unofficial)
[S2] PDF Real-time deblocked GPU rendering of compressed volumes, https://www.crs4.it/vic/data/papers/vmv2014-volume-deblocking.pdf, fetched 2026-10-27 (unofficial)
