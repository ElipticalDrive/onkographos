# Neural Deformable Voxel Grid (NDVG)
## Overview
Neural Deformable Voxel Grid (NDVG) is a fast optimization method for dynamic scene view synthesis that integrates voxel-grid optimization with a deformable radiance field [S1]. It replaces heavy Multi-Layer Perceptron (MLP) computations with fast voxel-grid feature interpolation to enable space-time view synthesis for dynamic scenes captured by a monocular camera [S1]. Unlike static scene applications where occlusion is determined solely by viewing direction, NDVG explicitly models occlusion in dynamic scenes, which is determined by both view direction and motion (time) [S1]. It achieves comparable performance to D-NeRF but trains more than 70× faster [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of images $\{I_t\}_{t=1}^T$ with corresponding camera poses $\{T_t\}_{t=1}^T$ from a monocular camera [S1]. A query consists of a 3D point $\mathbf{p}$, viewing direction $\mathbf{d}$, and time instance $t$ [S1].
*   **Outputs:** Density $\sigma$ and color $\mathbf{c}$ for the query, which are used to render pixel colors via volume rendering [S1].

**Data Structures:**
The method uses a hybrid explicit-implicit representation consisting of three 3D feature grids and two light-weight MLPs [S1].
1.  **Deformation Feature Grid ($G_d$):** Layout: Tensor of shape $(N_x, N_y, N_z, N_c)$ [S1]. Content: Stores 3D dynamic features for deformation [S1]. Access: Trilinear interpolation [S1].
2.  **Density Grid ($G_\sigma$):** Layout: Tensor of shape $(N'_x, N'_y, N'_z)$ [S1]. Content: Stores density values [S1]. Access: Trilinear interpolation followed by softplus activation [S1].
3.  **Color Feature Grid ($G_c$):** Layout: Tensor of shape $(H', W', D', N'_c)$ [S1]. Content: Stores view-dependent color features [S1]. Access: Trilinear interpolation [S1].
4.  **Deformation MLP ($F_{\theta^d_1}$):** Architecture: 4 layers, width 64 [S1]. Input: Concatenation of positional encoding of $\mathbf{p}$, positional encoding of $t$, and interpolated feature $\hat{\mathbf{f}}^d$ [S1]. Output: Deformation offset $\Delta \mathbf{p}$ and occlusion weight $\hat{w}^{occ}$ [S1].
5.  **Canonical MLP ($F_{\theta^c_2}$):** Architecture: 3 layers, width 128 [S1]. Input: Concatenation of positional encoding of canonical position $\mathbf{p}'$, viewing direction $\mathbf{d}$, and interpolated feature $\hat{\mathbf{f}}^c$ [S1]. Output: Color $\mathbf{c}$ [S1].

**Steps:**
1.  **Deformation Query:** For a query point $\mathbf{p}$ at time $t$, perform trilinear interpolation on $G_d$ to get feature $\hat{\mathbf{f}}^d$ [S1]. Input $\mathbf{p}$, $t$, and $\hat{\mathbf{f}}^d$ into $F_{\theta^d_1}$ to regress offset $\Delta \mathbf{p}$ and occlusion weight $\hat{w}^{occ}$ [S1]. If $t = t_{can}$, set $\Delta \mathbf{p} = \mathbf{0}$ and $\hat{w}^{occ} = 1$ [S1].
2.  **Canonical Position:** Compute canonical position $\mathbf{p}' = \mathbf{p} + \Delta \mathbf{p}$ [S1].
3.  **Density Query:** For canonical point $\mathbf{p}'$, perform trilinear interpolation on $G_\sigma$ [S1]. Apply softplus activation: $\sigma = \text{softplus}(\text{interp}(\mathbf{p}', G_\sigma))$ [S1].
4.  **Color Query:** For canonical point $\mathbf{p}'$ and viewing direction $\mathbf{d}$, perform trilinear interpolation on $G_c$ to get $\hat{\mathbf{f}}^c$ [S1]. Input $\mathbf{p}'$, $\mathbf{d}$, and $\hat{\mathbf{f}}^c$ into $F_{\theta^c_2}$ to get color $\mathbf{c}$ [S1].
5.  **Occlusion Filtering:** Multiply density and color by occlusion mask: $\sigma' = \sigma \times \hat{w}^{occ}$, $\mathbf{c}' = \mathbf{c} \times \hat{w}^{occ}$ [S1].
6.  **Volume Rendering:** Render ray color $\hat{C}(r)$ using standard volume rendering equation with filtered density and color [S1].

**Formulas:**
*   **Photometric Loss:** $L_{\text{photo}} = \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \| \hat{C}(r) - C(r) \|_2^2$, where $\mathcal{R}$ is the set of rays sampled in one batch [S1].
*   **Deformation L1 Regularization:** $L_{d\_norm} = \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \sum_{i=1}^{K} \| \Delta \mathbf{p}_i \|_1$, where $i$ indexes sampled points $K$ on ray $r$ [S1].
*   **Total Variation Loss on Deformation Grid:** $L_{d\_tv} = \frac{1}{|\mathcal{V}|} \sum_{\mathbf{v} \in \mathcal{V}} \sum_{c \in [N_c]} \sqrt{\Delta_x^2(\mathbf{v},c) + \Delta_y^2(\mathbf{v},c) + \Delta_z^2(\mathbf{v},c)}$, where $\Delta_x^2(\mathbf{v},c)$ is the squared difference between the $c$-th element in voxel $\mathbf{v}(i,j,k)$ and voxel $\mathbf{v}(i+1,j,k)$ [S1].
*   **Total Loss:** $L = L_{\text{photo}} + w_{ptc} L_{ptc} + w_{bg} L_{bg} + w_{d\_norm} L_{d\_norm} + w_{d\_tv} L_{d\_tv}$ [S1].

**Parameters:**
*   **Positional Encoding:** Frequency 5 for position and time, 4 for direction [S1].
*   **Canonical Time ($t_{can}$):** The reference time for the canonical space [S1].
*   **Grid Resolutions:** Coarse Stage: Deformation module $1,664k$ voxels; Canonical module $1,024k$ voxels [S1]. Fine Stage: Deformation module $190^3$ voxels; Canonical module $160^3$ voxels [S1]. Coarse Stage Color Grid Dimension: 3 (RGB) [S1].
*   **Training Iterations:** Full Resolution: 10k iterations for coarse stage, 20k for fine stage [S1]. Half Resolution: 5k iterations for coarse stage, 10k for fine stage [S1].
*   **Optimization:** Adam optimizer, 8,192 rays sampled per iteration [S1].

**Edge Cases and Pitfalls:**
*   **Occlusion Artifacts:** If an empty point in observation space is static ($\Delta \mathbf{p}=\mathbf{0}$) but maps to an occupied location in canonical space, it retrieves non-zero density/color, causing artifacts [S1]. This is solved by the explicit occlusion mask $\hat{w}^{occ}$ [S1].
*   **Inefficient 4D Grid:** Directly storing offsets in a 4D feature grid (3 for $\mathbf{p}$, 1 for $t$) is inefficient [S1]. NDVG uses a 3D feature grid and a light-weight MLP to decode the feature instead [S1].
*   **Resolution Sensitivity:** Deformation is more sensitive to resolution than radiance (density and color), so the deformation module grid resolution is set higher than the canonical module [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def query_ndvg(p, d, t, Gd, Gs, Gc, Fd1, Fc2, t_can):
    # 1. Deformation Query
    f_d = trilinear_interp(p, Gd)
    if t == t_can:
        delta_p = [0.0, 0.0, 0.0]
        w_occ = 1.0
    else:
        # Positional encoding applied to p and t before MLP input [G]
        delta_p, w_occ = Fd1(pos_enc(p), pos_enc(t), f_d)
    
    # 2. Canonical Position
    p_prime = [p[i] + delta_p[i] for i in range(3)]
    
    # 3. Density Query
    v_sigma = trilinear_interp(p_prime, Gs)
    sigma = softplus(v_sigma)
    
    # 4. Color Query
    f_c = trilinear_interp(p_prime, Gc)
    # Positional encoding applied to p_prime and d before MLP input [G]
    c = Fc2(pos_enc(p_prime), pos_enc(d), f_c)
    
    # 5. Occlusion Filtering
    sigma_prime = sigma * w_occ
    c_prime = c * w_occ
    
    return sigma_prime, c_prime, w_occ

def render_ray(ray_origin, ray_dir, t, K, Gd, Gs, Gc, Fd1, Fc2, t_can):
    # Sample K points along ray and compute distances between them [G]
    points = sample_points(ray_origin, ray_dir, K)
    deltas = compute_distances(points)
    
    sigma_primes = []
    c_primes = []
    for p in points:
        sigma_p, c_p, _ = query_ndvg(p, ray_dir, t, Gd, Gs, Gc, Fd1, Fc2, t_can)
        sigma_primes.append(sigma_p)
        c_primes.append(c_p)
        
    # Volume Rendering
    color = 0.0
    T = 1.0
    for k in range(K):
        sigma_k = sigma_primes[k]
        delta_k = deltas[k]
        alpha_k = 1.0 - exp(-sigma_k * delta_k)
        color += T * alpha_k * c_primes[k]
        T *= exp(-sigma_k * delta_k)
    return color
```

## Operating characteristics
*   **Speed:** NDVG achieves comparable performance to D-NeRF using only 20 minutes for training, which is more than 70× faster than D-NeRF [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Evaluated on the D-NeRF dataset (eight dynamic scenes, 360° viewpoint settings) using PSNR, SSIM, and LPIPS metrics [S1].
*   **Strengths:** Fast optimization, efficient space-time view synthesis for dynamic scenes, explicit occlusion handling for monocular input [S1].
*   **Limitations:** Not given in the sources.
*   **When to use:** When fast training of dynamic scenes from monocular video is required [S1].
*   **When not to use:** When multi-view data is available and DeVRF is preferred, or when image quality refinement via look-up tables (V4D) is prioritized over training speed [S1].

## Background
NDVG was proposed by X. Guo et al. in the paper "Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis" [S1]. It is based on the canonical scene framework where a deformation module maps observation space to canonical space, and volume rendering is performed in the canonical space [S1]. It builds on voxel-grid optimization techniques used in static radiance fields [S1].

## Sources
[S1] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
