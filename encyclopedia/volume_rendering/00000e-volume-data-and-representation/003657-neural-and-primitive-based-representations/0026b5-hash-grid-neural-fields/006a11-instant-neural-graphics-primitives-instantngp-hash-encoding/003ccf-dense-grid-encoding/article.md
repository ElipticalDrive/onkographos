# Dense Grid Encoding

Also called: Dense Grid Parametric Encoding, Parametric Grid Encoding

## Overview
Dense Grid Encoding is a parametric input encoding method that maps input coordinates to feature vectors by storing trainable parameters in a dense, regular grid structure [S1, unofficial][S2]. It trades a larger memory footprint for reduced computational cost during training, as only a small number of grid points are updated per back-propagated sample [S1, unofficial][S2]. Unlike fixed frequency encodings, it achieves better results in bounded domains, but it is single-scale and allocates parameters uniformly across the entire domain, which is wasteful in empty space [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   Input: A coordinate vector $\mathbf{x} \in \mathbb{R}^d$ (typically $d=3$ for 3D space) [G].
*   Output: A feature vector $\mathbf{f} \in \mathbb{R}^k$ (where $k$ is the feature dimension) [G].

**Data Structures:**
*   **Grid:** A dense $d$-dimensional array of shape $N_1 \times N_2 \times \dots \times N_d$ [G].
*   **Feature Vectors:** Each cell in the grid stores a trainable vector of size $k$ [S1, unofficial][S2].
*   **Total Parameters:** $P = (\prod_{i=1}^d N_i) \times k$ [G].

**Steps:**
1.  **Normalization:** Map the input coordinate $\mathbf{x}$ to the grid's coordinate space, typically normalizing it to the range $[0, 1]$ depending on the scene bounds [G]. Let the normalized coordinate be $\mathbf{x}'$ [G].
2.  **Grid Indexing:** Determine the integer grid indices for the corner cells surrounding $\mathbf{x}'$ [G]. For a 3D grid with resolution $N$, the indices are computed as $i_d = \lfloor x'_d \cdot N \rfloor$, where $i_d \in \{0, \dots, N-1\}$ [G]. This yields 8 corner indices for the voxel containing $\mathbf{x}'$ [G].
3.  **Interpolation Weights:** Compute the fractional part of the normalized coordinate to determine interpolation weights [G]. For each dimension $d$, $w_d = x'_d \cdot N - i_d$ [G]. The weights for the lower and upper corners are $1-w_d$ and $w_d$ respectively [G].
4.  **Feature Lookup:** Retrieve the $k$-dimensional feature vectors $\mathbf{v}_{c}$ for each of the 8 corner cells $c \in \{0, \dots, 7\}$ from the grid [S1, unofficial][S2].
5.  **Trilinear Interpolation:** Compute the final feature vector $\mathbf{f}$ by trilinearly interpolating the 8 corner vectors using the weights [G]. The formula is $\mathbf{f} = \sum_{c=0}^{7} \alpha_c \mathbf{v}_c$, where $\alpha_c$ is the product of the appropriate lower/upper weights for each dimension corresponding to corner $c$ [G].
6.  **Output:** Return $\mathbf{f}$ [G].

**Parameters:**
*   **Grid Resolution ($N$):** Determines the granularity. Typical values range from $16^3$ to $173^3$ depending on scene complexity and memory constraints [S1, unofficial][S2].
*   **Feature Dimension ($k$):** Size of the feature vector per cell. Values such as 2 or 16 are used [S1, unofficial][S2].

**Edge Cases and Pitfalls:**
*   **Empty Space Waste:** The method allocates features to empty space equally to surface areas. The parameter count grows as $O(N^d)$, while the visible surface area grows as $O(N^{d-1})$ [S1, unofficial][S2].
*   **Precision:** Floating-point precision in coordinate normalization and index calculation must be handled carefully to avoid boundary artifacts [G].
*   **Boundary Conditions:** Coordinates outside the grid bounds must be clamped or handled via padding to prevent out-of-bounds memory access [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dense_grid_encode(x, grid, resolution, feature_dim):
    """
    x: input coordinate vector of shape (3,), assumed in [-1, 1]
    grid: trainable parameter array of shape (resolution, resolution, resolution, feature_dim)
    resolution: integer grid size N
    feature_dim: integer k
    """
    # Normalize x to [0, 1] range
    x_norm = [(xi + 1.0) / 2.0 for xi in x]
    
    # Compute grid indices for the 8 corners
    indices = [int(xi * resolution) for xi in x_norm]
    
    # Clamp indices to valid range [0, resolution-1]
    indices = [max(0, min(resolution - 1, idx)) for idx in indices]
    
    # Compute fractional weights
    frac = [xi * resolution - idx for xi, idx in zip(x_norm, indices)]
    
    # Weights for lower (0) and upper (1) corners in each dimension
    w_low = [1.0 - f for f in frac]
    w_high = [f for f in frac]
    
    # Initialize output feature vector
    f = [0.0] * feature_dim
    
    # Iterate over 8 corners
    for i in (0, 1):
        for j in (0, 1):
            for l in (0, 1):
                # Corner index
                ci, cj, cl = indices[0] + i, indices[1] + j, indices[2] + l
                
                # Weight for this corner
                weight = (w_low[0] if i == 0 else w_high[0]) * \
                         (w_low[1] if j == 0 else w_high[1]) * \
                         (w_low[2] if l == 0 else w_high[2])
                 
                # Lookup feature vector
                v = grid[ci][cj][cl]
                
                # Accumulate
                for d in range(feature_dim):
                    f[d] += weight * v[d]
                    
    return f
```

## Subtopics

### Grid-based MS Module
A variant where the positional encoding $\gamma(\mathbf{p})$ in a hybrid multi-space module is replaced by features queried from a smaller dense grid $V_{vis}$ [S3]. The query function $Q(V_{vis}, \mathbf{p})$ replaces the standard positional encoding, allowing the multi-space scheme to be more position-dependent and efficient [S3].

## Operating characteristics
*   **Time Complexity:** Lookup and interpolation are $O(1)$ with respect to grid size, depending only on the number of corners (8 for 3D trilinear) and feature dimension $k$ [G].
*   **Memory Complexity:** $O(N^d \cdot k)$ [G]. For a $128^3$ grid with 16-dim features, this is approximately 33.6 million parameters [S1, unofficial][S2].
*   **Update Efficiency:** During back-propagation, only the 8 grid points affected by a sample are updated, regardless of the MLP size [S1, unofficial][S2]. This allows for faster convergence compared to fixed encodings where all weights are updated [S1, unofficial][S2].
*   **Strengths:** Efficient gradient updates; high approximation quality for bounded domains; simpler implementation than hash-based methods [S1, unofficial][S2].
*   **Limitations:** Wasteful in empty space; single-scale (unless combined with multi-resolution variants); memory usage grows cubically with resolution [S1, unofficial][S2].
*   **When to Use:** When memory is sufficient and the scene is bounded; when fast training convergence is prioritized over memory efficiency [S1, unofficial][S2].
*   **When Not to Use:** When the scene has large empty volumes or when memory is constrained [S1, unofficial][S2].

## Background
Dense Grid Encoding is a type of parametric encoding introduced in the context of neural implicit representations [S1, unofficial][S2]. It builds on earlier work using grids for 3D reconstruction [S1, unofficial][S2] and was popularized by its use in Instant-NGP [S1, unofficial][S2]. The method trades memory for computational efficiency by localizing gradient updates to a small number of grid cells [S1, unofficial][S2].

## Sources
[S1] Nvidia – learning representations of positions in a 3D scene [pdf], https://nvlabs.github.io/instant-ngp/assets/mueller2022instant.pdf, fetched 2026-10-26 (unofficial)
[S2] Instant Neural Graphics Primitives with a Multiresolution ..., https://www.cs.jhu.edu/~misha/ReadingSeminar/Papers/Mueller22.pdf, fetched 2026-10-28
[S3] MS-NeRF: Multi-Space Neural Radiance Fields, https://doi.org/10.1109/tpami.2025.3540074, fetched 2026-10-26
