# SqueezeNeRF

Also called: SqueezeNeRF (1K cache)

## Overview
SqueezeNeRF is a memory-efficient neural rendering method that further factorizes the position-dependent Multi-Layer Perceptron (MLP) of FastNeRF into three separate 2D networks operating on coordinate pairs $(x,y)$, $(y,z)$, and $(z,x)$ to reduce cache size complexity from $O(n^3)$ to $O(n^2)$ [S1]. It builds on [NeRF Network Query with Hashed Embeddings](3107-nerf-network-query-with-hashed-embeddings.md) by replacing the dense 3D position cache with a set of 2D position caches and a 2D direction cache, enabling real-time inference with significantly lower memory overhead than FastNeRF or NeRF-SH while maintaining high rendering speeds [S1].

## How it works
### Inputs and Outputs
**Inputs:**
- Camera viewing direction $(\theta, \phi)$ [S1].
- 3D spatial coordinates $(x, y, z)$ of sample points along a ray [S1].
- Pre-trained cache files containing outputs of the position-conditioned and view-conditioned networks [S1].

**Outputs:**
- Volume density $\sigma$ [S1].
- Radiance map components $(u, v, w)$ [S1].
- Final pixel color $(r, g, b)$ via volume rendering [S1].

### Data Structures and Cache Layout
The method relies on dense caches for the outputs of the neural networks. The cache layout differs between the default architecture and the alternate architecture.

**Default Architecture Cache:**
- **Position Caches:** Three caches corresponding to networks $F_{pos}^{xy}$, $F_{pos}^{yz}$, and $F_{pos}^{zx}$. Each cache stores outputs for $n_p^2$ coordinate pairs.
  - For each pair, the network outputs a scalar $\sigma_0^j$ and three $D$-dimensional vectors $(u_0^j, v_0^j, w_0^j)$, along with scaling weights $W_\sigma^j$ (scalar) and $W_k^j$ ($D$-dimensional vectors for $k \in \{u,v,w\}$) [S1].
  - Storage: $\sigma_0^j$ is stored as a 32-bit float. $u_0^j, v_0^j, w_0^j$ and weights are stored as 16-bit floats [S1].
  - Total storage per 3D point combination (conceptually $n_p^3$) is $3n_p^2(3D + 1)$ values [S1].
- **Direction Cache:** One cache for $F_{dir}$ storing $\beta$ for $n_d^2$ direction pairs $(\theta, \phi)$.
  - Storage: $\beta$ is stored as 16-bit floats. Total storage is $n_d^2(D)$ values [S1].

**Alternate Architecture Cache (SqueezeNeRF alt):**
- **Position Caches:** Same structure as default, but outputs are vectors.
  - $F_{pos}^j$ outputs $\sigma^j \in \mathbb{R}^{D_\sigma}$ and $k^j \in \mathbb{R}^{D_k}$ for $k \in \{u,v,w\}$ [S1].
  - Storage: All values stored as 16-bit floats [S1].
  - Total storage per 3D point combination is $3n_p^2(3D + D_\sigma)$ values [S1].
- **Direction Cache:** Same as default [S1].

**Parameters:**
- $n_p$: Number of bins per position dimension. Typical values: 512 ('512 cache') or 1024 ('1K cache') [S1].
- $n_d$: Number of bins per direction dimension. Typical values: 64 (synthetic 360° scenes) or 32 (LLFF dataset) [S1].
- $D$: Dimensionality of radiance map vectors. Typical value: 8 [S1].
- $D_\sigma, D_k$: Dimensionality of outputs in alternate architecture. Typical values: $D_\sigma=12, D_k=8$ [S1].
- MLP Hidden Dimensions: Position MLPs have 6 layers with 256 hidden units; View MLP has 4 layers with 128 hidden units [S1].

### Numbered Steps

1.  **Network Factorization:** Split the position-dependent network $F_{pos}$ from FastNeRF into three separate MLPs: $F_{pos}^{xy}$, $F_{pos}^{yz}$, and $F_{pos}^{zx}$ [S1].
2.  **Network Definition:**
    -   For the **default** configuration, each $F_{pos}^j$ (where $j \in \{(x,y), (y,z), (z,x)\}$) maps 2D coordinates to:
        $$ F_{pos}^j: j \rightarrow \{(\sigma_0^j, W_\sigma^j), (w_j^u, u_0^j, w_j^v, v_0^j, w_j^w, w_0^j)\} $$
        where $\sigma_0^j$ is a scalar and $u_0^j, v_0^j, w_0^j$ are $D$-dimensional vectors [S1].
    -   For the **alternate** configuration, $F_{pos}^j$ maps to $\{\sigma^j, u^j, v^j, w^j\}$ where $\sigma^j \in \mathbb{R}^{D_\sigma}$ and $k^j \in \mathbb{R}^{D_k}$ [S1].
    -   The view-dependent network $F_{dir}$ remains unchanged from FastNeRF, mapping $(\theta, \phi) \rightarrow \beta \in \mathbb{R}^D$ [S1].
3.  **Cache Generation (Training Phase):**
    -   Train the networks using standard NeRF training procedures (hierarchical sampling, positional encoding) [S1].
    -   Discretize the input space into a grid of $n_p \times n_p$ for each position pair and $n_d \times n_d$ for directions [S1].
    -   Compute the network outputs for all grid points and store them in the cache files using the precision specified (32-bit for $\sigma_0$, 16-bit for others) [S1].
4.  **Inference Query (Ray Marching):**
    -   Sample $N=256$ points along the ray [S1].
    -   For each sample point with coordinates $(x,y,z)$ and direction $(\theta, \phi)$:
        a.  **Fetch Position Outputs:** Retrieve $\sigma_0^j, u_0^j, v_0^j, w_0^j$ (or $\sigma^j, k^j$ for alt) from the three position caches using **nearest neighbor interpolation** [S1].
        b.  **Fetch Direction Output:** Retrieve $\beta$ from the direction cache using **bilinear interpolation** [S1].
        c.  **Compute Intermediate Density and Radiance (Default):**
            $$ \sigma^j = \sigma_0^j \cdot W_\sigma^j $$
            $$ k^j = W_k^j \cdot k_0^j \quad \forall k \in \{u, v, w\} $$
            where $\cdot$ denotes pointwise multiplication [S1].
        d.  **Combine Outputs (Default):**
            $$ \sigma = \prod_{j \in \{xy, yz, zx\}} \sigma^j $$
            $$ k = \sum_{j \in \{xy, yz, zx\}} k^j \quad \forall k \in \{u, v, w\} $$
            Thus, $u = \sum u^j, v = \sum v^j, w = \sum w^j$ [S1].
        e.  **Combine Outputs (Alternate):**
            $$ \sigma = W_\sigma [\text{concat}(\{\sigma^j\})] + b_\sigma $$
            $$ k = W_k [\text{concat}(\{k^j\})] + b_k \quad \forall k \in \{u, v, w\} $$
            where $W_\sigma \in \mathbb{R}^{1 \times 3D_\sigma}, b_\sigma \in \mathbb{R}, W_k \in \mathbb{R}^{D \times 3D_k}, b_k \in \mathbb{R}^D$ are learnable parameters stored in the cache [S1].
        f.  **Compute Color:**
            $$ c = (r, g, b) = \sum_{i=1}^D \beta_i (u_i, v_i, w_i) = \beta^T \cdot (u, v, w) $$
            [S1].
        g.  **Volume Rendering:** Compute final pixel color using standard volume rendering integration (Eq. 1 in [S1]) based on $\sigma$ and $c$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def load_cache(cache_path, dtype_sigma=np.float32, dtype_other=np.float16):
    """
    Loads the pre-computed caches for SqueezeNeRF.
    Returns dictionaries for position caches (xy, yz, zx) and direction cache.
    """
    # Implementation detail: Load binary files or numpy arrays.
    # Structure depends on n_p and n_d.
    # For default:
    # pos_cache[j] contains:
    #   sigma_0: shape (n_p, n_p), dtype=dtype_sigma
    #   u_0: shape (n_p, n_p, D), dtype=dtype_other
    #   v_0: shape (n_p, n_p, D), dtype=dtype_other
    #   w_0: shape (n_p, n_p, D), dtype=dtype_other
    #   w_sigma: shape (n_p, n_p), dtype=dtype_other
    #   w_u: shape (n_p, n_p, D), dtype=dtype_other
    #   w_v: shape (n_p, n_p, D), dtype=dtype_other
    #   w_w: shape (n_p, n_p, D), dtype=dtype_other
    #
    # For alt:
    # pos_cache[j] contains:
    #   sigma: shape (n_p, n_p, D_sigma), dtype=dtype_other
    #   u: shape (n_p, n_p, D_k), dtype=dtype_other
    #   v: shape (n_p, n_p, D_k), dtype=dtype_other
    #   w: shape (n_p, n_p, D_k), dtype=dtype_other
    #   W_sigma: shape (1, 3*D_sigma), dtype=dtype_other (shared)
    #   b_sigma: shape (1,), dtype=dtype_other (shared)
    #   W_k: shape (D, 3*D_k), dtype=dtype_other (shared)
    #   b_k: shape (D,), dtype=dtype_other (shared)
    #
    # dir_cache contains:
    #   beta: shape (n_d, n_d, D), dtype=dtype_other
    #   W_sigma, b_sigma, W_k, b_k for alt mode if applicable.
    pass

def nearest_neighbor_interpolate(cache_grid, coord):
    """
    Performs nearest neighbor interpolation on a 2D cache grid.
    coord: (u, v) float coordinates in [0, 1] or [0, n_p-1] depending on grid definition.
    """
    # Map continuous coord to integer indices
    # Handle boundary clamping
    idx_u = int(coord[0] * (cache_grid.shape[0] - 1))
    idx_v = int(coord[1] * (cache_grid.shape[1] - 1))
    idx_u = np.clip(idx_u, 0, cache_grid.shape[0] - 1)
    idx_v = np.clip(idx_v, 0, cache_grid.shape[1] - 1)
    return cache_grid[idx_u, idx_v]

def bilinear_interpolate(cache_grid, coord):
    """
    Performs bilinear interpolation on a 2D cache grid.
    """
    # Standard bilinear interpolation logic
    pass

def squeeze_nerf_infer_default(ray_origins, ray_directions, n_samples=256, 
                               pos_caches=None, dir_cache=None, D=8):
    """
    Default SqueezeNeRF inference.
    """
    # 1. Sample points along rays
    # t_vals = np.linspace(0, 1, n_samples)
    # points = ray_origins[:, None, :] + ray_directions[:, None, :] * t_vals[None, :, None]
    # points shape: (num_rays, n_samples, 3)
    
    # 2. Initialize accumulators
    # sigma_total = 0.0
    # u_total = np.zeros((num_rays, D))
    # v_total = np.zeros((num_rays, D))
    # w_total = np.zeros((num_rays, D))
    
    # 3. Iterate over samples
    # for i in range(n_samples):
    #     p = points[:, i, :] # (num_rays, 3)
    #     theta = np.arccos(p[:, 2] / np.linalg.norm(p, axis=1)) # Example mapping, actual depends on coordinate system
    #     phi = np.arctan2(p[:, 1], p[:, 0])
    #     # Normalize theta, phi to [0, 1] for cache lookup
    #     
    #     # Fetch from position caches
    #     # xy_cache = pos_caches['xy']
    #     # sigma_0_xy = nearest_neighbor_interpolate(xy_cache['sigma_0'], (p_x, p_y))
    #     # ... repeat for yz, zx
    #     
    #     # Compute sigma^j and k^j
    #     # sigma_xy = sigma_0_xy * w_sigma_xy
    #     # u_xy = w_u_xy * u_0_xy
    #     # ...
    #     
    #     # Combine
    #     sigma_total *= (sigma_xy * sigma_yz * sigma_zx) # Product rule
    #     u_total += (u_xy + u_yz + u_zx)
    #     v_total += (v_xy + v_yz + v_zx)
    #     w_total += (w_xy + w_yz + w_zx)
    #
    # 4. Fetch direction cache
    # beta = bilinear_interpolate(dir_cache['beta'], (theta, phi))
    #
    # 5. Compute color
    # c = beta * np.stack([u_total, v_total, w_total], axis=-1)
    # c = np.sum(c, axis=-1) # Dot product
    #
    # 6. Volume render
    # return final_colors

def squeeze_nerf_infer_alt(ray_origins, ray_directions, n_samples=256, 
                           pos_caches=None, dir_cache=None, D=8, D_sigma=12, D_k=8):
    """
    Alternate SqueezeNeRF inference.
    """
    # Similar structure to default, but combination step uses linear layers
    # sigma_combined = W_sigma @ concat(sigma_xy, sigma_yz, sigma_zx) + b_sigma
    # k_combined = W_k @ concat(u_xy, u_yz, u_zx, ...) + b_k
    pass
```

### Edge Cases and Pitfalls
-   **Coordinate Pairing:** Splitting coordinates into individual dimensions $(x), (y), (z)$ instead of pairs $(x,y), (y,z), (z,x)$ leads to considerably inferior results [S1].
-   **Interpolation:** Nearest neighbor interpolation is used for position caches, which may introduce artifacts if $n_p$ is too small. Bilinear interpolation is used for direction caches [S1].
-   **Precision:** Using 16-bit floats for radiance components and 32-bit for density scalars is critical for memory efficiency but may affect precision in extreme cases [S1].
-   **Cache Size vs Quality:** Smaller caches (512 vs 1K) reduce memory but may degrade image quality (PSNR/SSIM) [S1].

## Operating characteristics
-   **Memory Complexity:** $O(n_p^2)$ for position caches and $O(n_d^2)$ for direction cache, compared to $O(n^3)$ for FastNeRF [S1].
-   **Memory Usage:**
    -   SqueezeNeRF (1K cache, $n_p=1024$): ~155 MB [S1].
    -   SqueezeNeRF (512 cache, $n_p=512$): ~39 MB [S1].
    -   SqueezeNeRF (alt) (512 cache): ~58 MB [S1].
    -   Comparison: FastNeRF 1K cache ~54 GB, NeRF 1K cache ~35 TB [S1].
-   **Speed:**
    -   Over 150 FPS on high-end GPUs (e.g., V100, A100) [S1].
    -   Specifics: 168 FPS on V100, 202 FPS on A100 for 512 cache [S1].
    -   Cache generation time: < 9 seconds on V100 [S1].
-   **Quality:**
    -   Slightly inferior to NeRF and some baselines in PSNR/SSIM due to simple combination scheme [S1].
    -   Synthetic 360° (800x800): 29.61 PSNR (1K), 28.12 PSNR (512) [S1].
    -   LLFF (504x378): 21.62 PSNR (1K), 23.87 PSNR (512) [S1].
-   **Strengths:** Extremely low memory footprint, fast cache generation, real-time inference speed.
-   **Limitations:** Lower image quality compared to full NeRF or FastNeRF with sparse caching; simple combination function limits expressiveness [S1].

## Background
SqueezeNeRF was proposed by Wadhwani et al. in 2022 [S1, S2]. It is a further factorization of FastNeRF [S1]. FastNeRF itself was the first NeRF-based model to render at over 200 Hz on GPU [S2]. SqueezeNeRF addresses the substantial memory overhead of FastNeRF by reducing cache size complexity from $O(n^3)$ to $O(n^2)$ [S1]. It is designed for deployment on embedded systems and real-time applications where memory is constrained [S1].

## Sources
[S1] SqueezeNeRF: Further factorized FastNeRF for memory-efficient inference, https://doi.org/10.1109/cvprw56347.2022.00307, fetched 2026-10-26
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
