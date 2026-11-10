# Colored Stochastic Shadow Mapping for Volume Rendering

Also called: CSSM for DVR, Colored Stochastic Shadow Mapping (CSSM), Colored Stochastic Shadow Mapping (CSSM) for Direct Volume Rendering, Stochastic Shadow Mapping for Volume Rendering

## Overview
Colored Stochastic Shadow Mapping (CSSM) is a two-step rendering technique for Direct Volume Rendering (DVR) that generates physically plausible, colored shadows for translucent volumetric objects [S1]. It solves the limitation of traditional shadow mapping, which produces binary, often incorrect dark shadows for translucent volumes, by using stochastic sampling to record the depth at which photons of specific wavelengths are absorbed or scattered [S1]. Unlike [Deep Shadow Mapping](9473-deep-shadow-mapping.md), which stores piecewise linear approximations, CSSM stores a single stochastic depth value per color channel, offering a memory-efficient alternative that produces softer, color-distinct shadows [S1].

## How it works
The method consists of two distinct phases: CSSM Generation (building the shadow map) and DVR Rendering (compositing the volume with shadow factors).

### Inputs and Outputs
*   **Inputs:**
    *   Volumetric dataset $V$ with opacity $\alpha$ and color/transmission properties.
    *   Light source position and direction (assumed to be a single point light for the standard formulation).
    *   Camera parameters for the DVR ray casting.
    *   CSSM texture resolution (e.g., $1024^2$, $2048^2$).
    *   Number of offset queries $m$ for denoising (typical value: 13).
*   **Outputs:**
    *   CSSM Texture: An RGB texture where each fragment stores three depth values $(c_R, c_G, c_B)$.
    *   Final Image: Volume rendered image with colored shadows applied.

### Data Structures
*   **CSSM Texture:** An RGB texture. Each fragment (texel) stores three floating-point or fixed-point depth values:
    *   $c_R$: Depth of the closest segment where red photons were not transmitted.
    *   $c_G$: Depth of the closest segment where green photons were not transmitted.
    *   $c_B$: Depth of the closest segment where blue photons were not transmitted.
    *   Initial value for all accumulators is 1.0 (representing "no shadow" or maximum distance).

### Pseudocode: CSSM Generation
This step builds the shadow map by simulating photon traversal through the volume from the light source.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_cssm(volume, light_source, cssm_texture, resolution):
    """
    Generates the Colored Stochastic Shadow Map.
    volume: The volumetric dataset, accessible via ray intersection.
    light_source: Position and direction of the light.
    cssm_texture: The output RGB texture of size resolution x resolution.
    """
    # Iterate over all fragments (pixels) of the shadow map
    for y in range(resolution):
        for x in range(resolution):
            # Step 1: Initialize depth accumulators for RGB channels
            c_R = 1.0
            c_G = 1.0
            c_B = 1.0
            
            # Step 2: Cast and discretize a ray from the light source through the current fragment (x, y)
            # into the volume. This ray represents the path of a photon.
            ray = cast_ray_from_light(light_source, x, y, resolution)
            
            # Step 3: Iterate through each segment i of the discretized ray
            for i in range(1, num_segments + 1):
                # Step 4: Compute the depth z_i of segment i, normalized to [0, 1]
                z_i = compute_normalized_depth(ray, i)
                
                # Step 5: Generate a random number r_i uniformly distributed in [0, 1]
                r_i = random_uniform(0.0, 1.0)
                
                # Step 6 & 7: For each wavelength lambda in {R, G, B}
                for lambda in ['R', 'G', 'B']:
                    # Get opacity alpha_i for segment i
                    alpha_i = get_opacity(volume, ray, i)
                    
                    # Get conditional transmission probability t_{i, lambda}
                    # This depends on the color properties of the volume segment.
                    t_i_lambda = get_transmission_probability(volume, ray, i, lambda)
                    
                    # Compute probability rho_{i, lambda} that the photon is NOT transmitted
                    # Formula: p_{i, lambda} = alpha_i * (1 - t_{i, lambda})
                    rho_i_lambda = alpha_i * (1.0 - t_i_lambda)
                    
                    # If the probability of non-transmission exceeds the random threshold,
                    # the photon is considered absorbed/blocked at this segment.
                    if rho_i_lambda > r_i:
                        # Update the depth accumulator for channel lambda
                        # Store the minimum depth (closest to light) where absorption occurred
                        if lambda == 'R':
                            c_R = min(c_R, z_i)
                        elif lambda == 'G':
                            c_G = min(c_G, z_i)
                        elif lambda == 'B':
                            c_B = min(c_B, z_i)
                
                # Step 8: Terminate early if all color accumulators are < 1
                # This indicates the photon was fully absorbed or blocked in previous segments.
                if c_R < 1.0 and c_G < 1.0 and c_B < 1.0:
                    break
            
            # Step 9: Store the final accumulated depths in the CSSM fragment
            cssm_texture.set_pixel(x, y, (c_R, c_G, c_B))
```

### Pseudocode: DVR Rendering with CSSM
This step integrates the shadow map into the standard DVR ray casting loop.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_dvr_with_cssm(volume, camera, cssm_texture, offsets, m):
    """
    Renders the volume using the pre-generated CSSM for shadows.
    volume: The volumetric dataset.
    camera: Camera parameters for ray casting.
    cssm_texture: The input CSSM texture.
    offsets: List of m 2D offset vectors (dx, dy).
    m: Number of offset queries.
    """
    # Initialize output image
    image = create_image(camera.resolution)
    
    # Iterate over all pixels in the image
    for y in range(camera.resolution.y):
        for x in range(camera.resolution.x):
            # Standard DVR ray casting setup
            ray = cast_ray_from_camera(camera, x, y)
            
            # Initialize color and opacity accumulators for DVR
            color_acc = (0.0, 0.0, 0.0)
            opacity_acc = 0.0
            
            # Iterate through samples along the ray
            for sample in ray.samples:
                # Step 10: Initialize shadow factors for RGB channels
                l_R = 0.0
                l_G = 0.0
                l_B = 0.0
                
                # Step 11: Compute sample position (sx, sy, sz) in light coordinate system
                # This projects the 3D sample point onto the 2D CSSM texture plane.
                sx, sy = project_to_light_space(sample.position, camera, light_source)
                sz = sample.depth_normalized # Depth of the current sample from light
                
                # Step 12: Iterate through m offset queries
                for j in range(m):
                    # Step 13: Read CSSM entries from fragment at (sx, sy) + offset_j
                    offset = offsets[j]
                    c_R, c_G, c_B = cssm_texture.get_pixel(sx + offset.dx, sy + offset.dy)
                    
                    # Step 14: Compare sample depth sz with stored CSSM depths
                    # If CSSM depth > sample depth, the sample is in shadow for that color
                    if c_R > sz:
                        l_R += 1.0 / m
                    if c_G > sz:
                        l_G += 1.0 / m
                    if c_B > sz:
                        l_B += 1.0 / m
                
                # Shadow factors l_R, l_G, l_B are now computed
                # Apply shadow factors to the sample's contribution
                sample_color = get_sample_color(volume, sample)
                sample_opacity = get_sample_opacity(volume, sample)
                
                # Composite with shadowing
                # Note: Standard DVR compositing formula:
                # C_out = C_in * (1 - alpha_out) + C_sample * alpha_sample * shadow_factor
                # Here, shadow_factor is applied to the emitted/transmitted light.
                
                shadowed_color = (
                    sample_color.R * l_R,
                    sample_color.G * l_G,
                    sample_color.B * l_B
                )
                
                # Standard front-to-back compositing
                alpha_out = opacity_acc + (1.0 - opacity_acc) * sample_opacity
                color_acc = color_acc * (1.0 - (1.0 - opacity_acc) * sample_opacity) + shadowed_color * (1.0 - opacity_acc) * sample_opacity
                opacity_acc = alpha_out
                
                # Early termination if fully opaque
                if opacity_acc >= 1.0:
                    break
            
            image.set_pixel(x, y, color_acc)
    
    return image
```

### Parameters and Formulas
*   **Opacity $\alpha_i$**: The opacity of the volume in segment $i$. Derived from the volume's transfer function or data values [S1].
*   **Conditional Transmission Probability $t_{i,\lambda}$**: The probability that a photon of wavelength $\lambda$ is transmitted through the volume of segment $i$, given it interacts. Depends on color properties [S1].
*   **Non-Transmission Probability $\rho_{i,\lambda}$**: The probability that the photon of wavelength $\lambda$ is *not* transmitted through segment $i$.
    $$ \rho_{i,\lambda} = \alpha_i \cdot (1 - t_{i,\lambda}) $$
    This is derived from $1 - (P(\overline{V}_i) + P(V_i) \cdot P(T_{i,\lambda} | V_i))$, where $P(\overline{V}_i) = 1 - \alpha_i$ and $P(V_i) = \alpha_i$ [S1].
*   **Shadow Factor $l_\lambda$**: The average of $m$ binary tests for color channel $\lambda$.
    $$ l_\lambda = \frac{1}{m} \sum_{j=1}^{m} \mathbb{I}(c_{\lambda,j} > s_z) $$
    where $c_{\lambda,j}$ is the CSSM depth at offset $j$ and $s_z$ is the sample depth [S1].
*   **Offsets $\Delta_j$**: Spatial offsets applied to CSSM lookup coordinates. For $m=13$, a typical set is $\{(0,0), (\pm3,\pm3), (\pm4,0), (0,\pm4), (\pm7,0), (0,\pm7)\}$ [S1].
*   **Number of Offsets $m$**: Typical value used in experiments is 13. 7 offsets may be sufficient for simple scenes with no noticeable noise; 21 offsets produce softer shadows but increase computation time and blurring [S1].

### Edge Cases and Pitfalls
*   **Noise**: Stochastic generation introduces noise in the shadow map. This is mitigated by averaging over multiple CSSM lookups with offsets. Complex scenes may require more offsets to achieve pleasing results without noise [S1].
*   **Blurring**: Increasing the number of offsets $m$ softens shadows but increases computation time and causes blurring [S1].
*   **Single Light Source**: The method is limited to a single point light source. Additional light sources require additional shadow maps, which is a general limitation of shadow mapping approaches [S1].
*   **Memory vs. Quality**: Higher CSSM resolution improves shadow detail but increases memory usage. Typical tested resolutions include $1024^2$, $2048^2$, and $4096^2$ [S1].

## Operating characteristics
*   **Time Complexity**: The CSSM generation step scales with the number of fragments in the shadow map and the number of segments per ray. The DVR rendering step scales with the number of pixels and samples per ray, plus the cost of $m$ texture fetches per sample.
*   **Memory Complexity**: CSSM uses significantly less memory than Deep Shadow Maps (DSM). CSSM stores one color buffer (3 values per fragment), while DSM with 3 nodes per channel stores 12 values per fragment (4x storage). DSM memory scales linearly with nodes (+4 values per node), whereas CSSM memory is constant regardless of offset count [S1].
*   **Measured Performance**: At $1024^2$ resolution, CSSM rendering time is approximately 4.9 ms, which is higher than Traditional Shadow Mapping (TSM, 3.2 ms) and DSM (3.3 ms) due to increased texture fetches (13 per query for CSSM vs 1 per node for DSM (3 nodes) vs 1 for TSM) [S1].
*   **Strengths**: Produces higher-quality colored shadows with less texture memory than DSM. Handles translucent objects correctly, creating partial, colored shadows, unlike TSM which uses a threshold at opacity 0.5, leading to dark, incorrect shadows in the interior of translucent objects [S1].
*   **Limitations**: Limited to a single point light source. Stochastic noise requires averaging, which can blur shadows if too many offsets are used. Performance is slightly higher than TSM/DSM due to multiple texture fetches [S1].
*   **When to Use**: When colored shadows for translucent volumes are required, and memory is a constraint compared to DSM. Suitable for interactive applications where slightly higher rendering time is acceptable for improved visual quality.
*   **When Not to Use**: When multiple light sources are needed (requires multiple shadow maps). When strict binary shadowing is acceptable and performance is critical (use TSM). When memory is extremely constrained and shadow quality is secondary (use TSM).

## Background
Colored Stochastic Shadow Mapping (CSSM) was originally developed for surface rendering [ME11] and was adapted for Direct Volume Rendering (DVR) in a 2018 paper [S1]. The adaptation modifies the probabilistic events to represent photon traversal through a volume with changing opacities, adopting the computations of the discrete volume rendering integral. The method was implemented in the Voreen framework [Vor] and tested on synthetic datasets (Spheres) and real-world medical data (Walnut CT dataset) [S1]. It provides an alternative to Deep Shadow Maps, offering better color representation and lower memory usage [S1].

## Sources
[S1] Colored Stochastic Shadow Mapping for Direct Volume Rendering, https://doi.org/10.2312/eurovisshort.20181072, fetched 2026-10-26
