# Rational Bézier Deblocking Filter

Also called: Deblocking Filter (Rational Bézier), Deblocking filter based on projecting voxel scalar values onto rational Bézier curves, Rational Bézier filtering

## Overview
The Rational Bézier Deblocking Filter is a post-processing technique applied during GPU volume rendering to reduce blocking artifacts at block boundaries in compressed volume data [S1, unofficial][S2, unofficial]. It solves the problem of signal discontinuities introduced by block-based compression by projecting voxel scalar values onto rational Bézier curves [S1, unofficial][S2, unofficial]. Unlike standard Bézier filtering, it uses adjustable weights based on local gradient magnitude to preserve features while smoothing artifacts [S1, unofficial][S2, unofficial]. It balances artifact removal and feature preservation [S1, unofficial][S2, unofficial]. It is distinct from its siblings because it is a general, data-dependent filter that requires no a-priori knowledge of the compression technique or modifications to the encoded data [S1, unofficial][S2, unofficial].

## How it works
The method operates on a volume $V$ that has been subdivided into blocks $B$, where each block is independently compressed and decompressed [S1, unofficial][S2, unofficial]. The filter is applied sequentially along the three main axis directions (X, Y, Z) [S1, unofficial][S2, unofficial].

### Inputs and Outputs
- **Input:** A decompressed volumetric scalar field $V = f(x)$ where $x \in \mathbb{R}^3$, subdivided into a set of blocks $B$ [S1, unofficial][S2, unofficial].
- **Output:** A filtered volumetric scalar field with reduced blocking artifacts [S1, unofficial][S2, unofficial].

### Data Structures
- **Volume $V$:** A regular 3D grid of voxels [G].
- **Blocks $B$:** A partition of $V$ into smaller sub-volumes (e.g., $8 \times 8 \times 8$ voxels as used in evaluations) [S1, unofficial].
- **Gradient Field:** A 3D grid storing the gradient magnitude $|\nabla v|$ for each voxel $v$, computed via central differences [S1, unofficial][S2, unofficial].

### Parameters
- **$\lambda$:** A user-defined parameter controlling filtering intensity. It balances artifact removal and feature preservation. Typical value: $\lambda = 3$ [S1, unofficial][S2, unofficial].

### Steps
1. **Feature Detection:** Compute the gradient magnitude $|\nabla v|$ for every voxel $v$ in $V$ using the central differences method [S1, unofficial][S2, unofficial].
2. **Axis-wise Filtering:** For each axis direction (X, Y, Z):
   a. Iterate through all block boundaries along the current axis [S1, unofficial][S2, unofficial].
   b. For each boundary, extract sets of axis-aligned samples $S$ traversing the boundary. Each set $S$ consists of voxels from the two adjacent blocks aligned along the axis [S1, unofficial][S2, unofficial].
   c. For each set $S$, compute a rational Bézier curve using the samples as control points [S1, unofficial][S2, unofficial].
   d. Project the samples in $S$ onto this curve to compute filtered intensities [S1, unofficial][S2, unofficial].
   e. Update the voxels closer to the border (specifically, half the voxels of each block) with the projected values [S1, unofficial][S2, unofficial].
3. **Final Intensity:** For each voxel $v$, compute the final intensity $I_f(v)$ as the average of its three filtered values obtained from the X, Y, and Z passes [S1, unofficial][S2, unofficial].

### Formulas
The filtered intensity $I_f(t_S)$ for a sample $s$ within a set $S$ is given by:

$$
I_f(t_S) = \frac{\sum_{i=0}^{d} \binom{d}{i} t_S^i (1-t_S)^{d-i} I(s_i) \omega_i}{\sum_{i=0}^{d} \binom{d}{i} t_S^i (1-t_S)^{d-i} \omega_i}
$$

Where:
- $N$ is the number of samples in set $S$ [S1, unofficial][S2, unofficial].
- $d = N-1$ is the degree of the Bézier curve [G]. The source defines $n$ as the number of samples, but the binomial coefficient $\binom{n}{i}$ and standard Bézier notation require the degree to be $n-1$; implementation uses $d = N-1$ to resolve this notation conflict [S1, unofficial][S2, unofficial].
- $t_S \in [0, 1]$ is the normalized position of sample $s$ within $S$. $t_S = 0$ for the first control point, $t_S = 1$ for the last [S1, unofficial][S2, unofficial].
- $I(s_i)$ is the compressed intensity of the $i$-th sample (control point) [S1, unofficial][S2, unofficial].
- $\omega_i$ is the weight of the $i$-th control point [S1, unofficial][S2, unofficial].

The weight $\omega_i$ is computed as:

$$
\omega_i = e^{\lambda(1 - |\nabla s_i|)}
$$

Where:
- $\lambda$ is the filtering intensity parameter [S1, unofficial][S2, unofficial].
- $|\nabla s_i|$ is the gradient magnitude at sample $s_i$ [S1, unofficial][S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_gradient_magnitude(volume):
    """Computes gradient magnitude using central differences [S1, unofficial][S2, unofficial]."""
    dims = volume.shape
    grad_mag = [[[0.0] * dims[2] for _ in range(dims[1])] for _ in range(dims[0])]
    for x in range(1, dims[0] - 1):
        for y in range(1, dims[1] - 1):
            for z in range(1, dims[2] - 1):
                dx = (volume[x+1][y][z] - volume[x-1][y][z]) / 2.0
                dy = (volume[x][y+1][z] - volume[x][y-1][z]) / 2.0
                dz = (volume[x][y][z+1] - volume[x][y][z-1]) / 2.0
                grad_mag[x][y][z] = math.sqrt(dx*dx + dy*dy + dz*dz)
    # Boundary handling not specified in sources; use zero or one-sided differences [G]
    return grad_mag

def rational_bezier_filter_1d(samples, grad_mags, lambda_param):
    """Applies the rational Bézier filter to a 1D set of samples [S1, unofficial][S2, unofficial]."""
    N = len(samples)
    d = N - 1  # Degree of the curve
    filtered = [0.0] * N
    
    weights = [math.exp(lambda_param * (1.0 - grad_mags[i])) for i in range(N)]
    
    for i in range(N):
        t = i / d if d > 0 else 0.0
        numerator = 0.0
        denominator = 0.0
        for j in range(N):
            binom = math.comb(d, j)
            bernstein = binom * (t ** j) * ((1 - t) ** (d - j))
            numerator += bernstein * samples[j] * weights[j]
            denominator += bernstein * weights[j]
        filtered[i] = numerator / denominator if denominator > 0 else samples[i]
    return filtered

def deblock_volume(volume, block_size, lambda_param):
    """Applies the Rational Bézier Deblocking Filter to a 3D volume [S1, unofficial][S2, unofficial]."""
    dims = volume.shape
    grad_mag = compute_gradient_magnitude(volume)
    
    temp_x = [[[volume[x][y][z] for z in range(dims[2])] for y in range(dims[1])] for x in range(dims[0])]
    temp_y = [[[volume[x][y][z] for z in range(dims[2])] for y in range(dims[1])] for x in range(dims[0])]
    temp_z = [[[volume[x][y][z] for z in range(dims[2])] for y in range(dims[1])] for x in range(dims[0])]
    
    half_block = block_size // 2
    
    # X-axis pass
    for y in range(dims[1]):
        for z in range(dims[2]):
            num_blocks = dims[0] // block_size
            for b in range(num_blocks):
                start_x = b * block_size
                end_x = (b + 1) * block_size
                line_x = [volume[x][y][z] for x in range(start_x, end_x + block_size)]
                line_grad = [grad_mag[x][y][z] for x in range(start_x, end_x + block_size)]
                filtered_line = rational_bezier_filter_1d(line_x, line_grad, lambda_param)
                
                # Update first half of block b
                for i in range(half_block):
                    temp_x[start_x + i][y][z] = filtered_line[i]
                # Update second half of block b+1
                for i in range(block_size - half_block, block_size):
                    temp_x[end_x + i][y][z] = filtered_line[block_size + i]

    # Y-axis pass
    for x in range(dims[0]):
        for z in range(dims[2]):
            num_blocks = dims[1] // block_size
            for b in range(num_blocks):
                start_y = b * block_size
                end_y = (b + 1) * block_size
                line_y = [volume[x][y][z] for y in range(start_y, end_y + block_size)]
                line_grad = [grad_mag[x][y][z] for y in range(start_y, end_y + block_size)]
                filtered_line = rational_bezier_filter_1d(line_y, line_grad, lambda_param)
                for i in range(half_block):
                    temp_y[x][start_y + i][z] = filtered_line[i]
                for i in range(block_size - half_block, block_size):
                    temp_y[x][end_y + i][z] = filtered_line[block_size + i]

    # Z-axis pass
    for x in range(dims[0]):
        for y in range(dims[1]):
            num_blocks = dims[2] // block_size
            for b in range(num_blocks):
                start_z = b * block_size
                end_z = (b + 1) * block_size
                line_z = [volume[x][y][z] for z in range(start_z, end_z + block_size)]
                line_grad = [grad_mag[x][y][z] for z in range(start_z, end_z + block_size)]
                filtered_line = rational_bezier_filter_1d(line_z, line_grad, lambda_param)
                for i in range(half_block):
                    temp_z[x][y][start_z + i] = filtered_line[i]
                for i in range(block_size - half_block, block_size):
                    temp_z[x][y][end_z + i] = filtered_line[block_size + i]

    result = [[[0.0] * dims[2] for _ in range(dims[1])] for _ in range(dims[0])]
    for x in range(dims[0]):
        for y in range(dims[1]):
            for z in range(dims[2]):
                result[x][y][z] = (temp_x[x][y][z] + temp_y[x][y][z] + temp_z[x][y][z]) / 3.0
    return result
```

### Edge Cases and Pitfalls
- **Boundary Conditions:** The sources do not specify how to handle voxels at the volume boundary or blocks that are not fully filled [G]. Standard practice is to clamp or pad, but this is not defined in the method [G].
- **Block Size:** The method assumes a regular block decomposition. Irregular blocks are not supported [S1, unofficial][S2, unofficial].
- **Gradient Calculation:** Central differences require interior voxels. Boundary voxels in the gradient calculation need special handling (e.g., one-sided differences), which is not detailed in the sources [G].
- **Computational Cost:** The filter is computationally intensive. While it achieves interactive frame rates, optimization is noted as future work [S1, unofficial].

## Operating characteristics
- **Time Complexity:** The filter is separable and applied sequentially along three axes [S1, unofficial][S2, unofficial]. For each axis, it processes block boundaries [S1, unofficial][S2, unofficial]. The complexity is roughly $O(N)$ where $N$ is the number of voxels, but with a high constant factor due to the Bézier projection and gradient computation [S1, unofficial].
- **Memory Complexity:** Requires storage for the original volume, the gradient magnitude field, and temporary storage for the axis-wise filtered results. $O(N)$ additional memory [S1, unofficial][S2, unofficial].
- **Performance:** Achieves interactive frame rates on standard graphics PCs for massive static and dynamic datasets (e.g., $512^4$ time-varying simulation) [S1, unofficial].
- **Strengths:**
  - General and pluggable with various compression methods (K-SVD, HVQ) [S1, unofficial].
  - No a-priori knowledge of compression technique required [S1, unofficial][S2, unofficial].
  - Preserves local features better than simple averaging or standard Bézier filtering [S1, unofficial][S2, unofficial].
  - Improves visual quality (SSIM gain 5–11%, MS-SSIM gain 2–8%) [S1, unofficial].
- **Limitations:**
  - Computationally intensive [S1, unofficial].
  - Artifacts in homogeneous regions are barely perceivable, so the filter's adaptive strength is crucial [S2, unofficial].
  - Does not modify the compressed data encoding [S1, unofficial].

## Background
The Rational Bézier Deblocking Filter was proposed by F. Marton, José A. Iglesias Guitián, Jose Díaz, and Enrico Gobbetti in the paper "Real-time Deblocked GPU Rendering of Compressed Volumes" (2014) [S1, unofficial][S2, unofficial]. It extends deferred filtering for GPU volume rendering to multi-resolution settings, combining levels of detail with adaptive loading and visibility culling [S1, unofficial]. The method addresses the visual degradation caused by block-based compression in volume rendering, where shading emphasizes artifacts at high gradient regions [S2, unofficial]. It builds on the principle that rational Bézier curves provide more accurate approximations to the uncompressed signal than standard Bézier curves by using adjustable weights [S1, unofficial][S2, unofficial].

## Sources
[S1] Real-time Deblocked GPU Rendering of Compressed Volumes, https://doi.org/10.2312/vmv.20141290, fetched 2026-10-26 (unofficial)
[S2] PDF Real-time deblocked GPU rendering of compressed volumes, https://www.crs4.it/vic/data/papers/vmv2014-volume-deblocking.pdf, fetched 2026-10-27 (unofficial)
