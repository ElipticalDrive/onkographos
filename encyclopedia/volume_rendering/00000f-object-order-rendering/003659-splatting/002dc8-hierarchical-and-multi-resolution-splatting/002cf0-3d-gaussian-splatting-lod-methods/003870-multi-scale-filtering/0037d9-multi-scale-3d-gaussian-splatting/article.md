# Multi-scale 3D Gaussian Splatting

Also called: Anti-Aliased 3D Gaussian Splatting, MS 3DGS, MS-GS, Multi-Scale Gaussians, Our Full Method

## Overview
Multi-scale 3D Gaussian Splatting (MS 3DGS) is a rendering algorithm that maintains 3D Gaussians at different levels of detail (LOD) to represent the same scene, using selective rendering based on pixel coverage to mitigate aliasing and improve rendering speed at low resolutions [S1]. It solves the problem where standard 3D Gaussian Splatting (3DGS) suffers from severe aliasing and performance degradation when rendering at resolutions significantly different from the training resolution [S1]. Unlike [MIP-Splatting](6063-mip-splatting.md), which applies a low-pass filter to individual Gaussians, MS 3DGS tackles aliasing on the scene level by aggregating small, high-frequency Gaussians into larger, low-frequency Gaussians and selecting the appropriate subset based on the current rendering scale [S1].

## How it works

### Inputs and Outputs
*   **Inputs:** A set of 3D Gaussians $\{G_k\}$ with attributes (position, scaling, rotation, opacity, spherical harmonics coefficients), a target rendering resolution scale $l_r$ (relative to the base training resolution), and a set of training images/cameras.
*   **Outputs:** A rendered image at the target resolution.

### Data Structures
Each Gaussian $G_k$ stores the standard 3DGS attributes plus two additional scalar values for selective rendering:
*   $S_k^{max}$: Maximum pixel coverage observed during training.
*   $S_k^{min}$: Minimum pixel coverage observed during training.
*   $l_k$: The level of detail index ($1 \le l_k \le l_{max}$) to which the Gaussian belongs.

The scene is organized into $l_{max}$ levels of detail. In the source implementation, $l_{max} = 4$, corresponding to scales $1\times, 4\times, 16\times, 64\times$ [S1].

### Parameters
*   $l_{max}$: Number of detail levels. Default: 4 [S1].
*   $S_T$: Selective rendering threshold (minimum pixel coverage to avoid aliasing). Default: 2 pixels [S1].
*   $\sigma_T$: Opacity threshold for defining pixel coverage extent. Default: $1/255$ [S1].
*   $S_{rel\_max}$: Relative maximum coverage threshold. Default: 1.5 [S1].
*   $S_{rel\_min}$: Relative minimum coverage threshold. Default: 0.5 [S1].
*   $\lambda_1$: Decay coefficient for updating $S_k^{max}$. Default: 0.95 [S1].
*   $\lambda_2$: Decay coefficient for updating $S_k^{min}$. Default: 1.05 [S1].
*   $V_{res}$: Voxel grid resolution for aggregation at level $l_m$. Formula: $(400/l_m)^3$ [S1].
*   $N$: Number of samples for 8-bit color normalization. Default: 255 [S1].
*   $I_{warmup}$: Warm-up iterations before aggregation begins. Default: 1,000 [S1].
*   $I_{total}$: Total training iterations. Default: 40,000 [S1].

### Definitions

**Pixel Coverage ($S_k$):**
The pixel coverage $S_k$ of a splatted 2D Gaussian is defined as the length of its horizontal or vertical axis until the opacity drops to the threshold $\sigma_T = 1/255$ [S1]. For a 2D Gaussian with parameter $a$ (where the Gaussian is $e^{-a(x-\mu)^2}$), the coverage $S$ is derived from:
$$ e^{-a(0.5S)^2} = \frac{1}{N} $$
Solving for $S$:
$$ S_k = \frac{2}{\sqrt{a}} \sqrt{\ln N} $$
This approximates the spatial extent of the Gaussian in pixels [S1].

**3dB Bandwidth:**
The bandwidth of a Gaussian with parameter $a$ is $f_{3dB} = \frac{1}{\pi} \sqrt{a \ln 2}$ [S1]. After aggregation and enlargement, the new parameter $a'$ becomes $\frac{\ln N}{S_T^2}$. The resulting bandwidth is:
$$ f'_{3dB} = \frac{1}{\pi} \sqrt{\ln N \cdot \ln 2} \approx 0.441 \, \text{px}^{-1} $$
This is below the Nyquist limit of $0.5 \, \text{px}^{-1}$, ensuring the aggregated Gaussian does not alias [S1].

### Training Phase

1.  **Initialization:** All 3D Gaussians initialized from the input point cloud belong to the finest level $l=1$. They are densified by splitting and cloning as in standard 3DGS [12], inheriting level $l=1$ [S1].
2.  **Warm-up:** For the first $I_{warmup}$ iterations, standard 3DGS training proceeds without multi-scale modifications [S1].
3.  **Aggregation (Post-Warm-up):** After iteration $I_{warmup}$, coarse-level Gaussians are introduced by aggregating fine-level Gaussians that are too small. This is performed periodically or at specific intervals [S1].
    *   **Step 1: Identify Small Gaussians.** For all levels $l_m$ from 2 to $l_{max}$, render all 3D Gaussians from levels $[1, l_m - 1]$ at a $4^{l_m - 1}$ times downsampled resolution for all training cameras. Calculate pixel coverage $S_k$ for each. Select Gaussians where $S_k < S_T$ [S1].
    *   **Step 2: Voxelization.** Bin the selected small Gaussians into a voxel grid of resolution $(400/l_m)^3$ based on their 3D positions [S1].
    *   **Step 3: Aggregation.** For each voxel containing small Gaussians, compute the average of their attributes: position, rotation, spherical harmonics features, opacity, and scaling. This creates a candidate "average" Gaussian [S1].
    *   **Step 4: Enlargement.** Calculate the average pixel coverage $S_{avg}$ of the small Gaussians in the voxel. Enlarge the scaling of the new aggregated Gaussian by a factor of $S_T / S_{avg}$. This ensures the new Gaussian has a pixel coverage of approximately $S_T$ at level $l_m$. Assign this new Gaussian to level $l_m$ [S1].
    *   **Step 5: Insertion.** Insert the new large Gaussians into the scene [S1].
4.  **Multi-Scale Training:** After large Gaussians are added, the model is trained using both the original high-resolution images and the downsampled images corresponding to the various levels [S1].
5.  **Range Update:** During training, if the rendering downsample scale equals the scale at which a Gaussian $G_k$ was created, update its coverage range using decay coefficients:
    $$ S_k^{max'} = \max(\lambda_1 S_k^{max}, S_k) $$
    $$ S_k^{min'} = \min(\lambda_2 S_k^{min}, S_k) $$
    where $\lambda_1 = 0.95$ and $\lambda_2 = 1.05$ [S1].

### Rendering Phase

1.  **Calculate Coverage:** For the target rendering scale $l_r$, calculate the pixel coverage $S_k$ for all Gaussians in the scene [S1].
2.  **Selection:** A Gaussian $G_k$ is selected for rendering if it satisfies the following condition:
    $$ \left( \frac{S_k}{S_k^{max}} \le S_{rel\_max} \right) \land \left( \frac{S_k}{S_k^{min}} \ge S_{rel\_min} \lor S_k \ge S_T \right) $$
    *   The first term filters out Gaussians that are too large (causing excessive blur/computation).
    *   The second term filters out Gaussians that are too small (causing aliasing), unless they are already larger than the absolute threshold $S_T$ [S1].
3.  **Render:** Render the selected subset of Gaussians using standard alpha blending [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants
ST = 2.0          # Selective rendering threshold (pixels)
N = 255.0         # 8-bit normalization
S_rel_max = 1.5   # Relative max coverage
S_rel_min = 0.5   # Relative min coverage
lambda1 = 0.95    # Decay for max coverage
lambda2 = 1.05    # Decay for min coverage
l_max = 4         # Number of detail levels
voxel_base = 400  # Base voxel resolution factor

def calculate_pixel_coverage(gaussian):
    """
    Calculates pixel coverage Sk for a 2D Gaussian.
    Assumes gaussian has attribute 'a' (inverse variance factor).
    Sk = (2 / sqrt(a)) * sqrt(ln(N))
    """
    a = gaussian.a
    if a <= 0:
        return 0.0
    return (2.0 / math.sqrt(a)) * math.sqrt(math.log(N))

def aggregate_small_gaussians(all_gaussians, current_iteration):
    """
    Algorithm 1: Aggregate Small Gaussians.
    Called periodically after warm-up (iteration > 1000).
    """
    if current_iteration <= 1000:
        return

    # Iterate through coarse levels to create
    for lm in range(2, l_max + 1):
        # Determine downsample scale for this level
        # Level 2 renders at 4x downsample, Level 3 at 16x, etc.
        scale_factor = 4 ** (lm - 1)
        
        # Collect candidates from finer levels [1, lm-1]
        candidates = []
        for Gk in all_gaussians:
            if Gk.level < lm:
                # Render at downsampled resolution to get Sk
                # Note: In implementation, this requires projecting to 
                # the specific downsampled camera space.
                Sk = calculate_pixel_coverage(Gk) 
                if Sk < ST:
                    candidates.append(Gk)
        
        if not candidates:
            continue

        # Voxelization
        voxel_size = voxel_base / lm
        voxel_grid = create_voxel_grid(voxel_size)
        
        for Gk in candidates:
            voxel_idx = voxel_grid.get_index(Gk.position)
            voxel_grid[voxel_idx].append(Gk)

        # Aggregation and Insertion
        for voxel_idx, group in voxel_grid.items():
            if not group:
                continue
            
            # Average Pooling
            new_pos = average([g.position for g in group])
            new_rot = average([g.rotation for g in group])
            new_opacity = average([g.opacity for g in group])
            new_sh = average([g.sh for g in group])
            new_scaling = average([g.scaling for g in group])
            
            # Calculate average coverage of the group
            S_avg = average([calculate_pixel_coverage(g) for g in group])
            
            # Enlarge scaling
            # The scaling needs to be increased such that the new coverage is ST
            # Since coverage is inversely proportional to sqrt(a) and scaling 
            # is related to 1/sqrt(a), we scale the size by ST/S_avg.
            enlargement_factor = ST / S_avg
            new_scaling = new_scaling * enlargement_factor
            
            # Create new Gaussian
            new_G = Gaussian(
                position=new_pos,
                rotation=new_rot,
                opacity=new_opacity,
                sh=new_sh,
                scaling=new_scaling,
                level=lm,
                Sk_max=ST, # Initial bounds
                Sk_min=ST
            )
            
            # Insert into scene
            all_gaussians.append(new_G)

def selective_rendering(all_gaussians, render_scale):
    """
    Algorithm 2: Selective Rendering Based on Pixel Coverage.
    """
    # 1. Calculate current pixel coverage for all Gaussians
    # This depends on the camera projection at render_scale
    for Gk in all_gaussians:
        Gk.Sk = calculate_pixel_coverage(Gk) # Simplified: assumes Sk is pre-calculated or projected
        
    # 2. Update ranges for Gaussians created at this scale
    # If render_scale matches the level creation scale, update bounds
    # (Implementation detail: map render_scale to level index)
    for Gk in all_gaussians:
        if Gk.level == get_level_for_scale(render_scale):
            Gk.Sk_max = max(lambda1 * Gk.Sk_max, Gk.Sk)
            Gk.Sk_min = min(lambda2 * Gk.Sk_min, Gk.Sk)

    # 3. Filter Gaussians
    selected_gaussians = []
    for Gk in all_gaussians:
        # Condition: (Sk / Sk_max <= S_rel_max) AND ( (Sk / Sk_min >= S_rel_min) OR (Sk >= ST) )
        cond1 = (Gk.Sk / Gk.Sk_max) <= S_rel_max
        cond2 = (Gk.Sk / Gk.Sk_min) >= S_rel_min
        cond3 = Gk.Sk >= ST
        
        if cond1 and (cond2 or cond3):
            selected_gaussians.append(Gk)
            
    # 4. Render
    return render_image(selected_gaussians, render_scale)
```

### Edge Cases and Pitfalls
*   **Missing Parts:** Naively filtering small Gaussians without aggregation causes holes in the rendered image because the low-frequency signal encoded by the small Gaussians is lost [S1].
*   **Performance Overhead:** All Gaussians must be splatted (projected) to calculate pixel coverage before filtering can occur. This prevents a linear speedup at very low resolutions, as the overhead of splatting remains high even if few Gaussians are finally rendered [S1].
*   **Precision:** The pixel coverage calculation relies on the opacity threshold $\sigma_T = 1/255$. Numerical stability in calculating $a$ from scaling parameters is required [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Training:** Training time is comparable to standard 3DGS (40,000 iterations) because the aggregation step is performed periodically and adds less than 5% to the total Gaussian count [S1].
*   **Rendering:** Rendering speed improves significantly at low resolutions. At 64x downsample, rendering time drops from ~100ms (3DGS) to ~5ms (MS 3DGS) [S1]. However, the speedup is not linear with resolution decrease because all Gaussians must be splatted to determine coverage [S1].
*   **Memory:** Memory usage increases slightly due to the storage of $S_k^{max}$, $S_k^{min}$, and the additional coarse-level Gaussians (<5% increase in count) [S1].

### Measured Performance
*   **360 Dataset (64x scale):** PSNR 25.35 vs 15.23 (3DGS), LPIPS 0.066 vs 0.256 (3DGS), Time 4.9ms vs 103.3ms (3DGS) [S1].
*   **Tank and Temples (64x scale):** PSNR 19.67 vs 13.88 (3DGS), LPIPS 0.034 vs 0.447 (3DGS), Time 5.9ms vs 82.6ms (3DGS) [S1].
*   **Deep Blending (64x scale):** PSNR 25.70 vs 17.75 (3DGS), LPIPS 0.036 vs 0.379 (3DGS), Time 3.4ms vs 59.7ms (3DGS) [S1].
*   **1x Scale:** Maintains comparable quality and speed to standard 3DGS [S1].

### Strengths and Limitations
*   **Strengths:** Eliminates aliasing artifacts at low resolutions; significantly reduces rendering time at low resolutions; preserves high-frequency detail at high resolutions.
*   **Limitations:** Requires periodic aggregation steps during training; rendering speed does not scale linearly with resolution reduction due to the splatting overhead; naive filtering causes missing geometry [S1].

## Background

### Origin
Multi-scale 3D Gaussian Splatting was introduced in the paper "Multi-Scale 3D Gaussian Splatting for Anti-Aliased Rendering" [S1]. It addresses the limitations of the original 3D Gaussian Splatting [12] regarding resolution independence.

### History and Development
Standard 3DGS optimizes Gaussians for a specific training resolution, leading to aliasing when rendered at different scales. Previous anti-aliasing methods like EWA splatting applied filters to individual primitives. MS 3DGS shifts the approach to a scene-level multi-scale representation, inspired by mipmapping techniques in texture rendering [S1]. It was evaluated on the MipNeRF-360, Tank and Temples, and Deep Blending datasets [S1].

## Sources

[S1] Multi-Scale 3D Gaussian Splatting for Anti-Aliased Rendering, https://doi.org/10.1109/cvpr52733.2024.01977, fetched 2026-10-26
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
