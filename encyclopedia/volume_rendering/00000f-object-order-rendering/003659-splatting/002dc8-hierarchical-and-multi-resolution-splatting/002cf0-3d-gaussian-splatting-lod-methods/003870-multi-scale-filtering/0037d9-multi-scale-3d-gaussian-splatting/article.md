# Multi-scale 3D Gaussian Splatting

Also called: Anti-Aliased 3D Gaussian Splatting, MS 3DGS, MS-GS, Multi-Scale Gaussians, Our Full Method

## Overview
Multi-scale 3D Gaussian Splatting (MS 3DGS) is a rendering algorithm that mitigates aliasing and performance degradation in 3D Gaussian Splatting by maintaining a multi-scale representation of the scene. Unlike single-scale methods that suffer from severe aliasing and speed deterioration at low resolutions, MS 3DGS selects Gaussians from different levels of detail based on their pixel coverage relative to the current render resolution [S1]. It solves the "missing parts" problem caused by naive filtering by aggregating small, high-frequency Gaussians into larger, low-frequency Gaussians during training [S1].

## How it works

### Inputs and Outputs
*   **Input:** A set of 3D Gaussians $\{G_k\}$ with attributes (position, scaling, opacity, color, spherical harmonics), a target rendering resolution scale $l_r$, and a set of training images/cameras.
*   **Output:** A rendered image at the target resolution.

### Data Structures
Each 3D Gaussian $G_k$ stores the following attributes:
*   **Position:** $\mu_k \in \mathbb{R}^3$
*   **Scaling:** $s_k \in \mathbb{R}^3$ (or covariance matrix derived from it)
*   **Opacity:** $\alpha_k \in [0, 1]$
*   **Color/Features:** Spherical Harmonics coefficients
*   **Level:** $l_k \in \{1, \dots, l_{max}\}$, where $l=1$ is the finest level.
*   **Pixel Coverage Bounds:**
    *   $S_k^{max}$: Maximum observed pixel coverage.
    *   $S_k^{min}$: Minimum observed pixel coverage.

**Parameters:**
*   $l_{max}$: Maximum number of detail levels. The method uses 4 levels corresponding to $1\times, 4\times, 16\times, 64\times$ downsampled resolutions [S1].
*   $S_T$: Selective rendering threshold. Set to $2$ pixels [S1].
*   $\sigma_T$: Opacity threshold for coverage calculation. Set to $1/255$ [S1].
*   $N$: Number of samples for 8-bit color normalization. Set to $255$ [S1].
*   $S_{rel\_max}$: Relative maximum coverage threshold. Empirical value $1.5$ [S1].
*   $S_{rel\_min}$: Relative minimum coverage threshold. Empirical value $0.5$ [S1].
*   $\lambda_1$: Decay coefficient for $S_k^{max}$. Empirical value $0.95$ [S1].
*   $\lambda_2$: Decay coefficient for $S_k^{min}$. Empirical value $1.05$ [S1].
*   Voxel Grid Resolution: For level $l_m$, the voxel grid size is $(400/l_m)^3$ [S1].

### Training Phase

#### Initialization
All 3D Gaussians initialized from the input point cloud belong to the finest level $l=1$. They are densified by splitting and cloning as in standard 3D Gaussian Splatting, inheriting level $l=1$ [S1].

#### Aggregation of Small Gaussians
After a warm-up stage of the first 1,000 iterations, coarse-level Gaussians are introduced by aggregating fine-level Gaussians that are too small [S1]. This process is repeated for each coarse level $l_m$ from $2$ to $l_{max}$:

1.  **Calculate Pixel Coverage:** Render all 3D Gaussians from levels $[1, l_m - 1]$ at a $4^{l_m - 1}$ times downsampled resolution for all training cameras. Calculate the pixel coverage $S_k$ for each Gaussian [S1].
    *   The pixel coverage $S_k$ is defined as the length of the horizontal or vertical axis of the splatted 2D Gaussian until the opacity drops to the threshold $\sigma_T = 1/255$ [S1].
    *   Mathematically, for a 2D Gaussian with parameter $a$ (inverse variance), $S_k$ is solved from $e^{-a(0.5 S_k)^2} = 1/N$, yielding $S_k = \frac{2}{\sqrt{a}} \sqrt{\ln N}$ [S1].
2.  **Select Small Gaussians:** Identify Gaussians as "too small" if their pixel coverage $S_k < S_T$ (where $S_T = 2$ pixels) [S1].
3.  **Voxelization:** Bin the selected small Gaussians into a voxel grid of resolution $(400/l_m)^3$ based on their positions [S1].
4.  **Average Pooling:** For each voxel containing small Gaussians, aggregate their attributes using average pooling to create a new Gaussian $G_{new}$. This includes averaging position, rotation, spherical harmonics features, opacity, and scaling [S1].
5.  **Enlarge Scaling:** Calculate the average pixel coverage $S_{avg}$ of the aggregated small Gaussians in the voxel. Enlarge the scaling of the new Gaussian by a factor of $S_T / S_{avg}$ so that its target pixel coverage is approximately $S_T$ [S1].
    *   This enlargement ensures the 3dB bandwidth of the scaled Gaussian is $f_{3dB}' = \frac{1}{\pi} \sqrt{\ln N \ln 2} \approx 0.441 \text{ px}^{-1}$, which is below the aliasing threshold of $0.5 \text{ px}^{-1}$ [S1].
6.  **Insert:** Assign the new Gaussian to level $l_m$ and insert it into the scene [S1].

#### Update Coverage Bounds
If the rendering downsample scale equals the scale at which a Gaussian $G_k$ was created (i.e., $l_k = l_m$), update its coverage bounds using the current pixel coverage $S_k$:
$$ S_k^{max'} = \max(\lambda_1 S_k^{max}, S_k) $$
$$ S_k^{min'} = \min(\lambda_2 S_k^{min}, S_k) $$
where $\lambda_1 = 0.95$ and $\lambda_2 = 1.05$ [S1].

#### Multi-Scale Training
After large Gaussians are added, the model is trained with both the original high-resolution images and the downsampled images corresponding to the various levels [S1]. Training runs for 40,000 iterations [S1].

### Rendering Phase

#### Selective Rendering
For a given rendering resolution scale $l_r$, select the subset of Gaussians to render based on their pixel coverage $S_k$ relative to their stored bounds $S_k^{max}$ and $S_k^{min}$ [S1].

A Gaussian $G_k$ is selected for rendering if it satisfies:
$$ \frac{S_k}{S_k^{max}} \le S_{rel\_max} \quad \land \quad \left( \frac{S_k}{S_k^{min}} \ge S_{rel\_min} \lor S_k \ge S_T \right) $$
where $S_{rel\_max} = 1.5$ and $S_{rel\_min} = 0.5$ [S1].

*   **Too Large:** If $S_k / S_k^{max} > S_{rel\_max}$, the Gaussian is filtered out because it is too large for the current resolution, preventing unnecessary computation and potential blur [S1].
*   **Too Small:** If $S_k / S_k^{min} < S_{rel\_min}$ AND $S_k < S_T$, the Gaussian is filtered out to avoid aliasing [S1]. The absolute threshold $S_T$ ensures that large Gaussians from lower scales (which do not cause aliasing) are not filtered out even if their relative coverage is small [S1].

#### Update Range
For all Gaussians at the current rendering level $l_r$, update their $S_k^{max}$ and $S_k^{min}$ using the decay coefficients as described in the Training Phase [S1].

#### Render
Render the selected subset of Gaussians $G_{selected} = G_1 \cap G_2$ at resolution $l_r$ using standard alpha blending [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def aggregate_small_gaussians(G_all, l_max, S_T, voxel_res_base=400):
    """
    Aggregates small Gaussians into larger ones for coarser levels.
    G_all: List of all Gaussians with attributes and level l_k.
    l_max: Maximum level index (e.g., 4).
    S_T: Selective rendering threshold (2 pixels).
    """
    for l_m in range(2, l_max + 1):
        # 1. Render all Gaussians from levels [1, l_m-1] at 4^(l_m-1) downsampled resolution
        # Calculate pixel coverage S_k for each Gaussian
        S_k = {}
        for G in G_all:
            if G.level < l_m:
                # Render G at scale 4^(l_m-1)
                # Calculate S_k based on opacity threshold sigma_T = 1/255
                S_k[G.id] = calculate_pixel_coverage(G, scale=4**(l_m-1))
        
        # 2. Select small Gaussians
        G_small = [G for G in G_all if G.id in S_k and S_k[G.id] < S_T]
        
        # 3. Voxelize and Aggregate
        voxel_size = voxel_res_base / l_m
        voxels = create_voxel_grid(voxel_size)
        
        for G in G_small:
            voxels[get_voxel_index(G.position, voxel_size)].append(G)
            
        for voxel in voxels:
            if not voxel:
                continue
            
            # Average pooling of attributes
            G_new = average_pool(voxel)
            
            # Calculate average pixel coverage of aggregated Gaussians
            S_avg = mean([S_k[G.id] for G in voxel])
            
            # Enlarge scaling by S_T / S_avg
            G_new.scale = G_new.scale * (S_T / S_avg)
            
            # Assign to level l_m
            G_new.level = l_m
            
            # Insert into scene
            G_all.append(G_new)

def selective_render(G_all, l_r, S_rel_max=1.5, S_rel_min=0.5, S_T=2.0):
    """
    Selects Gaussians for rendering based on pixel coverage.
    """
    # 1. Calculate current pixel coverage S_k for all Gaussians at resolution l_r
    S_k = {}
    for G in G_all:
        S_k[G.id] = calculate_pixel_coverage(G, scale=4**(l_r-1))
        
    # 2. Update bounds for Gaussians at the current level
    for G in G_all:
        if G.level == l_r:
            Sk = S_k[G.id]
            G.S_max = max(0.95 * G.S_max, Sk)
            G.S_min = min(1.05 * G.S_min, Sk)
            
    # 3. Select Gaussians
    G_selected = []
    for G in G_all:
        Sk = S_k[G.id]
        cond1 = (Sk / G.S_max) <= S_rel_max
        cond2 = (Sk / G.S_min) >= S_rel_min
        cond3 = Sk >= S_T
        
        if cond1 and (cond2 or cond3):
            G_selected.append(G)
            
    # 4. Render selected Gaussians
    return render_alpha_blend(G_selected)

def calculate_pixel_coverage(G, scale):
    """
    Calculates pixel coverage Sk for a Gaussian at a given scale.
    Sk is the axis length where opacity drops to 1/255.
    """
    # Inverse variance 'a' derived from Gaussian scaling
    a = get_inverse_variance(G)
    N = 255
    # S_k = (2 / sqrt(a)) * sqrt(ln(N))
    Sk = (2.0 / math.sqrt(a)) * math.sqrt(math.log(N))
    return Sk
```

## Operating characteristics

*   **Time Complexity:** Rendering time is not linearly proportional to resolution decrease. While fewer Gaussians are rendered at lower resolutions, all Gaussians must still be splatted to calculate pixel coverage for selection, creating overhead [S1].
*   **Memory Complexity:** Increases slightly due to the storage of additional coarse-level Gaussians (typically <5% of total) and per-Gaussian coverage bounds [S1].
*   **Performance:**
    *   On the 360 dataset at 64x scale: PSNR 25.35 vs 15.23 (3DGS), LPIPS 0.066 vs 0.256 (3DGS), Time 4.9ms vs 103.3ms (3DGS) [S1].
    *   On Tank and Temples dataset at 64x scale: PSNR 19.67 vs 13.88 (3DGS), LPIPS 0.034 vs 0.447 (3DGS), Time 5.9ms vs 82.6ms (3DGS) [S1].
    *   On Deep Blending dataset at 64x scale: PSNR 25.70 vs 17.75 (3DGS), LPIPS 0.036 vs 0.379 (3DGS), Time 3.4ms vs 59.7ms (3DGS) [S1].
    *   Achieves 13%-66% PSNR and 160%-2400% rendering speed improvements at 4x-128x scale rendering compared to vanilla 3DGS [S1].
*   **Strengths:** Eliminates aliasing artifacts at low resolutions; significantly improves rendering speed at low resolutions; maintains comparable quality and speed at 1x resolution [S1].
*   **Limitations:** Performance overhead at very low resolutions due to the necessity of splatting all Gaussians for coverage calculation; requires multi-scale training with downsampled images [S1].

## Background

Multi-scale 3D Gaussian Splatting was introduced to address the aliasing and speed deterioration issues inherent in the original 3D Gaussian Splatting method when rendering at varying resolutions. The method was proposed by the authors of the paper "Multi-Scale 3D Gaussian Splatting for Anti-Aliased Rendering" [S1]. It builds upon the principles of 3D Gaussian Splatting [12] and addresses limitations found in traditional anti-aliasing techniques like EWA splatting, which apply filters to individual Gaussians rather than the scene level [S1]. The method is evaluated on the MipNeRF360, Tank and Temples, and Deep Blending datasets [S1].

## Sources

[S1] Multi-Scale 3D Gaussian Splatting for Anti-Aliased Rendering, https://doi.org/10.1109/cvpr52733.2024.01977, fetched 2026-10-26
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
