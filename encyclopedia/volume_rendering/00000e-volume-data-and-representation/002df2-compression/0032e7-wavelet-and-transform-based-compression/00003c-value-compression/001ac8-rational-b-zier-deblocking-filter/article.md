# Rational Bézier Deblocking Filter

Also called: Deblocking Filter (Rational Bézier), Deblocking filter based on projecting voxel scalar values onto rational Bézier curves, Rational Bézier filtering

## Overview
The Rational Bézier Deblocking Filter is a post-processing technique applied during GPU volume rendering to reduce blocking artifacts at block boundaries in compressed volume data [S1, S2, unofficial]. It solves the problem of signal discontinuities introduced by block-based compression by projecting voxel scalar values onto rational Bézier curves [S1, S2, unofficial]. It sets itself apart from standard Bézier filtering by using adjustable weights based on local gradient magnitude, which allows for better preservation of high-frequency features while smoothing artifacts [S1, S2, unofficial]. Unlike methods that require knowledge of the compression algorithm, this filter is data-dependent and block-aware but compression-agnostic [S1, S2, unofficial].

## How it works
The filter operates on a volumetric scalar field $V$ that has been subdivided into blocks $B$, compressed, and decompressed. It is applied as a deferred post-processing step during rendering, supporting adaptive multi-resolution data loading and visibility culling [S1, unofficial].

### Inputs and Outputs
- **Input**: Decompressed volumetric scalar field $V = f(x)$ where $x \in \mathbb{R}^3$, subdivided into a set of blocks $B$.
- **Output**: Filtered volumetric scalar field with reduced blocking artifacts.

### Data Structures
- **Volumetric Scalar Field $V$**: A 3D grid of voxels.
- **Blocks $B$**: The volume is partitioned into cubic blocks. The source specifies a block size of $B=8$ voxels in the evaluation [S1, unofficial].
- **Gradient Magnitude Field**: A 3D grid storing the gradient magnitude $|\nabla v|$ for each voxel $v$, computed via central differences.

### Parameters
- **$\lambda$**: A user-defined parameter controlling the intensity of the filtering and the trade-off between artifact removal and feature preservation. The typical value is $\lambda = 3$ [S1, S2, unofficial].

### Algorithm Steps
1.  **Feature Detection**: Compute the gradient magnitude $|\nabla v|$ for every voxel $v$ in the decompressed volume $V$ using the central differences method [S1, S2, unofficial].
2.  **Iterate Axes**: Apply the rational Bézier filter sequentially along the three main axis directions (X, Y, Z) of the volume [S1, S2, unofficial].
3.  **Boundary Processing**: For each axis direction, iterate over all pairs of adjacent blocks $b_i$ and $b_{i+1}$ sharing a boundary [S1, S2, unofficial].
4.  **Curve Construction**: For each line of voxels crossing the boundary between $b_i$ and $b_{i+1}$, define a set $S$ of aligned samples. This set consists of the axis-aligned voxels from both blocks at that specific boundary location [S1, S2, unofficial].
5.  **Weight Calculation**: For each sample $s_i \in S$, compute the weight $\omega_i$ using the precomputed gradient magnitude [S1, S2, unofficial]:
    $$ \omega_i = e^{\lambda(1 - |\nabla s_i|)} $$
    where $|\nabla s_i|$ is the gradient magnitude at sample $s_i$ [S1, S2, unofficial].
6.  **Filtering**: For each sample $s \in S$, compute the filtered intensity $I_f(t_s)$ by projecting $s$ onto the rational Bézier curve defined by the control points $S$ [S1, S2, unofficial]:
    $$ I_f(t_s) = \frac{\sum_{i=0}^{n} \binom{n}{i} t_s^i (1 - t_s)^{n-i} I(s_i) \omega_i}{\sum_{i=0}^{n} \binom{n}{i} t_s^i (1 - t_s)^{n-i} \omega_i} $$
    where:
    - $n$ is the number of samples in set $S$ (i.e., $n = |S| - 1$).
    - $I(s_i)$ is the compressed intensity of the $i$-th control point $s_i$.
    - $t_s \in [0, 1]$ is the normalized position of sample $s$ within the set $S$ ($t_s=0$ for the first control point, $t_s=1$ for the last) [S1, S2, unofficial].
    - $\omega_i$ is the weight of the $i$-th control point [S1, S2, unofficial].
7.  **Update Voxels**: Modify the voxels closer to the border (specifically, half the voxels of each block adjacent to the boundary) with the computed filtered values $I_f(t_s)$ [S1, S2, unofficial].
8.  **Final Intensity**: For each voxel $v$, the final filtered intensity $I_f(v)$ is the average of its three filtered values obtained from the X, Y, and Z passes [S1, S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_gradient_magnitude(volume, block_size):
    """
    Computes gradient magnitude for all voxels using central differences.
    volume: 3D array of voxel intensities.
    block_size: integer size of the blocks.
    Returns: 3D array of gradient magnitudes.
    """
    dims = volume.shape
    grad_mag = [[0.0 for _ in range(dims[2])] for _ in range(dims[1])]
    for _ in range(dims[0]):
        grad_mag.append([[0.0 for _ in range(dims[2])] for _ in range(dims[1])])

    for x in range(1, dims[0] - 1):
        for y in range(1, dims[1] - 1):
            for z in range(1, dims[2] - 1):
                dx = volume[x+1][y][z] - volume[x-1][y][z]
                dy = volume[x][y+1][z] - volume[x][y-1][z]
                dz = volume[x][y][z+1] - volume[x][y][z-1]
                grad_mag[x][y][z] = math.sqrt(dx*dx + dy*dy + dz*dz)
    return grad_mag

def compute_weight(grad_mag, lambda_param):
    """
    Computes the weight omega_i for a sample based on gradient magnitude.
    grad_mag: gradient magnitude at the sample.
    lambda_param: filtering intensity parameter.
    Returns: weight omega.
    """
    return math.exp(lambda_param * (1.0 - grad_mag))

def binomial_coefficient(n, k):
    """Computes n choose k."""
    if k < 0 or k > n:
        return 0
    if k == 0 or k == n:
        return 1
    if k > n // 2:
        k = n - k
    
    res = 1
    for i in range(k):
        res = res * (n - i) // (i + 1)
    return res

def rational_bezier_filter(samples, grad_mags, lambda_param):
    """
    Applies the rational Bézier filter to a 1D set of samples.
    samples: list of intensity values I(s_i).
    grad_mags: list of gradient magnitudes |grad s_i|.
    lambda_param: filtering intensity parameter.
    Returns: list of filtered intensity values I_f(t_s).
    """
    n = len(samples) - 1
    filtered = [0.0] * len(samples)
    
    # Precompute weights
    weights = [compute_weight(g, lambda_param) for g in grad_mags]
    
    for i, s in enumerate(samples):
        t_s = i / n if n > 0 else 0.0
        numerator = 0.0
        denominator = 0.0
        
        for j in range(len(samples)):
            # Bernstein basis polynomial B_{i,n}(t)
            # Note: The formula uses t_s^i (1-t_s)^{n-i} for the i-th term.
            # In standard notation, B_{i,n}(t) = C(n,i) * t^i * (1-t)^{n-i}
            # Here, the index in the sum corresponds to the control point index.
            # The sample 's' is at index 'i' in the list 'samples', so t_s corresponds to i/n.
            # The control point is at index 'j'.
            
            binom = binomial_coefficient(n, j)
            t_pow = math.pow(t_s, j)
            one_minus_t_pow = math.pow(1.0 - t_s, n - j)
            
            basis = binom * t_pow * one_minus_t_pow
            numerator += basis * samples[j] * weights[j]
            denominator += basis * weights[j]
            
        if denominator > 0:
            filtered[i] = numerator / denominator
        else:
            filtered[i] = samples[i] # Fallback
            
    return filtered

def apply_deblocking_filter(volume, block_size, lambda_param=3.0):
    """
    Applies the Rational Bézier Deblocking Filter to the volume.
    volume: 3D array of voxel intensities (decompressed).
    block_size: integer size of the blocks.
    lambda_param: filtering intensity parameter.
    Returns: 3D array of filtered voxel intensities.
    """
    dims = volume.shape
    # Initialize output with original values
    filtered_volume = [[[volume[x][y][z] for z in range(dims[2])] for y in range(dims[1])] for x in range(dims[0])]
    
    # Step 1: Compute gradient magnitude
    grad_mag = compute_gradient_magnitude(volume, block_size)
    
    # Temporary storage for filtered values per axis
    # We will accumulate the sum of filtered values for each axis
    filtered_x = [[[0.0 for _ in range(dims[2])] for _ in range(dims[1])] for _ in range(dims[0])]
    filtered_y = [[[0.0 for _ in range(dims[2])] for _ in range(dims[1])] for _ in range(dims[0])]
    filtered_z = [[[0.0 for _ in range(dims[2])] for _ in range(dims[1])] for _ in range(dims[0])]
    
    # Step 2 & 3: Iterate axes and adjacent blocks
    # X-axis filtering
    for b_x in range(0, dims[0], block_size):
        for b_y in range(0, dims[1], block_size):
            for b_z in range(0, dims[2], block_size):
                # Process boundary between block (b_x, b_y, b_z) and (b_x+block_size, b_y, b_z)
                # Boundary is at x = b_x + block_size
                # We need to filter voxels near the boundary.
                # The source says "voxels closer to the border (half the voxels of each block)".
                # Let's assume we filter the last half of the left block and first half of the right block.
                
                # Left block range: [b_x, b_x + block_size)
                # Right block range: [b_x + block_size, b_x + 2*block_size)
                
                # Boundary index
                boundary_x = b_x + block_size
                
                # Check if right block exists
                if boundary_x >= dims[0]:
                    continue
                    
                # Define the region to filter in X direction
                # Half of block size
                half_block = block_size // 2
                
                # Left block voxels to filter: from boundary_x - half_block to boundary_x - 1
                # Right block voxels to filter: from boundary_x to boundary_x + half_block - 1
                
                start_x_left = max(b_x, boundary_x - half_block)
                end_x_left = boundary_x # exclusive
                
                start_x_right = boundary_x
                end_x_right = min(boundary_x + half_block, dims[0]) # exclusive
                
                # Iterate over all Y and Z coordinates within the blocks
                for y in range(b_y, min(b_y + block_size, dims[1])):
                    for z in range(b_z, min(b_z + block_size, dims[2])):
                        
                        # Construct set S for this (y, z) line crossing the boundary
                        # S contains samples from left block and right block along X
                        # The source says "all axis-aligned voxels of bi and bi+1 at a certain point of the boundary".
                        # This implies the entire line segment crossing the boundary.
                        # However, the formula applies to the set S.
                        # Let's collect all samples in the line segment that crosses the boundary.
                        
                        # To be precise, the filter is applied to the samples near the boundary.
                        # The source says "voxels closer to the border ... are modified".
                        # So S should likely include the voxels being modified.
                        
                        samples_x = []
                        grad_mags_x = []
                        
                        # Add left block voxels
                        for x in range(start_x_left, end_x_left):
                            samples_x.append(volume[x][y][z])
                            grad_mags_x.append(grad_mag[x][y][z])
                            
                        # Add right block voxels
                        for x in range(start_x_right, end_x_right):
                            samples_x.append(volume[x][y][z])
                            grad_mags_x.append(grad_mag[x][y][z])
                            
                        # Apply filter
                        filtered_vals = rational_bezier_filter(samples_x, grad_mags_x, lambda_param)
                        
                        # Write back to filtered_x
                        idx = 0
                        for x in range(start_x_left, end_x_left):
                            filtered_x[x][y][z] = filtered_vals[idx]
                            idx += 1
                        for x in range(start_x_right, end_x_right):
                            filtered_x[x][y][z] = filtered_vals[idx]
                            idx += 1

    # Y-axis filtering
    for b_y in range(0, dims[1], block_size):
        for b_x in range(0, dims[0], block_size):
            for b_z in range(0, dims[2], block_size):
                boundary_y = b_y + block_size
                if boundary_y >= dims[1]:
                    continue
                    
                half_block = block_size // 2
                start_y_left = max(b_y, boundary_y - half_block)
                end_y_left = boundary_y
                start_y_right = boundary_y
                end_y_right = min(boundary_y + half_block, dims[1])
                
                for x in range(b_x, min(b_x + block_size, dims[0])):
                    for z in range(b_z, min(b_z + block_size, dims[2])):
                        samples_y = []
                        grad_mags_y = []
                        
                        for y in range(start_y_left, end_y_left):
                            samples_y.append(volume[x][y][z])
                            grad_mags_y.append(grad_mag[x][y][z])
                        for y in range(start_y_right, end_y_right):
                            samples_y.append(volume[x][y][z])
                            grad_mags_y.append(grad_mag[x][y][z])
                            
                        filtered_vals = rational_bezier_filter(samples_y, grad_mags_y, lambda_param)
                        
                        idx = 0
                        for y in range(start_y_left, end_y_left):
                            filtered_y[x][y][z] = filtered_vals[idx]
                            idx += 1
                        for y in range(start_y_right, end_y_right):
                            filtered_y[x][y][z] = filtered_vals[idx]
                            idx += 1

    # Z-axis filtering
    for b_z in range(0, dims[2], block_size):
        for b_x in range(0, dims[0], block_size):
            for b_y in range(0, dims[1], block_size):
                boundary_z = b_z + block_size
                if boundary_z >= dims[2]:
                    continue
                    
                half_block = block_size // 2
                start_z_left = max(b_z, boundary_z - half_block)
                end_z_left = boundary_z
                start_z_right = boundary_z
                end_z_right = min(boundary_z + half_block, dims[2])
                
                for x in range(b_x, min(b_x + block_size, dims[0])):
                    for y in range(b_y, min(b_y + block_size, dims[1])):
                        samples_z = []
                        grad_mags_z = []
                        
                        for z in range(start_z_left, end_z_left):
                            samples_z.append(volume[x][y][z])
                            grad_mags_z.append(grad_mag[x][y][z])
                        for z in range(start_z_right, end_z_right):
                            samples_z.append(volume[x][y][z])
                            grad_mags_z.append(grad_mag[x][y][z])
                            
                        filtered_vals = rational_bezier_filter(samples_z, grad_mags_z, lambda_param)
                        
                        idx = 0
                        for z in range(start_z_left, end_z_left):
                            filtered_z[x][y][z] = filtered_vals[idx]
                            idx += 1
                        for z in range(start_z_right, end_z_right):
                            filtered_z[x][y][z] = filtered_vals[idx]
                            idx += 1

    # Step 8: Final intensity is the average of the three filtered values
    for x in range(dims[0]):
        for y in range(dims[1]):
            for z in range(dims[2]):
                filtered_volume[x][y][z] = (filtered_x[x][y][z] + filtered_y[x][y][z] + filtered_z[x][y][z]) / 3.0
                
    return filtered_volume
```

### Edge Cases and Pitfalls
- **Precision**: The exponential function for weights can lead to very small or very large numbers if gradient magnitudes are extreme. However, the source does not specify normalization or clamping [S1, S2, unofficial].
- **Boundaries**: The filter is applied only at block boundaries. Voxels in the interior of blocks are not modified by this specific filter step [S1, S2, unofficial].
- **Degenerate Input**: If a block boundary is at the edge of the volume, the adjacent block does not exist, and the filter is not applied [S1, S2, unofficial].
- **Feature Preservation**: The filter relies on gradient magnitude. In homogeneous regions, gradients are low, resulting in high weights and subtle filtering. In high-gradient regions, weights are lower, allowing more smoothing. This is critical because volume rendering shading emphasizes artifacts in high-gradient regions [S1, S2, unofficial].

## Subtopics

### Deferred Filtering Architecture
A rendering architecture that integrates decompression, deblocking, and rendering on the GPU, supporting adaptive multi-resolution data loading and visibility culling [S1, unofficial].

### GPU-Accelerated Implementation
An optimized implementation of the filter on graphics hardware to achieve interactive frame rates for massive datasets [S1, S2, unofficial].

## Operating characteristics
- **Time Complexity**: The filter is computationally intensive due to the calculation of gradient magnitudes and the rational Bézier curve projection for each boundary line [S1, unofficial].
- **Memory Complexity**: Requires storing the gradient magnitude field and temporary filtered values for each axis pass [S1, S2, unofficial].
- **Performance**: Achieves interactive frame rates on standard graphics PCs for massive static and dynamic datasets (e.g., 512^4 time-varying simulations) [S1, unofficial].
- **Strengths**:
    - Works without knowledge of the compression technique [S1, S2, unofficial].
    - Does not require storing additional information in the compressed data [S1, S2, unofficial].
    - Preserves local features better than simple averaging or standard Bézier filtering [S1, S2, unofficial].
    - Outperforms compressed-only rendering and lower-resolution downsampling in visual quality [S1, unofficial].
- **Limitations**:
    - Computationally intensive; requires future optimization for speed [S1, unofficial].
    - Only effective at block boundaries; does not address intra-block artifacts [S1, S2, unofficial].
- **When to Use**: When using block-based compression for volume rendering and blocking artifacts are visible, especially in high-gradient regions [S1, S2, unofficial].
- **When Not to Use**: When computational resources are extremely limited and real-time performance is critical without optimization, or when the data is not block-compressed [S1, unofficial].

## Background
The Rational Bézier Deblocking Filter was proposed by F. Marton, José A. Iglesias Guitián, Jose Díaz, and Enrico Gobbetti in the paper "Real-time Deblocked GPU Rendering of Compressed Volumes" [S1, S2, unofficial]. It was presented at the Eurographics Symposium on Volume Graphics and Visualization (VMV) in 2014 [S1, S2, unofficial]. The method extends state-of-the-art results in compression, filtering, and GPU volume rendering, specifically addressing the limitations of previous deblocking methods that were not suitable for volume rendering's semi-transparent shading [S1, S2, unofficial].

## Sources
[S1] Real-time Deblocked GPU Rendering of Compressed Volumes, https://doi.org/10.2312/vmv.20141290, fetched 2026-10-26 (unofficial)
[S2] PDF Real-time deblocked GPU rendering of compressed volumes, https://www.crs4.it/vic/data/papers/vmv2014-volume-deblocking.pdf, fetched 2026-10-27 (unofficial)
