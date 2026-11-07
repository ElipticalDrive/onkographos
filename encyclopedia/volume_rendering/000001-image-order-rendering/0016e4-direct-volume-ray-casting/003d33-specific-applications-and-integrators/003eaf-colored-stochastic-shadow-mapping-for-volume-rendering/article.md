# Colored Stochastic Shadow Mapping for Volume Rendering

Also called: CSSM for DVR, Colored Stochastic Shadow Mapping (CSSM), Colored Stochastic Shadow Mapping (CSSM) for Direct Volume Rendering, Stochastic Shadow Mapping for Volume Rendering

## Overview
Colored Stochastic Shadow Mapping (CSSM) is a two-step rendering technique for Direct Volume Rendering (DVR) that generates colored shadows by adapting stochastic shadow mapping concepts to volumetric data [S1]. Unlike traditional shadow maps that use binary occlusion or deep shadow maps that store piecewise linear approximations, CSSM uses probabilistic events to represent photon traversal through a volume with changing opacities, allowing translucent objects to cast partial, color-accurate shadows [S1]. It sets itself apart by storing only one depth value per color channel per fragment (stochastic sample) rather than multiple control points, resulting in significantly lower memory usage than Deep Shadow Maps (DSM) while producing higher-quality colored shadows than Traditional Shadow Maps (TSM) [S1].

## How it works
CSSM operates in two distinct phases: CSSM Generation (building the shadow map) and DVR Rendering (compositing the volume with shadow factors). It builds on the fundamental direct volume ray casting loop defined in [Direct volume ray casting](19-direct-volume-ray-casting.md).

### Inputs and Outputs
**Inputs:**
- Volume dataset $V$ with opacity $\alpha$ and color properties.
- Light source position and direction (assumed single point light).
- CSSM texture resolution $R_{cssm} \times R_{cssm}$.
- Number of offset queries $m$ for denoising.

**Outputs:**
- CSSM texture: An RGB texture where each fragment stores $(c_R, c_G, c_B)$, representing the closest depth at which a photon of that wavelength was absorbed.
- Final DVR image with colored shadows.

### Data Structures
**CSSM Texture:**
- Format: RGB floating-point or fixed-point texture.
- Layout: Each fragment contains three values: $c_R, c_G, c_B$.
- Value Range: $[0, 1]$, normalized depth from light source.
- Initialization: $(1, 1, 1)$ for all fragments.

### Parameters
- $\alpha_i$: Opacity of the volume in segment $i$. Derived from the volume's transfer function or data values [S1].
- $t_{i,\lambda}$: Conditional probability that a photon of wavelength $\lambda$ is transmitted through the volume of segment $i$, given it interacts. Depends on color properties [S1].
- $m$: Number of offset queries. Typical value: 13 [S1]. Other tested values: 7 (pleasing results, no noticeable noise), 21 (softer shadows) [S1].
- $\Delta_j$: Spatial offsets for CSSM lookup. Example set for $m=13$: $\{(0,0), (\pm3,\pm3), (\pm4,0), (0,\pm4), (\pm7,0), (0,\pm7)\}$ [S1].
- Shadow map resolution: Tested resolutions include $1024^2$, $2048^2$, $4096^2$ [S1].

### Pseudocode

#### CSSM Generation
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize CSSM texture
for all fragments (u, v) of CSSM texture:
    cR, cG, cB = 1.0, 1.0, 1.0

    # Cast and discretize ray from light source through fragment (u, v) into the volume
    ray = cast_ray_from_light_through_fragment(u, v)
    
    # Iterate through each segment i of the discretized ray
    for i = 1 to n_segments:
        # Compute depth zi of segment i, normalized to [0, 1]
        zi = normalize_depth(ray.segment[i].distance)
        
        # Generate random number ri uniformly distributed in [0, 1]
        ri = random_uniform(0, 1)
        
        # Check if all color accumulators are already blocked
        if cR < 1.0 and cG < 1.0 and cB < 1.0:
            break
            
        # For each wavelength lambda in {R, G, B}
        for lambda in {R, G, B}:
            # Compute probability rho_i,lambda that photon is not transmitted through segment i
            # rho_i,lambda = alpha_i * (1 - t_i,lambda)
            alpha_i = get_opacity(ray.segment[i])
            t_i_lambda = get_transmission_probability(ray.segment[i], lambda)
            rho_i_lambda = alpha_i * (1.0 - t_i_lambda)
            
            # If probability exceeds random threshold, update depth accumulator
            if rho_i_lambda > ri:
                if lambda == R:
                    cR = min(cR, zi)
                elif lambda == G:
                    cG = min(cG, zi)
                elif lambda == B:
                    cB = min(cB, zi)
    
    # Store final accumulated depths in CSSM fragment
    set_fragment_color(u, v, cR, cG, cB)
```

#### DVR Rendering with CSSM
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# For all samples of the DVR approach
for each sample s in volume ray casting:
    # Initialize shadow factors for RGB channels
    lR, lG, lB = 0.0, 0.0, 0.0
    
    # Compute sample position (sx, sy, sz) in light coordinate system
    sx, sy, sz = project_to_light_space(s.position)
    
    # Iterate through m offset queries
    for j = 1 to m:
        # Read CSSM entries (cR, cG, cB) from fragment at (sx, sy) + Delta_j
        offset = offsets[j]
        cR_j, cG_j, cB_j = get_fragment_color(sx + offset.x, sy + offset.y)
        
        # For each wavelength lambda in {R, G, B}
        for lambda in {R, G, B}:
            # If CSSM depth > sample depth, sample is in shadow for this color
            if lambda == R:
                if cR_j > sz:
                    lR += 1.0 / m
            elif lambda == G:
                if cG_j > sz:
                    lG += 1.0 / m
            elif lambda == B:
                if cB_j > sz:
                    lB += 1.0 / m
    
    # Use shadow factor (lR, lG, lB) for illuminating sample
    # Multiply sample's emitted/transmitted color by shadow factor
    sample.color *= (lR, lG, lB)
```

### Edge Cases and Pitfalls
- **Noise:** Stochastic generation introduces noise. Mitigated by averaging over multiple CSSM lookups with offsets. Complex scenes may require more offsets (e.g., 21 instead of 13) to achieve pleasing results without noise [S1].
- **Blurring:** Increasing the number of offsets ($m$) softens shadows but increases computation time and causes blurring [S1].
- **Single Light Source:** Limited to a single point light source. Additional light sources require additional shadow maps, a general limitation of shadow mapping approaches [S1].
- **Precision:** Depth values are normalized to $[0, 1]$. Precision issues may arise if the volume depth range is very large or very small relative to the light source [G].
- **Boundary Conditions:** If a photon is not absorbed in any segment, the accumulator remains 1.0, indicating no shadow for that color at that depth [S1].

## Operating characteristics
- **Time Complexity:** CSSM generation is $O(R_{cssm}^2 \cdot n_{segments})$. DVR rendering is $O(N_{samples} \cdot m)$, where $N_{samples}$ is the number of volume samples and $m$ is the number of offsets.
- **Memory Complexity:** CSSM texture memory is $O(R_{cssm}^2 \cdot 3)$ (3 values per fragment). This is significantly less than Deep Shadow Maps (DSM), which store $O(R_{cssm}^2 \cdot 4 \cdot n_{nodes})$ (4 values per node, $n_{nodes}$ nodes per channel) [S1].
- **Measured Performance:** At $1024^2$ resolution, CSSM rendering time is 4.9 ms, higher than TSM (3.2 ms) and DSM (3.3 ms) due to increased texture fetches (13 per query for CSSM vs 1 per node for DSM (3 nodes) vs 1 for TSM) [S1]. Performance scales equally with resolution for all approaches [S1].
- **Strengths:** Produces higher-quality colored shadows than TSM and DSM. Uses less texture memory than DSM. Handles translucent objects correctly. Compatible with mixed volume and surface rendering [S1].
- **Limitations:** Limited to single point light source. Stochastic noise requires averaging. Blurring with increased offsets. Higher rendering time than TSM/DSM due to multiple texture fetches [S1].
- **When to Use:** When colored shadows for translucent volumes are required, and memory is a constraint compared to DSM. Suitable for interactive applications where slightly higher rendering time is acceptable for quality [S1].
- **When Not to Use:** When multiple light sources are needed (requires multiple shadow maps). When real-time performance is critical and TSM/DSM are sufficient. When sharp, non-blurred shadows are required (increasing offsets causes blurring) [S1].

## Background
CSSM was introduced as an alternative to deep shadow maps for direct volume rendering, aiming to produce higher-quality colored shadows with less texture memory [S1]. It adapts the CSSM concept originally developed for surface rendering to volumetric data by modifying probabilistic events to represent photon traversal through a volume with changing opacities [S1]. The method was implemented in the Voreen framework using a GPU-based ray casting approach and tested on synthetic datasets (Spheres) and real-world data (Walnut CT scan) [S1]. It is compatible with mixed scenes containing both volume and surface renderings, allowing CSSM to be generated for surfaces first, and then DVR to append to the same buffer without clearing [S1].

## Sources
[S1] Colored Stochastic Shadow Mapping for Direct Volume Rendering, https://doi.org/10.2312/eurovisshort.20181072, fetched 2026-10-26
