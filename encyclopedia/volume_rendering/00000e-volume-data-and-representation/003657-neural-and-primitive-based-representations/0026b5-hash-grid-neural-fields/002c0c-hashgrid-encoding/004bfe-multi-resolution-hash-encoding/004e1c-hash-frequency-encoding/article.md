# Hash-Frequency Encoding
Also called: Composited Encoding, HE+FE

## Overview
Hash-Frequency Encoding (HE+FE) is a hybrid feature representation for neural radiance fields that combines the memory efficiency of multi-resolution hash encoding with the high-frequency approximation capabilities of sinusoidal frequency encoding. It solves the problem of severe performance degradation and artifact generation (floaters) in sparse input scenarios, where standard hash-only encoders fail to capture fine details. Unlike its sibling [Standard Multi-resolution Hash Encoding](13152-standard-multi-resolution-hash-encoding.md), which relies solely on hash lookups, HE+FE augments the hash features with frequency levels to stabilize reconstruction on sparse views, while training significantly faster than frequency-only alternatives like [MIP-NeRF 360](13152-standard-multi-resolution-hash-encoding.md).

## How it works
**Inputs and Outputs:**
- Input: 3D spatial coordinates $\mathbf{x} \in \mathbb{R}^3$.
- Output: Concatenated feature vector $\mathbf{f} \in \mathbb{R}^{D_{hash} + D_{freq}}$, where $D_{hash}$ is the hash grid dimension and $D_{freq}$ is the frequency encoding dimension.

**Data Structures:**
1.  **Hash Grids:** A set of $L_{hash}$ resolution levels. Each level $l$ consists of a hash table mapping hashed coordinate indices to feature vectors of size $d$. The hash function maps 3D coordinates to table indices.
2.  **Frequency Levels:** A set of $L_{freq}$ frequency levels. Each level $l$ defines a set of sinusoidal basis functions.

**Parameters:**
-   $L_{hash}$: Number of hash grid levels. Default: 16 [S1].
-   $L_{freq}$: Number of frequency levels. Swept values: 2, 4, 8, 16. Optimal: 8 [S1].
-   $d$: Feature dimension per hash level (standard Instant-NGP value, typically 2 or 4, though not explicitly specified in [S1], it is implied by the hash grid structure).

**Algorithm:**
The encoding process computes features from both components and concatenates them.

1.  **Hash Encoding Step:**
    For each hash level $l \in \{0, \dots, L_{hash}-1\}$:
    a.  Compute the voxel size $s_l$ for level $l$.
    b.  Compute the voxel index $\mathbf{i}_l = \lfloor \mathbf{x} / s_l \rfloor$.
    c.  Compute the hash index $h_l = \text{Hash}(\mathbf{i}_l) \pmod{N_l}$, where $N_l$ is the table size for level $l$.
    d.  Retrieve the feature vector $\mathbf{v}_l$ from the hash table at index $h_l$.
    e.  Accumulate or store $\mathbf{v}_l$.

2.  **Frequency Encoding Step:**
    For each frequency level $l \in \{0, \dots, L_{freq}-1\}$:
    a.  Compute the frequency multiplier $\omega_l$. (Note: Specific formula for $\omega_l$ is not given in [S1], but standard sinusoidal encoding uses powers of 2 or linear spacing).
    b.  Compute the sinusoidal features for each dimension $k \in \{1, 2, 3\}$ of $\mathbf{x}$:
        $$ \sin(2\pi \omega_l x_k), \cos(2\pi \omega_l x_k) $$
    c.  Concatenate these values into the frequency feature vector $\mathbf{f}_{freq}$.

3.  **Concatenation:**
    $$ \mathbf{f} = [\mathbf{v}_0, \dots, \mathbf{v}_{L_{hash}-1}, \mathbf{f}_{freq}] $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hash_frequency_encoding(x, hash_grids, freq_levels):
    """
    x: Tensor of shape (..., 3) containing 3D coordinates.
    hash_grids: List of L_hash tensors, each of shape (N_l, d).
    freq_levels: List of L_freq frequency multipliers.
    Returns: Concatenated feature tensor.
    """
    hash_features = []
    
    # Hash Encoding
    for l in range(len(hash_grids)):
        grid = hash_grids[l]
        # Compute voxel indices based on resolution level l
        # Note: Exact voxel size calculation depends on the specific hash grid implementation
        # Assuming standard Instant-NGP style scaling
        voxel_size = compute_voxel_size(l) 
        voxel_indices = torch.floor(x / voxel_size).long()
        
        # Hash the indices to get table indices
        # Note: The specific hash function is implementation-dependent
        table_indices = hash_function(voxel_indices) % grid.shape[0]
        
        # Gather features
        # Expand indices to match feature dimension
        expanded_indices = table_indices.unsqueeze(-1).expand(-1, -1, grid.shape[1])
        features = torch.gather(grid, 0, expanded_indices)
        hash_features.append(features)
        
    # Frequency Encoding
    freq_features = []
    for omega in freq_levels:
        # Compute sinusoidal features for each dimension
        # x shape: (..., 3)
        scaled_x = x * omega * 2 * 3.14159265359
        sin_features = torch.sin(scaled_x)
        cos_features = torch.cos(scaled_x)
        freq_features.append(sin_features)
        freq_features.append(cos_features)
        
    # Concatenate all features
    # hash_features is a list of tensors, freq_features is a list of tensors
    all_features = hash_features + freq_features
    return torch.cat(all_features, dim=-1)
```

**Edge Cases and Pitfalls:**
-   **Sparse Inputs:** Standard hash encoding degrades heavily with sparse inputs [S1]. HE+FE mitigates this by adding frequency information.
-   **Frequency Level Selection:** Too few frequency levels may not capture high-frequency details; too many may increase memory and computation without benefit. Optimal is 8 [S1].
-   **Training Time:** While slower than pure hash encoding, it is significantly faster than pure frequency encoding (5 mins vs 2 hours) [S1].

## Subtopics

### [Vanishing Point Aligned Frequency Encoding](14036-vanishing-point-aligned-frequency-encoding.md)
Aligns sinusoidal positional encoding with scene structures using estimated vanishing points, ensuring points on the same spatial line share the same mapping of specific encoding levels. Chosen for axis-aligned structures in 360° images.

## Operating characteristics
-   **Performance:** HE + FE8 provides stable PSNR, SSIM, and LPIPS on synthetic datasets (Classroom, Lone Monk) with sparse (2x2x2) and normal (3x3x2) camera setups [S1].
-   **Comparison:** Outperforms Nerfacto (hash-only) on sparse inputs, avoiding degradation and floaters [S1]. Comparable to MIP-NeRF 360 (frequency-only) in quality but trains much faster (5 minutes vs 2 hours on a single NVIDIA A40) [S1].
-   **Limitations:** May not be optimal for normal camera setups compared to sparse ones [S1].

## Background
Originated in the paper "Vanishing Point Aided Hash-Frequency Encoding for Neural Radiance Fields (NeRF) from Sparse 360° Input" [S1]. It was developed to address the limitations of hash-only encoders (like Nerfacto) when dealing with sparse 360° input data, combining the strengths of hash encoding and frequency encoding.

## Sources
[S1] Vanishing Point Aided Hash-Frequency Encoding for Neural Radiance Fields (NeRF) from Sparse 360°Input, https://doi.org/10.1109/ismar59233.2023.00131, fetched 2026-10-27
