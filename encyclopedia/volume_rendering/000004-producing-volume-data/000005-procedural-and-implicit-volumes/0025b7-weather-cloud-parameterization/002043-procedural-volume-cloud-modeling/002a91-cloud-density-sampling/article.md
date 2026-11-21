# Cloud Density Sampling

Also called: Base Cloud Density Generation, Low-Frequency Cloud Density, SampleCloudDensity, Schneider's Cloud Noise

## Overview
Cloud Density Sampling generates the base volumetric density of clouds by combining low-frequency 3D noise textures using a weighted sum (Fractal Brownian Motion, FBM) and remapping the result to control cloud shape and coverage [S1, unofficial]. It differs from sibling methods like [Procedural 3D Worley-Perlin Fractal Noise Density](5267-procedural-3d-worley-perlin-fractal-noise-density.md) by relying on pre-baked multi-channel 3D textures rather than procedural noise functions, allowing for distinct control over vertical stratification and global distribution via texture channels [S1, unofficial]. It serves as the foundational density evaluation step in procedural volume cloud modeling, building on [Procedural Volume Cloud Modeling](2256-procedural-volume-cloud-modeling.md).

## How it works
**Inputs:**
- `samplepoint`: `vec3`, world or texture space coordinate for density evaluation.
- `weather_data`: `vec4`, containing cloud coverage in the red channel (`weather_data.r`) and potentially other atmospheric parameters.
- `relativeHeight`: `float`, normalized height of the sample point within the atmosphere.
- `wind_direction`: `vec3`, default `(1.0, 0.0, 0.0)`.
- `cloud_speed`: `float`, default `10.0`.
- `cloud_top_offset`: `float`, default `5.0`.
- `Time`: `float`, global time variable.
- `iscCheap`: `bool`, if `true`, skips high-frequency sampling.
- `lowFrequencyTexture`: `sampler3D`, precomputed noise texture.
- `highFrequencyTexture`: `sampler3D`, precomputed high-frequency noise texture (used if `!iscCheap`).
- `curlNoiseTexture`: `sampler2D`, 2D curl noise texture (used if `!iscCheap`).

**Outputs:**
- `final_cloud`: `float`, the computed cloud density at `samplepoint` in the range $[0.0, 1.0]$.

**Data Structures:**
- `lowFrequencyTexture`: A 3D texture where channels store specific noise components:
    - `R`: Base noise values.
    - `G`: Component weighted by 0.625 in FBM.
    - `B`: Component weighted by 0.250 in FBM.
    - `A`: Component weighted by 0.125 in FBM.
- `highFrequencyTexture`: A 3D texture with similar channel packing for high-frequency details:
    - `R`: Component weighted by 0.625.
    - `G`: Component weighted by 0.250.
    - `B`: Component weighted by 0.125.

**Formulas:**
The core remapping function is defined as:
$$ \text{Remap}(v, v_{min}, v_{max}, u_{min}, u_{max}) = u_{min} + \frac{v - v_{min}}{v_{max} - v_{min}}(u_{max} - u_{min}) $$

Low-frequency FBM is computed from the RGBA sample $N$ of the low-frequency texture:
$$ F_{low} = 0.625 N_g + 0.250 N_b + 0.125 N_a $$
$$ D_{base} = \text{Remap}(N_r, -(1.0 - F_{low}), 1.0, 0.0, 1.0) $$

High-frequency FBM (if used) is computed from the RGBA sample $H$ of the high-frequency texture:
$$ F_{high} = 0.625 H_r + 0.250 H_g + 0.125 H_b $$
$$ M_{high} = \text{mix}(F_{high}, 1.0 - F_{high}, \text{clamp}(relativeHeight \times 10.0, 0.0, 1.0)) $$

**Steps:**
1.  **Wind Skew:** Adjust the sample point based on wind and height:
    $$ \mathbf{p}_{skewed} = \mathbf{p} + relativeHeight \times \mathbf{wind\_direction} \times cloud\_top\_offset \times 0.005 $$
2.  **Time Movement:** Apply time-based drift:
    $$ \mathbf{p}_{final} = \mathbf{p}_{skewed} + (\mathbf{wind\_direction} + (0.0, 1.0, 0.0)) \times Time \times cloud\_speed \times 0.001 $$
3.  **Low-Frequency Sampling:** Sample the low-frequency texture at $\mathbf{p}_{final}$:
    $$ N = \text{SampleLowFrequencyTexture}(\mathbf{p}_{final}) $$
4.  **Low-Frequency FBM:** Compute the weighted sum:
    $$ F_{low} = N_g \times 0.625 + N_b \times 0.250 + N_a \times 0.125 $$
5.  **Base Cloud Remap:** Remap the red channel to create the base cloud shape:
    $$ base\_cloud = \text{Remap}(N_r, -(1.0 - F_{low}), 1.0, 0.0, 1.0) $$
6.  **Height Gradient:** Compute the vertical attenuation factor:
    $$ g_h = \text{GetDensityHeightGradientForPoint}(\mathbf{p}, \text{weather\_data}, relativeHeight) $$
7.  **Apply Height Gradient:**
    $$ base\_cloud \leftarrow base\_cloud \times g_h $$
8.  **Coverage Mask:** Retrieve cloud coverage from weather data:
    $$ C = \text{weather\_data}.r $$
9.  **Apply Coverage:** Remap the base cloud against the coverage mask:
    $$ base\_cloud\_with\_coverage = \text{Remap}(base\_cloud, C, 1.0, 0.0, 1.0) $$
10. **Clamp:** Ensure density is within bounds:
    $$ base\_cloud\_with\_coverage = \text{clamp}(base\_cloud\_with\_coverage, 0.0, 1.0) $$
11. **Aesthetic Enhancement:** Boost density based on coverage:
    $$ base\_cloud\_with\_coverage \leftarrow base\_cloud\_with\_coverage \times C \times 4.0 $$
12. **High-Frequency Details (if `!iscCheap` and `base_cloud_with_coverage > 0.0`):**
    a. Sample curl noise: $C_{noise} = \text{SampleCurlNoiseTexture}(\mathbf{p}_{final}.xy)$.
    b. Displace UVs: $\mathbf{p}_{final}.xy \leftarrow \mathbf{p}_{final}.xy + C_{noise}.xy \times (1.0 - relativeHeight)$.
    c. Sample high-frequency texture: $H = \text{SampleHighFrequencyTexture}(\mathbf{p}_{final} \times 0.1)$.
    d. Compute high-frequency modifier:
       $$ F_{high} = H_r \times 0.625 + H_g \times 0.250 + H_b \times 0.125 $$
       $$ M_{high} = \text{mix}(F_{high}, 1.0 - F_{high}, \text{clamp}(relativeHeight \times 10.0, 0.0, 1.0)) $$
    e. Erode by remapping:
       $$ final\_cloud = \text{Remap}(base\_cloud\_with\_coverage, 0.2 \times M_{high}, 1.0, 0.0, 1.0) $$
    f. Clamp against modifier:
       $$ final\_cloud = \text{clamp}(final\_cloud, 0.2 \times M_{high}, 1.0) $$
13. **Return:** If `iscCheap` or step 12 was skipped, return $base\_cloud\_with\_coverage$. Otherwise, return $final\_cloud$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def SampleCloudDensity(samplepoint, weather_data, relativeHeight, wind_direction, cloud_speed, cloud_top_offset, Time, isCheap, lowFrequencyTexture, highFrequencyTexture, curlNoiseTexture):
    # 1. Wind Skew
    p = samplepoint + relativeHeight * wind_direction * cloud_top_offset * 0.005
    
    # 2. Time Movement
    p = p + (wind_direction + vec3(0.0, 1.0, 0.0)) * Time * cloud_speed * 0.001
    
    # 3. Low-Frequency Sampling
    N = lowFrequencyTexture.sample(p) # Returns vec4 (RGBA)
    
    # 4. Low-Frequency FBM
    F_low = N.g * 0.625 + N.b * 0.250 + N.a * 0.125
    
    # 5. Base Cloud Remap
    # Remap(N.r, -(1.0 - F_low), 1.0, 0.0, 1.0)
    v_min = -(1.0 - F_low)
    v_max = 1.0
    u_min = 0.0
    u_max = 1.0
    base_cloud = u_min + (N.r - v_min) / (v_max - v_min) * (u_max - u_min)
    
    # 6. Height Gradient
    # GetDensityHeightGradientForPoint is a helper defined in parent or weather system
    g_h = GetDensityHeightGradientForPoint(samplepoint, weather_data, relativeHeight)
    
    # 7. Apply Height Gradient
    base_cloud *= g_h
    
    # 8. Coverage Mask
    C = weather_data.r
    
    # 9. Apply Coverage
    # Remap(base_cloud, C, 1.0, 0.0, 1.0)
    v_min = C
    v_max = 1.0
    base_cloud_with_coverage = u_min + (base_cloud - v_min) / (v_max - v_min) * (u_max - u_min)
    
    # 10. Clamp
    base_cloud_with_coverage = clamp(base_cloud_with_coverage, 0.0, 1.0)
    
    # 11. Aesthetic Enhancement
    base_cloud_with_coverage *= C * 4.0
    
    # 12. High-Frequency Details
    if not isCheap and base_cloud_with_coverage > 0.0:
        # 12a. Sample Curl Noise
        curl_noise = curlNoiseTexture.sample(vec2(p.x, p.z)) # Assuming XY mapping from source logic
        
        # 12b. Displace UVs
        p_xy = vec2(p.x, p.z) # Assuming 2D UVs for curl noise
        p_xy += curl_noise.xy * (1.0 - relativeHeight)
        
        # 12c. Sample High-Frequency Texture
        # Source says samplepoint * 0.1, but we displaced UVs. 
        # Assuming the displacement affects the UVs passed to the high-freq texture.
        # The source code snippet: high_frequency_noises = SampleHighFrequencyTexture(samplepoint * 0.1)
        # But step 12b displaces samplepoint.xy. We must use the displaced point.
        # Note: Source code snippet uses 'samplepoint' directly in the function call, 
        # but the step description says 'Displace UVs: samplepoint.xy += ...'.
        # We assume the displaced UVs are used for the high-freq sample.
        H = highFrequencyTexture.sample(p_xy * 0.1) # vec3 to vec2 conversion implied or p_xy is vec2
        
        # 12d. Compute High-Frequency Modifier
        F_high = H.r * 0.625 + H.g * 0.250 + H.b * 0.125
        mix_factor = clamp(relativeHeight * 10.0, 0.0, 1.0)
        M_high = mix(F_high, 1.0 - F_high, mix_factor)
        
        # 12e. Erode by Remapping
        # Remap(base_cloud_with_coverage, 0.2 * M_high, 1.0, 0.0, 1.0)
        v_min = 0.2 * M_high
        v_max = 1.0
        final_cloud = u_min + (base_cloud_with_coverage - v_min) / (v_max - v_min) * (u_max - u_min)
        
        # 12f. Clamp against modifier
        final_cloud = clamp(final_cloud, 0.2 * M_high, 1.0)
        
        return final_cloud
    else:
        return base_cloud_with_coverage
```

**Edge Cases and Pitfalls:**
-   **Precision:** The remapping formula involves division by $(v_{max} - v_{min})$. If $v_{max} \approx v_{min}$, precision loss or division by zero may occur. In step 5, if $F_{low} \approx 1.0$, $v_{min} \approx 0.0$, which is safe. In step 9, if $C \approx 1.0$, $v_{min} \approx 1.0$, which is safe. In step 12e, if $M_{high} \approx 0.5$, $v_{min} \approx 0.1$, which is safe.
-   **Boundaries:** The `clamp` operations ensure density stays within $[0.0, 1.0]$.
-   **Degenerate Input:** If `weather_data.r` (coverage) is 0, the base cloud is remapped to 0, and the aesthetic enhancement results in 0. High-frequency details are skipped if `base_cloud_with_coverage` is 0.

## Subtopics

### [Procedural 3D Worley-Perlin Fractal Noise Density](5267-procedural-3d-worley-perlin-fractal-noise-density.md)
Generates billowy cloud shapes by synthesizing cloud density $\rho(\mathbf{x})$ procedurally using inverted Worley fractal Brownian motion (FBM), vertical gradient attenuation, and coverage thresholding. It differs from standard Perlin-based cloud noise by using the distance-to-nearest-cell metric of Worley noise, inverted to create bright, puffy "billowy" structures characteristic of cumulus clouds, rather than the smooth, turbulent gradients of Perlin noise. Choose this when procedural generation without pre-baked textures is required and billowy shapes are desired.

### [Full Cloud Density Function](5875-full-cloud-density-function.md)
The Full Cloud Density Function is a procedural method for computing the volumetric density of clouds by combining a vertical atmospheric profile with horizontal noise-based structures and a global coverage mask. It differs from sibling methods like [Procedural 3D Worley-Perlin Fractal Noise Density](5267-procedural-3d-worley-perlin-fractal-noise-density.md) by explicitly separating the vertical stratification (via a texture) from the horizontal structural modulation (via noise and a cubemap). This separation allows for distinct control over cloud layering and global distribution. Choose this when explicit control over vertical and horizontal cloud structures is needed.

### Cone Tracing Cloud Sampling
Samples cloud density along a cone by iterating through a pre-generated noise kernel, transforming positions to weather map coordinates, and summing densities [S1, unofficial]. It differs from standard point sampling by accumulating density along a cone path, which can improve visual quality at the cost of multiple texture samples. Choose this when cone tracing is used for anti-aliasing or volumetric integration in cloud rendering.

## Operating characteristics
-   **Time Complexity:** $O(1)$ per sample point for low-frequency sampling. $O(1)$ for high-frequency sampling if enabled. Cone tracing variant is $O(k)$ where $k$ is the number of cone samples (e.g., 5).
-   **Memory Complexity:** Depends on the size of the pre-baked 3D textures (`lowFrequencyTexture`, `highFrequencyTexture`, `curlNoiseTexture`). Not given in the sources.
-   **Measured Performance:** Not given in the sources.
-   **Scaling:** Performance scales linearly with the number of sample points. High-frequency sampling doubles the texture lookups.
-   **Strengths:** Pre-baked textures allow for complex noise patterns without runtime computation of noise functions. Separation of low and high frequencies allows for LOD control.
-   **Limitations:** Requires pre-computation of textures. Less flexible than purely procedural methods.
-   **When to Use:** When high-quality, pre-defined cloud shapes are needed and texture memory is available.
-   **When Not to Use:** When procedural generation without pre-baked assets is required, or when memory is constrained.

## Background
Originated in the context of real-time volumetric cloud rendering, specifically in the work associated with "realtime-volumetric-cloudscapes" [S1, unofficial]. The method is attributed to Schneider's Cloud Noise, suggesting a connection to earlier work by Schneider on cloud rendering. The use of multi-channel textures for FBM is a technique to accelerate noise evaluation by baking the fractal sum into texture channels.

## Sources
[S1] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
