# Dense Grid Encoding

Also called: Dense Grid Parametric Encoding, Parametric Grid Encoding

## Overview
Dense Grid Encoding is a parametric input encoding method that maps input coordinates to multi-dimensional feature vectors by storing trainable parameters in a dense, regular grid structure [S1, unofficial][S2]. It solves the problem of efficiently representing high-frequency scene details in neural radiance fields (NeRF) by trading a larger memory footprint for reduced computational cost during training [S1, unofficial][S2]. Unlike fixed frequency encodings, it allows the network to learn spatially varying features, but unlike sparse or hash-based variants, it allocates parameters uniformly across the entire domain, which can be wasteful in empty space [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   Input: A coordinate vector $\mathbf{x} \in \mathbb{R}^d$ (typically $d=3$ for 3D space).
*   Output: A feature vector $\mathbf{f} \in \mathbb{R}^k$ (where $k$ is the feature dimension).

**Data Structures:**
*   **Grid:** A dense $d$-dimensional array of shape $N_1 \times N_2 \times \dots \times N_d$.
*   **Feature Vectors:** Each cell in the grid stores a trainable vector of size $k$.
*   **Total Parameters:** $P = (\prod_{i=1}^d N_i) \times k$.

**Steps:**
1.  **Normalization:** Map the input coordinate $\mathbf{x}$ to the grid's coordinate space, typically normalizing it to the range $[0, 1]$ or $[-1, 1]$ depending on the scene bounds. Let the normalized coordinate be $\mathbf{x}'$.
2.  **Grid Indexing:** Determine the integer grid indices for the corner cells surrounding $\mathbf{x}'$. For a 3D grid with resolution $N$, the indices are computed as:
    $$ i = \lfloor x' \cdot N \rfloor $$
    where $i \in \{0, \dots, N-1\}$. This yields 8 corner indices $(i_0, i_1, i_2)$ for the voxel containing $\mathbf{x}'$.
3.  **Interpolation Weights:** Compute the fractional part of the normalized coordinate to determine interpolation weights. For each dimension $j$:
    $$ w_j = x'_j \cdot N - i_j $$
    The weights for the lower and upper corners are $1-w_j$ and $w_j$ respectively.
4.  **Feature Lookup:** Retrieve the $k$-dimensional feature vectors $\mathbf{v}_{c}$ for each of the 8 corner cells $c \in \{0, \dots, 7\}$ from the grid.
5.  **Trilinear Interpolation:** Compute the final feature vector $\mathbf{f}$ by trilinearly interpolating the 8 corner vectors using the weights:
    $$ \mathbf{f} = \sum_{c=0}^{7} \alpha_c \mathbf{v}_c $$
    where $\alpha_c$ is the product of the appropriate lower/upper weights for each dimension corresponding to corner $c$.
6.  **Output:** Return $\mathbf{f}$.

**Parameters:**
*   **Grid Resolution ($N$):** Determines the granularity. Typical values range from $16^3$ to $256^3$ depending on scene complexity and memory constraints [S1, unofficial][S2].
*   **Feature Dimension ($k$):** Size of the feature vector per cell. Common values are 2, 4, or 16 [S1, unofficial][S2].

**Edge Cases and Pitfalls:**
*   **Empty Space Waste:** The method allocates features to empty space equally to surface areas. The parameter count grows as $O(N^d)$, while the visible surface area grows as $O(N^{d-1})$ [S1, unofficial][S2].
*   **Precision:** Floating-point precision in coordinate normalization and index calculation must be handled carefully to avoid boundary artifacts [G].
*   **Boundary Conditions:** Coordinates outside the grid bounds must be clamped or handled via padding to prevent out-of-bounds memory access [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dense_grid_encode(x, grid, resolution, feature_dim):
    """
    x: input coordinate vector of shape (3,)
    grid: trainable parameter tensor of shape (resolution, resolution, resolution, feature_dim)
    resolution: integer grid size N
    feature_dim: integer k
    """
    # Normalize x to [0, 1] range assuming x is in [-1, 1]
    x_norm = (x + 1.0) / 2.0
    
    # Compute grid indices for the 8 corners
    # Using floor to get integer indices
    indices = (x_norm * resolution).long()
    
    # Clamp indices to valid range [0, resolution-1]
    indices = torch.clamp(indices, 0, resolution - 1)
    
    # Compute fractional weights
    frac = x_norm * resolution - indices.float()
    
    # Weights for lower (0) and upper (1) corners in each dimension
    w0 = 1.0 - frac
    w1 = frac
    
    # Initialize output feature vector
    f = torch.zeros(feature_dim, device=x.device)
    
    # Iterate over 8 corners
    for i in range(2):
        for j in range(2):
            for k in range(2):
                # Corner index
                idx_i = indices[0] + i
                idx_j = indices[1] + j
                idx_k = indices[2] + k
                
                # Weight for this corner
                weight = w0[0]**(1-i) * w1[0]**i * \
                         w0[1]**(1-j) * w1[1]**j * \
                         w0[2]**(1-k) * w1[2]**k
                 
                # Lookup feature vector
                v = grid[idx_i, idx_j, idx_k]
                
                # Accumulate
                f += weight * v
                
    return f
```

## Subtopics

### Grid-based MS Module
A variant where the positional encoding $\gamma(\mathbf{p})$ in a hybrid multi-space module is replaced by features queried from a smaller dense grid $V_{vis}$ [S3]. The query function $Q(V_{vis}, \mathbf{p})$ replaces the standard positional encoding, allowing the multi-space scheme to be more position-dependent and efficient [S3].

## Operating characteristics
*   **Time Complexity:** Lookup and interpolation are $O(1)$ with respect to grid size, depending only on the number of corners (8 for 3D trilinear) and feature dimension $k$.
*   **Memory Complexity:** $O(N^d \cdot k)$. For a $128^3$ grid with 16-dim features, this is approximately 33.6 million parameters [S1, unofficial][S2].
*   **Update Efficiency:** During back-propagation, only the 8 grid points affected by a sample are updated, regardless of the MLP size [S1, unofficial][S2]. This allows for faster convergence compared to fixed encodings where all weights are updated [S1, unofficial][S2].
*   **Strengths:** Efficient gradient updates; high approximation quality for bounded domains; simpler implementation than hash-based methods [S1, unofficial][S2].
*   **Limitations:** Wasteful in empty space; single-scale (unless combined with multi-resolution variants); memory usage grows cubically with resolution [S1, unofficial][S2].
*   **When to Use:** When memory is sufficient and the scene is bounded; when fast training convergence is prioritized over memory efficiency [S1, unofficial][S2].
*   **When Not to Use:** When the scene has large empty volumes or when memory is constrained [S1, unofficial][S2].

## Background
Dense Grid Encoding is a type of parametric encoding introduced in the context of neural implicit representations [S1, unofficial][S2]. It builds on earlier work using grids for 3D reconstruction [Chabra et al. 2020; Jiang et al. 2020; Liu et al. 2020; Mehta et al. 2021; Peng et al. 2020a; Sun et al. 2021; Tang et al. 2018; Yu et al. 2021a] and was popularized by its use in Instant-NGP [S1, unofficial][S2]. The method trades memory for computational efficiency by localizing gradient updates to a small number of grid cells [S1, unofficial][S2].

## Sources
[S1] Nvidia – learning representations of positions in a 3D scene [pdf], https://nvlabs.github.io/instant-ngp/assets/mueller2022instant.pdf, fetched 2026-10-26 (unofficial)
[S2] Instant Neural Graphics Primitives with a Multiresolution ..., https://www.cs.jhu.edu/~misha/ReadingSeminar/Papers/Mueller22.pdf, fetched 2026-10-28
[S3] MS-NeRF: Multi-Space Neural Radiance Fields, https://doi.org/10.1109/tpami.2025.3540074, fetched 2026-10-26
