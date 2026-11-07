# SparseOIT Rasterizer

Also called: GPU software rasterization of 3D Gaussians with pre-rendering

## Overview
The SparseOIT Rasterizer is a rendering technique for 3D Gaussian Splatting that computes a blended image using Order-Independent Transparency (OIT) without sorting Gaussians by depth [S1, unofficial]. It differs from standard 3DGS rasterization, which requires sorting by distance to the camera to correctly accumulate transmittance, by eliminating the sorting step entirely in favor of a weighted average that approximates the order-dependent result [S1, unofficial]. It simultaneously produces a final OIT color image and a pre-rendered state image using Blend-and-Normalize (BAN) and Blend-and-Update (BAU) functions.

## How it works
**Inputs:**
- Image dimensions: width $w$, height $h$.
- Gaussian attributes: 3D means $\mathbf{M}$, covariances $\mathbf{S}$, colors $\mathbf{C}$, opacities $\mathbf{A}$.
- View matrix: $\mathbf{V}$.

**Outputs:**
- Rendered image $\mathbf{I}$.
- Pre-rendered image $\mathbf{I}_{\text{pre}}$.

**Data Structures:**
- **Tiles ($T$):** A grid partitioning the image into square tiles.
- **Gaussian List ($L$):** A list of Gaussian data structures, duplicated per tile they overlap.
- **Tile Keys ($K$):** An integer key for each entry in $L$, representing the tile ID.
- **Tile Ranges ($R$):** A mapping from tile ID to the start and end indices in $L$ for that tile.

**Parameters:**
- **BAN (Blend and Normalize):** Computes the final OIT color for a pixel.
- **BAU (Blend and Update):** Updates the pre-rendered state for a pixel.
- **Weights ($w_i$):** Learnable or fixed weights associated with each Gaussian $i$.

**Algorithm Steps:**
1. **Cull:** Remove Gaussians outside the view frustum defined by $\mathbf{V}$ [S1, unofficial].
2. **Project:** Compute screen-space means $\mathbf{M}'$ and covariances $\mathbf{S}'$ for the remaining Gaussians using $\mathbf{V}$ [S1, unofficial].
3. **Tile Creation:** Partition the $w \times h$ image into tiles $T$ [S1, unofficial].
4. **Duplicate & Key:** For each Gaussian, determine which tiles it overlaps. Create an entry in $L$ for each overlap, assigning the corresponding tile ID to $K$ [S1, unofficial].
5. **Sort:** Sort the list $L$ based on keys $K$ (tile ID). This is the only sorting step; no depth sorting is performed [S1, unofficial].
6. **Range Identification:** Compute $R$, mapping each tile ID to the contiguous range of indices in $L$ belonging to that tile [S1, unofficial].
7. **Initialize:** Set $\mathbf{I} = 0$ and $\mathbf{I}_{\text{pre}} = 0$ (or appropriate initial state) [S1, unofficial].
8. **Rasterize Tiles:** Iterate over each tile $t \in T$:
   1. Get the range $r$ of Gaussians for tile $t$ from $R$ [S1, unofficial].
   2. Iterate over each pixel $i$ in tile $t$:
      1. Compute final color: $\mathbf{I}[i] \leftarrow \text{BAN}(i, L, r, K, \mathbf{M}', \mathbf{S}', \mathbf{C}, \mathbf{A}, \mathbf{I}_{\text{pre}}[i])$ [S1, unofficial].
      2. Update pre-rendered state: $\mathbf{I}_{\text{pre}}[i] \leftarrow \text{BAU}(i, L, r, K, \mathbf{M}', \mathbf{S}', \mathbf{C}, \mathbf{A}, \mathbf{I}_{\text{pre}}[i])$ [S1, unofficial].

**Formulas:**
The final color $\mathbf{C}$ (output $\mathbf{I}$) is computed as [S1, unofficial]:
$$
\mathbf{C} = T \mathbf{c}_0 + (1-T)F
$$
where:
- $\mathbf{c}_0$ is the background color.
- $T$ is the accumulated opacity/transmittance factor.
- $F$ is the weighted average color:
$$
F = \frac{P}{Q}, \quad P = \sum_{i=1}^{N} \mathbf{c}_i \alpha_i w_i, \quad Q = \sum_{i=1}^{N} \alpha_i w_i
$$
Here, $\mathbf{c}_i$ is the color of the $i$-th Gaussian, $\alpha_i$ is its opacity, and $w_i$ is its weight. $N$ is the number of Gaussians contributing to the pixel.

**Backward Propagation:**
The gradients for optimization are [S1, unofficial]:
$$
\frac{\partial \mathbf{C}}{\partial \mathbf{c}_i} = (1-T)\frac{\alpha_i w_i}{Q}
$$
$$
\frac{\partial \mathbf{C}}{\partial \alpha_i} = \frac{T}{1-\alpha_i}(F-\mathbf{c}_0) + (1-T)\frac{w_i}{Q}(\mathbf{c}_i - F)
$$
$$
\frac{\partial \mathbf{C}}{\partial w_i} = (1-T)\frac{\alpha_i}{Q}(\mathbf{c}_i - F)
$$
The variables of each Gaussian are mutually independent in these equations, allowing for parallel computation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def SparseOITRasterize(w, h, M, S, C, A, V):
    # 1. Cull Gaussians
    valid_indices = cull_gaussians(M, V)
    M_proj = M[valid_indices]
    S_proj = S[valid_indices]
    C_proj = C[valid_indices]
    A_proj = A[valid_indices]

    # 2. Create Tiles
    tiles = create_tiles(w, h)

    # 3. Duplicate with Keys
    # For each Gaussian, find overlapping tiles.
    # L stores Gaussian attributes for each tile overlap.
    # K stores the tile ID for each entry in L.
    L = []
    K = []
    for i in range(len(M_proj)):
        overlapping_tiles = get_overlapping_tiles(M_proj[i], S_proj[i], tiles)
        for tile_id in overlapping_tiles:
            L.append({
                'M': M_proj[i],
                'S': S_proj[i],
                'C': C_proj[i],
                'A': A_proj[i]
            })
            K.append(tile_id)

    # 4. Sort by Tile ID
    # Sort L and K based on K
    combined = list(zip(K, L))
    combined.sort(key=lambda x: x[0])
    K_sorted, L_sorted = zip(*combined)
    K_sorted = list(K_sorted)
    L_sorted = list(L_sorted)

    # 5. Identify Tile Ranges
    # R maps tile_id -> (start_index, end_index) in L_sorted
    R = {}
    current_tile = K_sorted[0]
    start = 0
    for idx, tile_id in enumerate(K_sorted):
        if tile_id != current_tile:
            R[current_tile] = (start, idx)
            start = idx
            current_tile = tile_id
    R[current_tile] = (start, len(K_sorted))

    # 6. Initialize Images
    I = zeros((h, w, 3))
    I_pre = zeros((h, w, 3)) # Or appropriate state structure

    # 7. Rasterize
    for tile in tiles:
        tile_id = tile.id
        if tile_id not in R:
            continue
        start_idx, end_idx = R[tile_id]
        tile_gaussians = L_sorted[start_idx:end_idx]

        for pixel in tile.pixels:
            # Compute BAN and BAU
            # Note: BAN and BAU details are not fully specified in the source
            # but BAN computes final color and BAU updates pre-render state.
            # We assume they iterate over tile_gaussians contributing to pixel.
            
            # Placeholder for BAN logic based on formula
            P = 0
            Q = 0
            for g in tile_gaussians:
                # Compute contribution to pixel
                # This requires evaluating Gaussian at pixel location
                # Using standard 3DGS projection logic [G]
                alpha = evaluate_gaussian_alpha(g['M'], g['S'], pixel)
                weight = g['A'] # Assuming A is opacity, weight w_i is separate or derived
                # Source uses w_i and alpha_i separately. 
                # In standard 3DGS, alpha is derived from covariance. 
                # Here, A is opacity. Let's assume w_i is a learnable parameter or 1.
                # The source defines P and Q with alpha_i and w_i.
                # If A is opacity, then alpha_i = A. w_i is separate.
                # If w_i is not provided, assume 1 [G]
                w_i = 1.0 
                alpha_i = g['A']
                c_i = g['C']
                
                P += c_i * alpha_i * w_i
                Q += alpha_i * w_i
            
            if Q > 0:
                F = P / Q
                # T is accumulated transmittance. 
                # In OIT without sorting, T is often approximated or part of the BAN state.
                # Source formula: C = T * c0 + (1-T) * F
                # If T is not explicitly computed in BAN, it might be 0 or derived from I_pre.
                # Assuming T=0 for pure OIT blend if no depth info [G]
                T = 0.0 
                c0 = background_color
                I[pixel] = T * c0 + (1 - T) * F
            
            # BAU updates I_pre
            # Details not provided in source [G]
            I_pre[pixel] = update_pre_render_state(I_pre[pixel], P, Q)

    return I, I_pre
```

## Subtopics

### [Weighted Order-Independent Transparency for 3DGS](7772-weighted-order-independent-transparency-for-3dgs.md)
Weighted Order-Independent Transparency (OIT) for 3D Gaussian Splatting (3DGS) is a rendering technique that computes a blended image without sorting Gaussians by depth, using a learnable weight function to approximate correct transparency blending. It differs from standard 3DGS rasterization, which requires sorting by distance to camera to correctly accumulate transmittance, by eliminating the sorting step entirely in favor of a weighted average that approximates the order-dependent result.

## Operating characteristics
- **Time Complexity:** The sorting step is $O(N \log N)$ where $N$ is the number of Gaussian-tile overlaps. The rasterization step is $O(P \cdot K)$ where $P$ is the number of pixels and $K$ is the average number of Gaussians per pixel.
- **Memory Complexity:** Requires storage for the duplicated Gaussian list $L$, which can be larger than the original Gaussian set if Gaussians overlap many tiles.
- **Strengths:** Eliminates the expensive depth sorting step required by standard 3DGS. Allows for parallel computation of gradients.
- **Limitations:** Approximates order-dependent transparency. The quality depends on the weight function $w_i$ and the BAN/BAU implementation.
- **When to use:** When order-independent transparency is desired and sorting is a bottleneck.
- **When not to use:** When exact depth ordering is critical for visual correctness.

## Background
The SparseOIT Rasterizer was introduced in the paper "SparseOIT: Improving Order-Independent Transparency 3DGS via Active Set Method" [S1, unofficial]. It builds on the 3D Gaussian Splatting framework [Parent](2812-3d-gaussian-splatting-rasterization.md) and aims to improve rendering efficiency and quality by avoiding depth sorting.

## Sources
[S1] SparseOIT: Improving Order-Independent Transparency 3DGS via Active Set Method, https://doi.org/10.48550/arXiv.2605.13855, fetched 2026-10-29 (unofficial)
