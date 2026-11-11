# Lagrangian Hashing

Also called: LagHash

## Overview
Lagrangian Hashing is a hybrid neural field representation that combines the memory efficiency of Eulerian hierarchical hash grids (specifically InstantNGP) with the adaptive density control of Lagrangian point-based representations (such as 3D Gaussian Splatting) [S1]. It solves the problem of high-frequency detail reconstruction in neural rendering by allowing feature points to move freely in space within hash buckets, rather than being fixed to uniform grid vertices [S1]. Unlike its sibling [HashGrid Encoding](3866-hashgrid-encoding.md), which uses static feature vectors per bucket, Lagrangian Hashing stores a mixture of Gaussians per bucket, enabling the representation to allocate its budget adaptively to regions requiring more detail [S1].

## How it works
The input is a continuous coordinate vector $\mathbf{x} \in \mathbb{R}^D$ (typically 3D spatial position) [S1]. The output is a field value $\mathcal{F}(\mathbf{x}) \in \mathbb{R}^F$ computed by passing concatenated per-level features through a shared decoder MLP [S1].

### Data Structures
1. **Feature Tensor ($\mathbf{F}_l$)**: For each level $l$, a tensor storing per-bucket features $\mathbf{F}_l[v]$ [S1]. If grid resolution $N_l < B$ (bucket capacity), shape is $\mathbb{R}^{N_l \times F}$ [S1]. If $N_l \ge B$, shape is $\mathbb{R}^{B \times F}$ [S1].
2. **Hash Function ($H_l$)**: Maps input coordinate $\mathbf{x}$ to indices in $\mathbf{F}_l$ [S1]. If $N_l < B$, $H_l$ is an injective map: $H_l: \mathbb{R}^D \rightarrow [1, N_l - 1]$ [S1]. If $N_l \ge B$, $H_l$ is a hash function: $H_l: \mathbb{R}^D \rightarrow [0, B - 1]$ [S1].
3. **Lagrangian Points**: Inside each hash bucket $v$ at level $l$, a small set of points is stored instead of a single static feature vector [S1]. Each point $p$ has a learnable position $\mathbf{c}_p$ and a learnable feature vector $\mathbf{f}_p$ [S1]. Each point is equipped with an isotropic Gaussian field of influence where the standard deviation $\sigma_l$ is implicitly proportional to the grid resolution $N_l$ and is non-trainable [S1].

### Parameters
* $L$: Number of levels of detail in the stack of regular grids [S1].
* $N_{min}$: Minimum resolution base for the geometric progression of grid resolutions [S1].
* $b$: Base of the geometric progression for grid resolution scaling [S1].
* $B$: Bucket size for the hash table, used when $N_l \ge B$ [S1].
* $\tilde{L}$: Number of high-resolution levels that use the Lagrangian (point-based) representation [S1]. The remaining $L - \tilde{L}$ shallow levels use standard Eulerian static features [S1].

### Algorithm Steps
1. **Grid Resolution Calculation**: For each level $l \in \{1, \dots, L\}$, calculate the grid resolution: $N_l = N_{min} \cdot b^l$ [S1].
2. **Index Retrieval**: For a query point $\mathbf{x}$, determine the $2^D$ surrounding grid corners (voxel vertices) for each level $l$ [S1]. Let $\mathcal{H}_l(\mathbf{x})$ be the set of indices $v$ in the hash table $\mathbf{F}_l$ corresponding to these corners [S1].
3. **Per-Level Feature Interpolation**: Compute the feature vector $\mathbf{f}_l(\mathbf{x})$ for level $l$ by interpolating the features from the $2^D$ corners [S1]. The interpolation weight $\alpha_v$ is determined by the trilinear interpolation of $\mathbf{x}$ within the grid cell at level $l$ [S1].
   $$\mathbf{f}_l(\mathbf{x}) = \sum_{v \in \mathcal{H}_l(\mathbf{x})} \alpha_v \cdot \mathbf{F}_l[v](\mathbf{x})$$
   Here, $\mathbf{F}_l[v](\mathbf{x})$ denotes the feature value retrieved from bucket $v$ at position $\mathbf{x}$ [S1].
   * **Eulerian Levels ($l < L - \tilde{L}$)**: $\mathbf{F}_l[v](\mathbf{x})$ is a static feature vector stored in the bucket [S1].
   * **Lagrangian Levels ($l \ge L - \tilde{L}$)**: $\mathbf{F}_l[v](\mathbf{x})$ is computed as a weighted sum of the isotropic Gaussians associated with the points stored in bucket $v$ [S1]. The weight of each point $p$ in bucket $v$ is determined by the Gaussian kernel $G(\mathbf{x}; \mathbf{c}_p, \sigma_l)$ centered at $\mathbf{c}_p$ with standard deviation $\sigma_l$ [S1].
4. **Concatenation**: Concatenate the feature vectors from all levels: $\mathbf{f}_{total}(\mathbf{x}) = \mathbf{f}_1(\mathbf{x}) \oplus \mathbf{f}_2(\mathbf{x}) \oplus \dots \oplus \mathbf{f}_L(\mathbf{x})$ [S1].
5. **Decoding**: Pass $\mathbf{f}_{total}(\mathbf{x})$ through a shared decoder MLP with parameters $\boldsymbol{\theta}$ to produce the final output: $\mathcal{F}(\mathbf{x}) = \text{MLP}(\mathbf{f}_{total}(\mathbf{x}); \boldsymbol{\theta})$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def lagrangian_hashing_forward(x, levels, hash_tables, decoder_mlp, N_min, b, B, sigma_scale, F_dim):
    """
    x: Query coordinate vector of shape (D,)
    levels: List of level indices [1, ..., L]
    hash_tables: List of feature tensors F_l for each level
    decoder_mlp: Neural network decoder
    N_min: Minimum resolution base
    b: Resolution scaling base
    B: Bucket size
    sigma_scale: Scaling factor for Gaussian standard deviation relative to grid resolution
    F_dim: Output feature dimension
    Returns: Output feature vector
    """
    concatenated_features = []
    
    for l in levels:
        N_l = N_min * (b ** l)
        corner_indices, alpha_weights = compute_grid_corners_and_weights(x, N_l)
        level_feature = [0.0] * F_dim
        
        for v, alpha_v in zip(corner_indices, alpha_weights):
            bucket_feature = get_bucket_feature(hash_tables[l], v, x, N_l, sigma_scale)
            level_feature = vector_add(level_feature, vector_scale(bucket_feature, alpha_v))
            
        concatenated_features.extend(level_feature)
        
    return decoder_mlp(concatenated_features)

def compute_grid_corners_and_weights(x, N_l):
    D = len(x)
    indices = []
    weights = []
    for mask in range(1 << D):
        corner = []
        w = 1.0
        for d in range(D):
            if (mask >> d) & 1:
                idx = math.floor(x[d] * N_l)
                frac = x[d] * N_l - idx
                corner.append(idx)
                w *= frac
            else:
                idx = math.floor(x[d] * N_l)
                frac = 1.0 - (x[d] * N_l - idx)
                corner.append(idx)
                w *= frac
        indices.append(tuple(corner))
        weights.append(w)
    return indices, weights

def get_bucket_feature(hash_table, v, x, N_l, sigma_scale):
    points = hash_table[v].points
    if points is None:
        return hash_table[v].static_feature
    sigma = sigma_scale * N_l
    weighted_sum = [0.0] * len(points[0].feature)
    total_weight = 0.0
    for p in points:
        dist_sq = sum((x[d] - p.center[d]) ** 2 for d in range(len(x)))
        weight = math.exp(-dist_sq / (2 * sigma ** 2))
        weighted_sum = vector_add(weighted_sum, vector_scale(p.feature, weight))
        total_weight += weight
    if total_weight > 0:
        return vector_scale(weighted_sum, 1.0 / total_weight)
    return [0.0] * len(points[0].feature)

def vector_add(a, b):
    return [a[i] + b[i] for i in range(len(a))]

def vector_scale(v, s):
    return [v[i] * s for i in range(len(v))]
```

### Edge Cases and Pitfalls
* **Initialization**: Unlike 3D Gaussian Splatting, Lagrangian Hashing does not require careful initialization from depth, COLMAP, or LiDAR data [S1]. The points are initialized within the hash buckets and guided by a loss function [S1].
* **Gaussian Standard Deviation**: The standard deviation of the isotropic Gaussians is non-trainable and implicitly proportional to the grid resolution $N_l$ [S1]. This ensures that the field of influence scales appropriately with the level of detail [S1].
* **Hash Collisions**: When $N_l \ge B$, the hash function $H_l$ maps to $[0, B-1]$, potentially causing collisions [S1]. The method relies on the hash table implementation (e.g., chaining or open addressing) to handle these, similar to InstantNGP [S1].

## Subtopics

### [Lagrangian Per-Bucket Feature Representation](3527-lagrangian-per-bucket-feature-representation.md)
This method defines the feature retrieval mechanism for the deepest levels of a hierarchical hash grid in Lagrangian Hashing. Instead of storing static feature vectors in hash buckets, it computes features as a weighted sum of isotropic Gaussians centered at learnable means. This allows for continuous, position-dependent feature retrieval, distinguishing it from the static bucket lookups of InstantNGP.

## Operating characteristics
* **Memory Efficiency**: The method achieves high-quality novel view synthesis with fewer parameters compared to baseline methods [S1]. For example, it achieves a PNSR of 35.60 dB with 6.68M parameters, compared to a baseline PNSR of 27.31 dB with the same parameter count [S1].
* **Performance**: It combines the fast training speed of Eulerian grids (InstantNGP) with the compact representation of point-based methods (3DGS) [S1].
* **Strengths**: Adaptive allocation of representation budget; no need for complex initialization or pruning heuristics; memory efficient [S1].
* **Limitations**: Requires a loss function to guide the movement of Gaussians to regions needing more representation [S1]. The use of isotropic Gaussians (vs. anisotropic in 3DGS) may limit the ability to capture highly anisotropic features [S1].

## Background
Lagrangian Hashing was introduced in 2024 by researchers from Simon Fraser University, University of Trento, University of Toronto, University of British Columbia, and Google DeepMind [S1]. It builds upon the hierarchical hash tables of InstantNGP and incorporates a point-based representation similar to 3D Gaussian Splatting [S1]. The method addresses the limitations of both Eulerian grids (fixed resolution) and Lagrangian point clouds (complex indexing) by hybridizing them within a hash table structure [S1].

## Sources
[S1] Lagrangian Hashing for Compressed Neural Field Representations, https://doi.org/10.1007/978-3-031-73383-3_11, fetched 2026-10-26
