# Colored Stochastic Shadow Mapping for Volume Rendering

Also called: CSSM for DVR, Colored Stochastic Shadow Mapping (CSSM), Colored Stochastic Shadow Mapping (CSSM) for Direct Volume Rendering, Stochastic Shadow Mapping for Volume Rendering

## Overview
Colored Stochastic Shadow Mapping (CSSM) is a two-step rendering technique for Direct Volume Rendering (DVR) that generates physically plausible, colored shadows by adapting stochastic shadow mapping concepts to volumetric data [S1]. Unlike traditional shadow maps that produce binary occlusion or deep shadow maps that require significant memory for piecewise linear approximations, CSSM stores a single stochastic depth sample per color channel, allowing translucent objects to cast partial, color-accurate shadows with minimal memory overhead [S1].

## How it works
CSSM operates in two distinct phases: CSSM Generation (building the shadow map) and DVR Rendering (compositing the volume with shadow factors) [S1].

### Inputs and Outputs
*   **Inputs**:
    *   Volumetric dataset $V$ with associated transfer functions for opacity $\alpha$ and transmission $t$.
    *   Light source position and direction (assumed to be a single point light for the base method) [S1].
    *   Camera parameters for the final DVR image [S1].
    *   CSSM resolution (e.g., $1024^2$, $2048^2$, $4096^2$) [S1].
    *   Number of offset queries $m$ (e.g., 7, 13, 21) [S1].
*   **Outputs**:
    *   CSSM Texture: An RGB texture where each fragment stores three depth values $(c_R, c_G, c_B)$ representing the stochastic absorption depth for each color channel [S1].
    *   Final DVR Image: The composited volume with shadowed illumination [S1].

### Data Structures
*   **CSSM Texture**: An RGB texture of resolution $W \times H$. Each texel stores three floating-point or fixed-point depth values: $c_R, c_G, c_B \in [0, 1]$ [S1].
    *   $c_\lambda = 1$ indicates no shadow (photon transmitted through the entire volume).
    *   $c_\lambda < 1$ indicates the normalized depth at which the photon was absorbed for wavelength $\lambda$ [S1].
*   **Offsets**: A predefined set of 2D integer offsets $\Delta_j$ used for stochastic averaging during the rendering phase [S1].

### CSSM Generation
This phase builds the CSSM texture by simulating photon traversal through the volume from the light source's perspective [S1].

1.  **Initialization**: For every fragment $(x, y)$ in the CSSM texture, initialize the depth accumulators for the RGB channels:
    $$ (c_R, c_G, c_B) \leftarrow (1, 1, 1) $$
2.  **Ray Casting**: Cast a ray from the light source through the current fragment $(x, y)$ into the volume. Discretize this ray into $n$ segments [S1].
3.  **Segment Iteration**: Iterate through each segment $i$ from $1$ to $n$:
    a.  Compute the normalized depth $z_i \in [0, 1]$ of segment $i$ along the ray [S1].
    b.  Generate a random number $r_i$ uniformly distributed in $[0, 1]$ [S1].
    c.  For each wavelength $\lambda \in \{R, G, B\}$:
        i.  Retrieve the opacity $\alpha_i$ of the volume at segment $i$.
        ii. Retrieve the conditional transmission probability $t_{i,\lambda}$ for wavelength $\lambda$ at segment $i$. This depends on the color properties of the volume [S1].
        iii. Compute the probability $\rho_{i,\lambda}$ that the photon is **not** transmitted through segment $i$:
        $$ \rho_{i,\lambda} = \alpha_i \cdot (1 - t_{i,\lambda}) $$
        *Derivation*: The probability of non-transmission is $1 - P(\text{not interacting}) - P(\text{interacting}) \cdot P(\text{transmitted} | \text{interacting}) = 1 - (1 - \alpha_i) - \alpha_i \cdot t_{i,\lambda} = \alpha_i(1 - t_{i,\lambda})$ [S1].
        iv. If $\rho_{i,\lambda} > r_i$, update the accumulator for channel $\lambda$:
        $$ c_\lambda \leftarrow \min(c_\lambda, z_i) $$
        This records the closest depth where absorption occurred for that color [S1].
    d.  **Early Termination**: If $c_R < 1$, $c_G < 1$, and $c_B < 1$, terminate the segment loop early, as the photon has been fully absorbed or blocked in previous segments [S1].
4.  **Storage**: Store the final accumulated depths $(c_R, c_G, c_B)$ in the CSSM fragment at $(x, y)$ [S1].

### DVR Rendering
This phase integrates the CSSM into the standard DVR ray casting loop to compute shadow factors [S1].

1.  **Initialization**: For every sample in the DVR ray, initialize shadow factors:
    $$ (l_R, l_G, l_B) \leftarrow (0, 0, 0) $$
2.  **Light Coordinates**: Compute the sample position $(s_x, s_y, s_z)$ in the light coordinate system [S1].
3.  **Stochastic Query**: Iterate through $m$ offset queries, for $j = 1$ to $m$:
    a.  Read the CSSM entries $(c_R, c_G, c_B)$ from the fragment located at $(s_x, s_y) + \Delta_j$, where $\Delta_j$ is the $j$-th offset [S1].
    b.  For each wavelength $\lambda \in \{R, G, B\}$:
        i.  Compare the CSSM depth $c_{\lambda,j}$ with the sample depth $s_z$.
        ii. If $c_{\lambda,j} > s_z$, the sample is in shadow for that color. Add $\frac{1}{m}$ to the shadow factor:
        $$ l_\lambda \leftarrow l_\lambda + \frac{1}{m} $$
        *Note*: If $c_{\lambda,j} \le s_z$, the sample is illuminated, and the factor remains unchanged (effectively adding 0) [S1].
4.  **Application**: Use the computed shadow factors $(l_R, l_G, l_B)$ to modulate the illumination of the volumetric sample during compositing [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CSSM Generation
def generate_cssm(volume, light, cssm_texture, resolution):
    # Initialize CSSM texture with 1.0 (no shadow)
    for x, y in cssm_texture:
        cssm_texture[x, y] = (1.0, 1.0, 1.0)

    # Cast rays from light through each fragment
    for x, y in cssm_texture:
        ray = cast_ray_from_light(light, x, y, resolution)
        segments = discretize_ray(ray)
        
        cR, cG, cB = 1.0, 1.0, 1.0
        
        for i, segment in enumerate(segments):
            # Early termination if all colors are shadowed
            if cR < 1.0 and cG < 1.0 and cB < 1.0:
                break
                
            z_i = segment.normalized_depth
            r_i = random_uniform(0, 1)
            
            for lambda in [R, G, B]:
                alpha_i = volume.get_opacity(segment.position)
                t_i_lambda = volume.get_transmission_probability(segment.position, lambda)
                
                # Probability photon is NOT transmitted
                rho_i_lambda = alpha_i * (1.0 - t_i_lambda)
                
                if rho_i_lambda > r_i:
                    c_lambda = min(c_lambda, z_i)
        
        cssm_texture[x, y] = (cR, cG, cB)

# DVR Rendering with CSSM
def render_dvr_with_cssm(volume, camera, cssm_texture, offsets):
    image = empty_image(camera.resolution)
    
    for each ray from camera:
        samples = cast_ray_through_volume(ray, volume)
        
        for sample in samples:
            # Initialize shadow factors
            lR, lG, lB = 0.0, 0.0, 0.0
            
            # Convert sample position to light coordinates
            sx, sy, sz = sample.position_in_light_space
            
            # Average over m offsets
            for delta in offsets:
                # Lookup CSSM with offset
                cx, cy = sx + delta.x, sy + delta.y
                cR, cG, cB = cssm_texture.lookup(cx, cy)
                
                # Check if sample is in shadow for each channel
                if cR > sz:
                    lR += 1.0 / len(offsets)
                if cG > sz:
                    lG += 1.0 / len(offsets)
                if cB > sz:
                    lB += 1.0 / len(offsets)
            
            # Apply shadow factors to illumination
            illuminated_color = sample.color * (lR, lG, lB)
            composite(image, sample.position, illuminated_color)
```

### Parameters and Typical Values
*   **$\alpha_i$**: Opacity of the volume in segment $i$. Derived from the volume's transfer function [S1].
*   **$t_{i,\lambda}$**: Conditional probability that a photon of wavelength $\lambda$ is transmitted through the volume segment $i$, given it interacts. Depends on color properties [S1].
*   **$m$**: Number of offset queries for denoising. Typical values: 7 (pleasing results, no noticeable noise), 13 (used in experiments), or 21 (softer shadows) [S1].
*   **$\Delta_j$**: Spatial offsets. Example set for $m=13$: $\{(0,0), (\pm3,\pm3), (\pm4,0), (0,\pm4), (\pm7,0), (0,\pm7)\}$ [S1].
*   **CSSM Resolution**: Tested resolutions include $1024^2$, $2048^2$, and $4096^2$. Higher resolution improves shadow detail but increases memory [S1].

### Edge Cases and Pitfalls
*   **Noise**: Stochastic generation introduces noise. Mitigated by averaging over multiple CSSM lookups with offsets. Complex scenes may require more offsets to achieve noise-free results [S1].
*   **Blurring**: Increasing $m$ softens shadows but increases computation time and causes blurring [S1].
*   **Single Light Source**: The method is limited to a single point light source. Additional lights require additional shadow maps, which is a general limitation of shadow mapping [S1].
*   **Precision**: Depth values are normalized to $[0, 1]$. Precision issues may arise if the volume depth range is very large relative to the CSSM resolution [S1].

## Operating characteristics
*   **Time Complexity**: CSSM generation is $O(W \cdot H \cdot n)$, where $W, H$ are CSSM dimensions and $n$ is the number of segments per ray. Rendering is $O(N \cdot m)$, where $N$ is the number of volumetric samples and $m$ is the number of offsets [S1].
*   **Memory Complexity**: CSSM stores 3 values per fragment (one per color channel). This is significantly less than Deep Shadow Maps (DSM), which store 4 values per node per channel. For 3 nodes, DSM uses 12 values per fragment, 4x the memory of CSSM [S1].
*   **Measured Performance**: At $1024^2$ resolution, CSSM rendering time is 4.9 ms, compared to 3.2 ms for Traditional Shadow Mapping (TSM) and 3.3 ms for DSM. The higher cost is due to 13 texture fetches per query (vs. 1 for TSM, 3 for DSM) [S1].
*   **Strengths**: Produces higher-quality colored shadows than TSM and DSM; uses less memory than DSM; handles translucent objects correctly; compatible with mixed volume/surface scenes [S1].
*   **Limitations**: Limited to single point light; noisy without sufficient offsets; blurring with high offset counts; performance scales with offset count [S1].
*   **When to Use**: When colored shadows for translucent volumes are required and memory is constrained, or when higher quality than TSM is needed without the memory cost of DSM [S1].
*   **When Not to Use**: When multiple light sources are required (without significant memory overhead for multiple CSSMs); when real-time performance is critical and noise/blur from offsets is unacceptable [S1].

## Background
CSSM was proposed as an alternative to deep shadow maps for direct volume rendering, aiming to produce higher-quality colored shadows with less texture memory [S1]. It adapts the CSSM concept, originally developed for surface rendering, to volumetric data by modifying probabilistic events to represent photon traversal through a volume with changing opacities [S1]. The method was implemented in the Voreen framework and tested on synthetic datasets and a Walnut CT scan [S1]. It builds on the core DVR loop defined in [Direct volume ray casting](19-direct-volume-ray-casting.md), specifically diverging in the compositing stage by incorporating probabilistic occlusion factors [S1, S9816].

## Sources
[S1] Colored Stochastic Shadow Mapping for Direct Volume Rendering, https://doi.org/10.2312/eurovisshort.20181072, fetched 2026-10-26
