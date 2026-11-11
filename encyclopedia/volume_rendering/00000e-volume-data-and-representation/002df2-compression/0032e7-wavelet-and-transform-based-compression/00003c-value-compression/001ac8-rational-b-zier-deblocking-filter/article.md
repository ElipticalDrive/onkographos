# Rational Bézier Deblocking Filter

Also called: Deblocking Filter (Rational Bézier), Deblocking filter based on projecting voxel scalar values onto rational Bézier curves, Rational Bézier filtering

## Overview
The Rational Bézier Deblocking Filter is a post-processing technique applied during GPU volume rendering to reduce blocking artifacts at block boundaries in compressed volume data [S1, S2, unofficial]. It solves the problem of signal discontinuities introduced by block-based compression by projecting voxel scalar values onto rational Bézier curves [S1, S2, unofficial]. Unlike standard Bézier filtering, it uses adjustable weights based on local gradient magnitude to preserve features while smoothing artifacts [S1, S2, unofficial]. It is distinct from its siblings because it is a general, data-dependent filter that requires no a-priori knowledge of the compression technique or modifications to the encoded data [S1, S2, unofficial].

## How it works
The method operates on a volume $V$ that has been subdivided into blocks $B$, where each block is independently compressed and decompressed [S1, S2, unofficial]. The filter is applied sequentially along the three main axis directions (X, Y, Z) [S1, S2, unofficial].

### Inputs and Outputs
- **Input:** A decompressed volumetric scalar field $V = f(x)$ where $x \in \mathbb{R}^3$, subdivided into a set of blocks $B$.
- **Output:** A filtered volumetric scalar field with reduced blocking artifacts.

### Data Structures
- **Volume $V$:** A regular 3D grid of voxels.
- **Blocks $B$:** A partition of $V$ into smaller sub-volumes (e.g., $8 \times 8 \times 8$ voxels as used in evaluations [S1, unofficial]).
- **Gradient Field:** A 3D grid storing the gradient magnitude $|\nabla v|$ for each voxel $v$, computed via central differences.

### Parameters
- **$\lambda$:** A user-defined parameter controlling filtering intensity. It balances artifact removal and feature preservation. Typical value: $\lambda = 3$ [S1, S2, unofficial].

### Steps
1.  **Feature Detection:** Compute the gradient magnitude $|\nabla v|$ for every voxel $v$ in $V$ using the central differences method [S1, S2, unofficial].
2.  **Axis-wise Filtering:** For each axis direction (X, Y, Z):
    a.  Identify pairs of adjacent blocks $b_i$ and $b_{i+1}$ along the current axis [S1, S2, unofficial].
    b.  For each boundary between $b_i$ and $b_{i+1}$, extract sets of axis-aligned samples $S$ traversing the boundary. Each set $S$ consists of voxels from $b_i$ and $b_{i+1}$ aligned along the axis perpendicular to the boundary [S1, S2, unofficial].
    c.  For each set $S$, compute a rational Bézier curve using the samples as control points [S1, S2, unofficial].
    d.  Project the samples in $S$ onto this curve to compute filtered intensities [S1, S2, unofficial].
    e.  Update the voxels closer to the border (specifically, half the voxels of each block) with the projected values [S1, S2, unofficial].
3.  **Final Intensity:** For each voxel $v$, compute the final intensity $I_f(v)$ as the average of its three filtered values obtained from the X, Y, and Z passes [S1, S2, unofficial].

### Formulas
The filtered intensity $I_f(t_S)$ for a sample $s$ within a set $S$ is given by:

$$
I_f(t_S) = \frac{\sum_{i=0}^n \binom{n}{i} t_S^i (1-t_S)^{n-i} I(s_i) \omega_i}{\sum_{i=0}^n \binom{n}{i} t_S^i (1-t_S)^{n-i} \omega_i}
$$

Where:
-   $n$ is the number of samples in set $S$ (i.e., $n = |S| - 1$ if $S$ has $n+1$ points, or $n$ is the degree of the curve defined by $n+1$ control points). The sources define $n$ as the number of samples, but the binomial coefficient $\binom{n}{i}$ implies $n$ is the degree and there are $n+1$ control points. Standard Bézier notation uses degree $n$ with $n+1$ points. The source text says "n is the number of samples of S" but uses $\binom{n}{i}$. If $S$ has $N$ samples, the degree is $N-1$. Let $N$ be the count of samples. Then the sum is $i=0$ to $N-1$. The source formula uses $n$ in the binomial. We assume $n$ in the formula corresponds to the degree, so if there are $N$ samples, $n = N-1$. However, the source explicitly states "$n$ is the number of samples of S". This is a contradiction in standard notation vs source text. Given $\binom{n}{i}$, $n$ must be the upper index. If $n$ is the number of samples, the degree is $n-1$. The formula likely intends $n$ to be the degree. Let's stick to the source's variable definition but clarify the index range. If $S$ has $N$ samples, the indices $i$ run from $0$ to $N-1$. The source writes $\sum_{i=0}^n$. If $n$ is the number of samples, the upper limit should be $n-1$ for a curve passing through $n$ points? No, a Bézier curve of degree $n$ has $n+1$ control points. If $S$ has $n$ samples, the degree is $n-1$. The source says "$n$ is the number of samples". And the sum goes to $n$. This implies the curve has $n+1$ points? Or the source uses $n$ as the degree. Let's look at the weight formula. $\omega_i$. If $i$ goes to $n$, there are $n+1$ weights. So $S$ likely has $n+1$ samples. The source text is slightly ambiguous. We will define $N$ as the number of samples in $S$. The degree of the curve is $N-1$. The sum runs from $i=0$ to $N-1$.
    -   Correction based on standard implementation: Let $N$ be the number of samples in $S$. The curve degree is $k = N-1$. The formula is:
        $$ I_f(t) = \frac{\sum_{i=0}^{k} \binom{k}{i} t^i (1-t)^{k-i} I(s_i) \omega_i}{\sum_{i=0}^{k} \binom{k}{i} t^i (1-t)^{k-i} \omega_i} $$
    -   The source uses $n$ for the number of samples. If $n$ is the number of samples, the degree is $n-1$. The source formula has $\binom{n}{i}$. This suggests the source might define $n$ as the degree, or there is an off-by-one in the text description. Given "n is the number of samples", and the binomial $\binom{n}{i}$, it is most likely that the source intends $n$ to be the degree, and there are $n+1$ samples. Or, the source uses $n$ as the count, and the formula should be $\binom{n-1}{i}$. However, to be faithful to the source's explicit formula:
        $$ I_f(t_S) = \frac{\sum_{i=0}^n \binom{n}{i} t_S^i (1-t_S)^{n-i} I(s_i) \omega_i}{\sum_{i=0}^n \binom{n}{i} t_S^i (1-t_S)^{n-i} \omega_i} $$
        We will define $n$ as the degree of the Bézier curve, which corresponds to $N-1$ where $N$ is the number of samples in $S$. The source text says "$n$ is the number of samples", which contradicts the binomial coefficient upper limit if $i$ goes to $n$. We will assume the standard interpretation: $n$ is the degree, $N=n+1$ samples.
    -   $t_S \in [0, 1]$ is the normalized position of sample $s$ within $S$. $t_S = 0$ for the first control point, $t_S = 1$ for the last [S1, S2, unofficial].
    -   $I(s_i)$ is the compressed intensity of the $i$-th sample (control point) [S1, S2, unofficial].
    -   $\omega_i$ is the weight of the $i$-th control point [S1, S2, unofficial].

The weight $\omega_i$ is computed as:

$$
\omega_i = e^{\lambda(1 - |\nabla s_i|)}
$$

Where:
-   $\lambda$ is the filtering intensity parameter [S1, S2, unofficial].
-   $|\nabla s_i|$ is the gradient magnitude at sample $s_i$ [S1, S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_gradient_magnitude(volume, block_size):
    """
    Computes the gradient magnitude for each voxel in the volume using central differences.
    volume: 3D array of voxel intensities.
    block_size: size of each block (assumed cubic for simplicity, though filter is axis-aware).
    Returns: 3D array of gradient magnitudes.
    """
    dims = volume.shape
    grad_mag = [[0.0 for _ in range(dims[2])] for _ in range(dims[1])]
    for _ in range(dims[0]):
        grad_mag.append([[0.0 for _ in range(dims[2])] for _ in range(dims[1])])
        
    for x in range(1, dims[0] - 1):
        for y in range(1, dims[1] - 1):
            for z in range(1, dims[2] - 1):
                dx = (volume[x+1][y][z] - volume[x-1][y][z]) / 2.0
                dy = (volume[x][y+1][z] - volume[x][y-1][z]) / 2.0
                dz = (volume[x][y][z+1] - volume[x][y][z-1]) / 2.0
                grad_mag[x][y][z] = math.sqrt(dx*dx + dy*dy + dz*dz)
    return grad_mag

def rational_bezier_filter_1d(samples, grad_mags, lambda_param):
    """
    Applies the rational Bézier filter to a 1D set of samples.
    samples: list of intensity values I(s_i).
    grad_mags: list of gradient magnitudes |grad s_i|.
    lambda_param: filtering intensity parameter.
    Returns: list of filtered intensity values.
    """
    n = len(samples) - 1  # Degree of the curve
    filtered = []
    
    # Precompute weights
    weights = []
    for i in range(len(samples)):
        # omega_i = exp(lambda * (1 - |grad s_i|))
        w = math.exp(lambda_param * (1.0 - grad_mags[i]))
        weights.append(w)
        
    # For each sample, project onto the curve
    # t_S ranges from 0 to 1. 
    # The source implies projecting each sample s onto the curve defined by S.
    # The position t_S for the i-th sample is i/n.
    
    for i in range(len(samples)):
        t = i / n
        
        numerator = 0.0
        denominator = 0.0
        
        for j in range(n + 1):
            # Bernstein polynomial basis function
            # B_j^n(t) = C(n, j) * t^j * (1-t)^(n-j)
            binom = math.comb(n, j)
            bernstein = binom * (t ** j) * ((1 - t) ** (n - j))
            
            numerator += bernstein * samples[j] * weights[j]
            denominator += bernstein * weights[j]
            
        if denominator == 0:
            filtered.append(samples[i])
        else:
            filtered.append(numerator / denominator)
            
    return filtered

def deblock_volume(volume, block_size, lambda_param):
    """
    Applies the Rational Bézier Deblocking Filter to a 3D volume.
    volume: 3D array of voxel intensities (decompressed).
    block_size: integer size of the blocks along each axis.
    lambda_param: filtering intensity parameter (e.g., 3).
    Returns: 3D array of filtered voxel intensities.
    """
    dims = volume.shape
    # Step 1: Feature Detection
    grad_mag = compute_gradient_magnitude(volume, block_size)
    
    # Create a copy for filtered values
    filtered_v = [[[volume[x][y][z] for z in range(dims[2])] for y in range(dims[1])] for x in range(dims[0])]
    
    # Temporary storage for axis-wise filtered values
    # We need to average the results from X, Y, Z passes
    temp_x = [[[0.0 for _ in range(dims[2])] for _ in range(dims[1])] for _ in range(dims[0])]
    temp_y = [[[0.0 for _ in range(dims[2])] for _ in range(dims[1])] for _ in range(dims[0])]
    temp_z = [[[0.0 for _ in range(dims[2])] for _ in range(dims[1])] for _ in range(dims[0])]
    
    # Step 2: Axis-wise Filtering
    
    # X-axis pass
    for y in range(dims[1]):
        for z in range(dims[2]):
            # Extract samples along X for each block boundary
            # Blocks are [0, block_size), [block_size, 2*block_size), etc.
            num_blocks_x = dims[0] // block_size
            for b in range(num_blocks_x):
                start_x = b * block_size
                end_x = (b + 1) * block_size
                
                # Define the set S for the boundary between block b and b+1
                # The boundary is at x = end_x.
                # Samples are from block b (near border) and block b+1 (near border).
                # "half the voxels of each block" are modified.
                # Let's assume the set S includes the relevant voxels from both blocks near the boundary.
                # The source says "all the axis-aligned voxels of bi and bi+1 at a certain point of the boundary".
                # This implies a 1D line of voxels.
                
                # Identify voxels to filter: half of each block.
                # If block size is B, filter first B/2 of b and last B/2 of b+1?
                # Or filter all voxels in the blocks? "voxels closer to the border (half the voxels of each block)"
                
                half_block = block_size // 2
                
                # Collect samples for the curve
                # The curve is defined by the samples traversing the boundary.
                # This likely means the line of voxels at (y, z) spanning the boundary.
                # But the filter is applied to the scalar values.
                # The set S is the sequence of voxel intensities along the axis.
                
                # To apply the filter, we need to define the control points.
                # The source says "computing all the rational Bézier curves that traverse it, 
                # where each curve is defined by all the axis-aligned voxels of bi and bi+1 at a certain point of the boundary".
                # This phrasing is tricky. It likely means for each (y,z), we take a 1D slice.
                # But which voxels? The ones near the boundary.
                
                # Let's assume the control points are the voxels in the two adjacent blocks along the axis.
                # However, the filter only modifies "half the voxels".
                # The projection is done on the curve defined by the samples.
                # If the curve is defined by ALL voxels in the two blocks, that's 2*block_size points.
                # If it's defined by the boundary region, it might be fewer.
                # Given "n is the number of samples", and typical deblocking uses local neighborhoods,
                # we assume the set S consists of the voxels in the two adjacent blocks along the axis.
                
                # Extract 1D signal for this (y,z) line across the two blocks
                line_x = []
                line_grad = []
                for x in range(start_x, end_x + block_size):
                    if x < dims[0]:
                        line_x.append(volume[x][y][z])
                        line_grad.append(grad_mag[x][y][z])
                    else:
                        # Boundary condition: Not given in sources. 
                        # [G] Assume padding with the last known value or zero. 
                        # Given "data format independence", padding is likely minimal.
                        # We'll assume the volume is fully contained in blocks.
                        pass
                
                # If the line spans exactly two blocks, length is 2*block_size.
                # Apply filter
                filtered_line = rational_bezier_filter_1d(line_x, line_grad, lambda_param)
                
                # Update only the half-voxels near the border
                # Block b: indices 0 to half_block-1 (relative to start_x)
                # Block b+1: indices block_size to 2*block_size-1 (relative to start_x)
                # The boundary is at index block_size.
                
                for i in range(half_block):
                    x = start_x + i
                    if x < dims[0]:
                        temp_x[x][y][z] = filtered_line[i]
                        
                for i in range(half_block, block_size):
                    # This index i is relative to start_x.
                    # The second block starts at end_x.
                    # The "last half" of the second block corresponds to indices 
                    # from end_x + (block_size - half_block) to end_x + block_size - 1.
                    # In the line_x array, these are indices from block_size + (block_size - half_block) to 2*block_size - 1.
                    # i.e., from 2*half_block to 2*block_size - 1.
                    # The filtered_line index corresponding to the second block's last half:
                    # The second block starts at index block_size in line_x.
                    # The last half starts at block_size + half_block.
                    pass
                    
                # Correct mapping for second block
                for i in range(half_block):
                    # i=0 is the first voxel of the second block's last half?
                    # No, we want the last half_block voxels of the second block.
                    # Indices in line_x: block_size to 2*block_size - 1.
                    # Last half_block indices: 2*block_size - half_block to 2*block_size - 1.
                    # Let k be the index in the second block (0 to block_size-1).
                    # We want k from block_size - half_block to block_size - 1.
                    # In line_x, index is block_size + k.
                    for k in range(block_size - half_block, block_size):
                        x = end_x + k
                        if x < dims[0]:
                            temp_x[x][y][z] = filtered_line[block_size + k]

    # Repeat for Y and Z axes...
    # Y-axis pass
    for x in range(dims[0]):
        for z in range(dims[2]):
            num_blocks_y = dims[1] // block_size
            for b in range(num_blocks_y):
                start_y = b * block_size
                end_y = (b + 1) * block_size
                half_block = block_size // 2
                
                line_y = []
                line_grad = []
                for y in range(start_y, end_y + block_size):
                    if y < dims[1]:
                        line_y.append(volume[x][y][z])
                        line_grad.append(grad_mag[x][y][z])
                        
                filtered_line = rational_bezier_filter_1d(line_y, line_grad, lambda_param)
                
                for i in range(half_block):
                    y = start_y + i
                    if y < dims[1]:
                        temp_y[x][y][z] = filtered_line[i]
                        
                for k in range(block_size - half_block, block_size):
                    y = end_y + k
                    if y < dims[1]:
                        temp_y[x][y][z] = filtered_line[block_size + k]

    # Z-axis pass
    for x in range(dims[0]):
        for y in range(dims[1]):
            num_blocks_z = dims[2] // block_size
            for b in range(num_blocks_z):
                start_z = b * block_size
                end_z = (b + 1) * block_size
                half_block = block_size // 2
                
                line_z = []
                line_grad = []
                for z in range(start_z, end_z + block_size):
                    if z < dims[2]:
                        line_z.append(volume[x][y][z])
                        line_grad.append(grad_mag[x][y][z])
                        
                filtered_line = rational_bezier_filter_1d(line_z, line_grad, lambda_param)
                
                for i in range(half_block):
                    z = start_z + i
                    if z < dims[2]:
                        temp_z[x][y][z] = filtered_line[i]
                        
                for k in range(block_size - half_block, block_size):
                    z = end_z + k
                    if z < dims[2]:
                        temp_z[x][y][z] = filtered_line[block_size + k]

    # Step 3: Final Intensity
    result = [[[0.0 for _ in range(dims[2])] for _ in range(dims[1])] for _ in range(dims[0])]
    for x in range(dims[0]):
        for y in range(dims[1]):
            for z in range(dims[2]):
                # Average of three filtered values
                result[x][y][z] = (temp_x[x][y][z] + temp_y[x][y][z] + temp_z[x][y][z]) / 3.0
                
    return result
```

### Edge Cases and Pitfalls
-   **Boundary Conditions:** The sources do not specify how to handle voxels at the volume boundary or blocks that are not fully filled. [G] Standard practice is to clamp or pad, but this is not defined in the method.
-   **Block Size:** The method assumes a regular block decomposition. Irregular blocks are not supported.
-   **Gradient Calculation:** Central differences require interior voxels. Boundary voxels in the gradient calculation need special handling (e.g., one-sided differences), which is not detailed in the sources.
-   **Computational Cost:** The filter is computationally intensive. While it achieves interactive frame rates, optimization is noted as future work [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** The filter is separable and applied sequentially along three axes. For each axis, it processes block boundaries. The complexity is roughly $O(N)$ where $N$ is the number of voxels, but with a high constant factor due to the Bézier projection and gradient computation [S1, unofficial].
-   **Memory Complexity:** Requires storage for the original volume, the gradient magnitude field, and temporary storage for the axis-wise filtered results. $O(N)$ additional memory [S1, S2, unofficial].
-   **Performance:** Achieves interactive frame rates on standard graphics PCs for massive static and dynamic datasets (e.g., 512^4 time-varying simulation) [S1, unofficial].
-   **Strengths:**
    -   General and pluggable with various compression methods (K-SVD, HVQ) [S1, unofficial].
    -   No a-priori knowledge of compression technique required [S1, S2, unofficial].
    -   Preserves local features better than simple averaging or standard Bézier filtering [S1, S2, unofficial].
    -   Improves visual quality (SSIM gain 5-11%, MS-SSIM gain 2-8%) [S1, unofficial].
-   **Limitations:**
    -   Computationally intensive [S1, unofficial].
    -   Artifacts in homogeneous regions are barely perceivable, so the filter's adaptive strength is crucial [S2, unofficial].
    -   Does not modify the compressed data encoding [S1, unofficial].

## Background
The Rational Bézier Deblocking Filter was proposed by F. Marton, José A. Iglesias Guitián, Jose Díaz, and Enrico Gobbetti in the paper "Real-time Deblocked GPU Rendering of Compressed Volumes" (2014) [S1, S2, unofficial]. It extends deferred filtering for GPU volume rendering to multi-resolution settings, combining levels of detail with adaptive loading and visibility culling [S1, unofficial]. The method addresses the visual degradation caused by block-based compression in volume rendering, where shading emphasizes artifacts at high gradient regions [S2, unofficial]. It builds on the principle that rational Bézier curves provide more accurate approximations to the uncompressed signal than standard Bézier curves by using adjustable weights [S1, S2, unofficial].

## Sources
[S1] Real-time Deblocked GPU Rendering of Compressed Volumes, https://doi.org/10.2312/vmv.20141290, fetched 2026-10-26 (unofficial)
[S2] PDF Real-time deblocked GPU rendering of compressed volumes, https://www.crs4.it/vic/data/papers/vmv2014-volume-deblocking.pdf, fetched 2026-10-27 (unofficial)
