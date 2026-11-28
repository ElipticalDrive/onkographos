# HashNeRF

Also called: Instant Neural Graphics Primitives, Instant Neural Graphics Primitives with a Multiresolution Hash Encoding

## Overview
HashNeRF is a neural radiance field (NeRF) implementation that achieves significantly faster training and rendering times by replacing traditional positional encoding with a trainable multi-resolution hash encoding [S1][S2, unofficial]. It solves the problem of high computational cost in standard NeRFs by mapping 3D coordinates to feature vectors in a compact hash grid, allowing the model to converge in fewer iterations [S1][S2, unofficial]. Unlike standard NeRFs which use fixed sinusoidal positional encoding, HashNeRF uses a shallow Multi-Layer Perceptron (MLP) decoder, making it a generic primitive for various neural graphics tasks [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D spatial coordinate $\mathbf{x} = (x, y, z)$ and viewing direction $\mathbf{v}$ [S2, unofficial].
*   **Output:** Log-space density $\sigma$ and color $\mathbf{c}$ [S2, unofficial].

**Data Structures:**
*   **Hash Tables:** $L$ hash tables (grids), indexed by resolution level $l \in \{0, \dots, L-1\}$ [S1, G].
*   **Table Size:** Each table $l$ has $T_l$ entries, limited by $T$ via modulo operation [S1].
*   **Feature Vector:** Each entry stores a feature vector of dimension $F$ [S1, G].
*   **Total Feature Dimension:** $L \times F$ [S1, G].
*   **MLP:** A shallow Multi-Layer Perceptron (MLP) with fully connected layers. It consists of a Density MLP and a Color MLP [S1][S2, unofficial].

**Algorithm Steps:**
1.  **Coordinate Scaling:** For each level $l$, scale the input coordinate:
    $$ \mathbf{x}_l = 2^l \mathbf{x} $$
2.  **Voxel Indexing:** Compute integer voxel indices $\mathbf{v}_l = (v_x, v_y, v_z)$ by flooring the scaled coordinates:
    $$ v_i = \lfloor (\mathbf{x}_l)_i \rfloor, \quad i \in \{x, y, z\} $$
3.  **Fractional Part:** Compute the fractional part $\mathbf{u}_l = (u_x, u_y, u_z)$ for interpolation weights:
    $$ u_i = (\mathbf{x}_l)_i - v_i $$
4.  **Hashing:** Map voxel indices to table indices using a hash function $H_l$. A common implementation uses a sum or XOR-based hash with modulo $T_l$ [S1]:
    $$ i_l = H_l(v_x, v_y, v_z) \pmod{T_l} $$
    The specific collision resolution strategy (e.g., chaining or open addressing) is implementation-dependent [G].
5.  **Feature Retrieval and Interpolation:** Retrieve feature vectors $\mathbf{f}_{l, \mathbf{v}'}$ from the 8 neighboring voxels $\mathbf{v}'$ of $\mathbf{v}_l$. Interpolate to get $\hat{\mathbf{f}}_l$:
    $$ \hat{\mathbf{f}}_l = \sum_{\mathbf{v}' \in \text{neighbors}(\mathbf{v}_l)} w(\mathbf{v}') \cdot \mathbf{f}_{l, \mathbf{v}'} $$
    where the weight $w(\mathbf{v}')$ for neighbor $\mathbf{v}' = (v'_x, v'_y, v'_z)$ is:
    $$ w(\mathbf{v}') = \prod_{i \in \{x,y,z\}} \begin{cases} u_i & \text{if } v'_i = v_i + 1 \\ 1 - u_i & \text{if } v'_i = v_i \end{cases} $$
6.  **Feature Concatenation:** Concatenate features from all $L$ levels to form $\mathbf{F}$:
    $$ \mathbf{F} = [\hat{\mathbf{f}}_0; \hat{\mathbf{f}}_1; \dots; \hat{\mathbf{f}}_{L-1}] $$
7.  **Density Prediction:** Pass $\mathbf{F}$ through the Density MLP to output log-space density $\sigma$ [S2, unofficial].
8.  **Color Prediction:** Pass $\mathbf{F}$ and viewing direction $\mathbf{v}$ through the Color MLP to output color $\mathbf{c}$ [S2, unofficial].

**Parameters:**
*   **Levels ($L$):** 16 [S1].
*   **Hardware:** Typically implemented using the Tiny-CUDA-NN library on NVIDIA GeForce RTX 3090 Ti GPUs [S1][S2, unofficial].
*   **Training Iterations:** Evaluated at 500 and 1,000 iterations [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def hash_function(v_x, v_y, v_z, table_size):
    """XOR-based hash mapping voxel indices to table indices."""
    h = v_x ^ (v_y << 16) ^ (v_z << 32)
    h = h & 0xFFFFFFFF
    return h % table_size

def trilinear_interpolation(features, u_x, u_y, u_z):
    """Interpolate 8 feature vectors using trilinear weights."""
    w000 = (1 - u_x) * (1 - u_y) * (1 - u_z)
    w100 = u_x * (1 - u_y) * (1 - u_z)
    w010 = (1 - u_x) * u_y * (1 - u_z)
    w110 = u_x * u_y * (1 - u_z)
    w001 = (1 - u_x) * (1 - u_y) * u_z
    w101 = u_x * (1 - u_y) * u_z
    w011 = (1 - u_x) * u_y * u_z
    w111 = u_x * u_y * u_z
    weights = [w000, w100, w010, w110, w001, w101, w011, w111]
    dim = len(features[0])
    result = [0.0] * dim
    for i, w in enumerate(weights):
        for j in range(dim):
            result[j] += w * features[i][j]
    return result

def hashnerf_encode(x, y, z, hash_tables, num_levels=16):
    """Multi-resolution hash encoding for a single 3D point."""
    concatenated_features = []
    for l in range(num_levels):
        scale = 2 ** l
        x_scaled = x * scale
        y_scaled = y * scale
        z_scaled = z * scale
        
        v_x = math.floor(x_scaled)
        v_y = math.floor(y_scaled)
        v_z = math.floor(z_scaled)
        
        u_x = x_scaled - v_x
        u_y = y_scaled - v_y
        u_z = z_scaled - v_z
        
        table = hash_tables[l]
        table_size = len(table)
        
        neighbors = []
        for dx in [0, 1]:
            for dy in [0, 1]:
                for dz in [0, 1]:
                    nx = v_x + dx
                    ny = v_y + dy
                    nz = v_z + dz
                    idx = hash_function(nx, ny, nz, table_size)
                    neighbors.append(table[idx])
                    
        interpolated = trilinear_interpolation(neighbors, u_x, u_y, u_z)
        concatenated_features.extend(interpolated)
    return concatenated_features

def hashnerf_forward(x, y, z, direction, hash_tables, density_mlp, color_mlp, num_levels=16):
    """Full HashNeRF forward pass."""
    features = hashnerf_encode(x, y, z, hash_tables, num_levels)
    density = density_mlp(features)
    color_input = features + direction
    color = color_mlp(color_input)
    return density, color
```

## Subtopics

### [HashNeRF Architectural Variants](8688-hashnerf-architectural-variants.md)
Specific modifications to the shallow MLP decoder (e.g., convolutional layers, residual connections) to optimize the trade-off between rendering accuracy and training time.

### [HollowNeRF](10709-hollownerf.md)
A compression method that sparsifies the feature grid during training using a trainable 3D saliency mask to guide feature pruning.

### [Zip-NeRF Hash Encoding with Gaussian Expectation](11492-zip-nerf-hash-encoding-with-gaussian-expectation.md)
Replaces large MLPs with multi-scale hash grids and uses Gaussian expectation approximation to prevent aliasing.

## Operating characteristics
*   **Time Complexity:** Training converges in fewer iterations than standard NeRFs due to the generic, trainable encoding and shallow network [S2, unofficial].
*   **Memory Complexity:** Memory utilization is optimized by the hash table structure, storing only necessary features [S1].
*   **Performance:** Achieves excellent results with minimal learning time [S1].
*   **Strengths:** Significant reduction in learning time compared to traditional position encoding methods [S1]. Fast convergence [S2, unofficial].
*   **Limitations:** Performance depends on the hash table size $T$ and the number of levels $L$ [S1].
*   **When to use:** When fast training and rendering of NeRFs are required [S1][S2, unofficial].

## Background
HashNeRF, also known as Instant Neural Graphics Primitives with a Multiresolution Hash Encoding, was proposed by Thomas Müller, Alex Evans, Christoph Schied, and Alexander Keller in 2022 [S2, unofficial]. It was designed to radically reduce training times for NeRF models [S2, unofficial]. The method is implemented using the Tiny-CUDA-NN library based on CUDA and C++ [S1]. It has applications in Gigapixel approximation, generating signed distance functions, and Neural Radiance Caching [S2, unofficial].

## Sources
[S1] Performance Evaluation and Optimization of 3D Gaussian Splatting in Indoor Scene Generation and Rendering, https://doi.org/10.3390/ijgi14010021, fetched 2026-10-26
[S2] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
