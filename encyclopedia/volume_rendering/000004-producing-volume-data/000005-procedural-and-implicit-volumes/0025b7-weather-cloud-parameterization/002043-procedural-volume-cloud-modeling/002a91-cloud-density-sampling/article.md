# Cloud Density Sampling

Also called: Base Cloud Density Generation, Low-Frequency Cloud Density, SampleCloudDensity, Schneider's Cloud Noise

## Overview
Cloud Density Sampling generates the base volumetric density of clouds by combining low-frequency 3D noise textures using a weighted sum (Fractal Brownian Motion, FBM) and remapping the result to control cloud shape and coverage [S1, unofficial]. It differs from its siblings by explicitly separating the generation of the base cloud structure (via low-frequency textures and height gradients) from high-frequency detail (via curl noise displacement and high-frequency textures) and by using a specific channel-weighted FBM from a single RGBA texture sample rather than procedural noise functions [S1, unofficial]. It builds on [Procedural Volume Cloud Modeling](2256-procedural-volume-cloud-modeling.md) by providing the specific evaluation routine for the density field.

## How it works
**Inputs:**
- `samplepoint`: `vec3`, world or texture space coordinate for density evaluation [S1, unofficial].
- `weather_data`: `vec4`, containing cloud coverage in the red channel (`weather_data.r`) [S1, unofficial].
- `relativeHeight`: `float`, normalized height of the sample point within the atmosphere [S1, unofficial].
- `iscCheap`: `bool`, if true, skips high-frequency sampling [S1, unofficial].
- `wind_direction`: `vec3`, default `(1.0, 0.0, 0.0)` [S1, unofficial].
- `cloud_speed`: `float`, default `10.0` [S1, unofficial].
- `cloud_top_offset`: `float`, default `5.0` [S1, unofficial].
- `Time`: `float`, global time variable [S1, unofficial].
- `lowFrequencyTexture`: `sampler3D`, precomputed noise texture [S1, unofficial].
- `highFrequencyTexture`: `sampler3D`, precomputed high-frequency noise texture [S1, unofficial].
- `curlNoiseTexture`: `sampler2D`, precomputed curl noise texture [S1, unofficial].

**Outputs:**
- `final_cloud`: `float`, the computed cloud density value in the range $[0.0, 1.0]$ (clamped) [S1, unofficial].

**Data Structures:**
- `lowFrequencyTexture`: A 3D texture where:
  - Channel R stores base noise values [S1, unofficial].
  - Channel G stores the component weighted by 0.625 [S1, unofficial].
  - Channel B stores the component weighted by 0.250 [S1, unofficial].
  - Channel A stores the component weighted by 0.125 [S1, unofficial].
- `highFrequencyTexture`: A 3D texture sampled at a scaled coordinate [S1, unofficial].
- `curlNoiseTexture`: A 2D texture sampled at the XY coordinates of the sample point [S1, unofficial].

**Formulas:**
The remapping function is defined as:
$$ \text{Remap}(v, v_{min}, v_{max}, u_{min}, u_{max}) = u_{min} + \frac{v - v_{min}}{v_{max} - v_{min}}(u_{max} - u_{min}) $$
[S1, unofficial]

The low-frequency FBM is computed from the RGBA sample $N$ of the low-frequency texture:
$$ F_{low} = 0.625 N_g + 0.250 N_b + 0.125 N_a $$
[S1, unofficial]

The base cloud density before height/coverage masking is:
$$ D_{base} = \text{Remap}(N_r, -(1.0 - F_{low}), 1.0, 0.0, 1.0) $$
[S1, unofficial]

The high-frequency FBM is computed from the RGBA sample $H$ of the high-frequency texture:
$$ F_{high} = 0.625 H_r + 0.250 H_g + 0.125 H_b $$
[S1, unofficial]

The high-frequency noise modifier is:
$$ M_{high} = \text{mix}(F_{high}, 1.0 - F_{high}, \text{clamp}(relativeHeight \times 10.0, 0.0, 1.0)) $$
[S1, unofficial]

**Steps:**
1. **Apply Wind Skew:** Adjust the sample point based on relative height and wind direction.
   $$ \mathbf{p}_{skewed} = \mathbf{p} + relativeHeight \times \mathbf{wind\_direction} \times cloud\_top\_offset \times 0.005 $$
   [S1, unofficial]
2. **Apply Time-Based Movement:** Displace the sample point by wind and vertical movement over time.
   $$ \mathbf{p}_{time} = \mathbf{p}_{skewed} + (\mathbf{wind\_direction} + (0.0, 1.0, 0.0)) \times Time \times cloud\_speed \times 0.001 $$
   [S1, unofficial]
3. **Sample Low-Frequency Texture:**
   $$ N = \text{SampleLowFrequencyTexture}(\mathbf{p}_{time}) $$
   [S1, unofficial]
4. **Compute Low-Frequency FBM:**
   $$ F_{low} = N_g \times 0.625 + N_b \times 0.250 + N_a \times 0.125 $$
   [S1, unofficial]
5. **Compute Base Cloud:**
   $$ D_{base} = \text{Remap}(N_r, -(1.0 - F_{low}), 1.0, 0.0, 1.0) $$
   [S1, unofficial]
6. **Apply Height Gradient:**
   $$ D_{height} = D_{base} \times \text{GetDensityHeightGradientForPoint}(\mathbf{p}_{time}, weather\_data, relativeHeight) $$
   [S1, unofficial]
7. **Apply Coverage Mask:**
   $$ D_{coverage} = \text{Remap}(D_{height}, weather\_data.r, 1.0, 0.0, 1.0) $$
   [S1, unofficial]
8. **Clamp and Enhance:**
   $$ D_{enhanced} = \text{clamp}(D_{coverage}, 0.0, 1.0) \times (weather\_data.r \times 4.0) $$
   [S1, unofficial]
9. **High-Frequency Details (if !iscCheap and $D_{enhanced} > 0.0$):**
   a. Sample Curl Noise:
      $$ C = \text{SampleCurlNoiseTexture}(\mathbf{p}_{time}.xy) $$
      [S1, unofficial]
   b. Displace UVs:
      $$ \mathbf{p}_{disp} = \mathbf{p}_{time} + C.xy \times (1.0 - relativeHeight) $$
      [S1, unofficial]
   c. Sample High-Frequency Texture:
      $$ H = \text{SampleHighFrequencyTexture}(\mathbf{p}_{disp} \times 0.1) $$
      [S1, unofficial]
   d. Compute High-Frequency FBM and Modifier:
      $$ F_{high} = H_r \times 0.625 + H_g \times 0.250 + H_b \times 0.125 $$
      [S1, unofficial]
      $$ M_{high} = \text{mix}(F_{high}, 1.0 - F_{high}, \text{clamp}(relativeHeight \times 10.0, 0.0, 1.0)) $$
      [S1, unofficial]
   e. Erode by Remapping:
      $$ D_{final} = \text{Remap}(\text{clamp}(D_{enhanced}, 0.2 \times M_{high}, 1.0), 0.2 \times M_{high}, 1.0, 0.0, 1.0) $$
      [S1, unofficial]
10. **Return:** $D_{final}$ if !iscCheap, else $D_{enhanced}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def SampleCloudDensity(samplepoint, weather_data, relativeHeight, ischeap, 
                       wind_direction, cloud_speed, cloud_top_offset, Time,
                       lowFrequencyTexture, highFrequencyTexture, curlNoiseTexture):
    # Step 1: Apply Wind Skew
    p = samplepoint + relativeHeight * wind_direction * cloud_top_offset * 0.005
    
    # Step 2: Apply Time-Based Movement
    p = p + (wind_direction + vec3(0.0, 1.0, 0.0)) * Time * cloud_speed * 0.001
    
    # Step 3: Sample Low-Frequency Texture
    N = lowFrequencyTexture.sample(p) # Returns vec4 (r, g, b, a)
    
    # Step 4: Compute Low-Frequency FBM
    low_freq_FBM = N.g * 0.625 + N.b * 0.250 + N.a * 0.125
    
    # Step 5: Compute Base Cloud
    # Remap(N.r, -(1.0 - low_freq_FBM), 1.0, 0.0, 1.0)
    v_min = -(1.0 - low_freq_FBM)
    v_max = 1.0
    u_min = 0.0
    u_max = 1.0
    base_cloud = u_min + (N.r - v_min) / (v_max - v_min) * (u_max - u_min)
    
    # Step 6: Apply Height Gradient
    # GetDensityHeightGradientForPoint is assumed to return a float multiplier
    density_height_gradient = GetDensityHeightGradientForPoint(p, weather_data, relativeHeight)
    base_cloud *= density_height_gradient
    
    # Step 7: Apply Coverage Mask
    cloud_coverage = weather_data.r
    # Remap(base_cloud, cloud_coverage, 1.0, 0.0, 1.0)
    v_min = cloud_coverage
    v_max = 1.0
    base_cloud_with_coverage = u_min + (base_cloud - v_min) / (v_max - v_min) * (u_max - u_min)
    
    # Step 8: Clamp and Enhance
    base_cloud_with_coverage = clamp(base_cloud_with_coverage, 0.0, 1.0)
    base_cloud_with_coverage *= cloud_coverage * 4.0
    
    # Step 9: High-Frequency Details
    if not ischeap and base_cloud_with_coverage > 0.0:
        # Sample Curl Noise
        curl_noise = curlNoiseTexture.sample(vec2(p.x, p.z)) # Assuming XY mapping as per source
        
        # Displace UVs
        p_disp = p
        p_disp.x += curl_noise.x * (1.0 - relativeHeight)
        p_disp.z += curl_noise.y * (1.0 - relativeHeight)
        
        # Sample High-Frequency Texture
        H = highFrequencyTexture.sample(p_disp * 0.1)
        
        # Compute High-Frequency FBM
        high_freq_FBM = H.r * 0.625 + H.g * 0.250 + H.b * 0.125
        
        # Compute High-Frequency Noise Modifier
        mix_factor = clamp(relativeHeight * 10.0, 0.0, 1.0)
        high_freq_noise_modifier = mix(high_freq_FBM, 1.0 - high_freq_FBM, mix_factor)
        
        # Erode by Remapping
        # Clamp base_cloud_with_coverage against modifier
        clamped_val = clamp(base_cloud_with_coverage, 0.2 * high_freq_noise_modifier, 1.0)
        # Remap
        v_min = 0.2 * high_freq_noise_modifier
        v_max = 1.0
        final_cloud = u_min + (clamped_val - v_min) / (v_max - v_min) * (u_max - u_min)
        return final_cloud
    else:
        return base_cloud_with_coverage
```
[S1, unofficial]

**Edge Cases and Pitfalls:**
- If `relativeHeight` is outside $[0.0, 1.0]$, the `clamp` operations in the modifier and coverage mask may produce unexpected results if not handled by the caller.
- The `Remap` function divides by $(v_{max} - v_{min})$. If $v_{max} = v_{min}$, division by zero occurs. In Step 5, $v_{max}=1.0$ and $v_{min}=-(1.0-F_{low})$. Since $F_{low} \in [0,1]$, $v_{min} \in [-1, 0]$, so $v_{max} \neq v_{min}$. In Step 7, $v_{min}=cloud\_coverage$. If `cloud_coverage` is 1.0, division by zero occurs. [G] Guard against this by returning 0.0 or the input value if $v_{max} \approx v_{min}$.
- The curl noise displacement uses `samplepoint.xy`. If the sample point is in world space, this assumes the curl noise texture is aligned with the XZ plane (or XY depending on coordinate system convention). The source uses `samplepoint.xy` [S1, unofficial].

## Subtopics

### [Procedural 3D Worley-Perlin Fractal Noise Density](5267-procedural-3d-worley-perlin-fractal-noise-density.md)
Generates billowy cloud shapes by synthesizing cloud density $\rho(\mathbf{x})$ procedurally using inverted Worley fractal Brownian motion (FBM), vertical gradient attenuation, and coverage thresholding. It differs from standard Perlin-based cloud noise by using the distance-to-nearest-cell metric of Worley noise, inverted to create bright, puffy "billowy" structures characteristic of cumulus clouds, rather than the smooth, turbulent gradients of Perlin noise. Choose this when simulating cumulus clouds with distinct, puffy tops. See [Procedural 3D Worley-Perlin Fractal Noise Density](5267-procedural-3d-worley-perlin-fractal-density.md).

### Cone Tracing Cloud Sampling
Samples cloud density along a cone by iterating through a pre-generated noise kernel, transforming positions to weather map coordinates, and summing densities. It differs from standard ray-marching by evaluating density along a cone spread to approximate volumetric scattering or shadowing effects more efficiently than a single ray. Choose this when approximating volumetric light transport or shadowing in cloud rendering. See [Cone Tracing Cloud Sampling](5267-cone-tracing-cloud-sampling.md).

### Full Cloud Density Function
Uses a combination of Perlin octaves, a vertical profile texture, and a cube map for global cloud cover to compute density. It differs from the base method by using a cube map for global coverage and a separate vertical profile function, rather than a single RGBA texture with weighted FBM. Choose this when global coverage maps and distinct vertical profiles are required. See [Full Cloud Density Function](5267-full-cloud-density-function.md).

## Operating characteristics
- **Time Complexity:** $O(1)$ per sample for the base method, assuming texture lookups are $O(1)$. The high-frequency branch adds constant overhead. Cone tracing is $O(k)$ where $k$ is the number of cone samples (e.g., 5) [S1, unofficial].
- **Memory Complexity:** Depends on the size of the precomputed textures (low-frequency, high-frequency, curl noise, weather data). Not given in the sources for specific resolutions, but [S2, unofficial] mentions 64x64x64 for noise and 512x512 for cover maps.
- **Measured Performance:** [S2, unofficial] reports 28.5 FPS on an AMD Ryzen 7 4700U at 640x480 resolution, using 3 deep opacity maps, one 512x512 offset layer, a 7x512x512 transmittance array, 5 octaves of 64x64x64 Worley noise for high-frequency detail, and 3 octaves of 64x64x64 Perlin-Worley noise for horizontal shapes.
- **Strengths:** Efficient evaluation using precomputed textures; flexible control over cloud shape via FBM weights and height gradients; supports time-based animation.
- **Limitations:** Requires precomputed textures; high-frequency details are skipped if `iscCheap` is true; coverage masking is simple and may not handle complex weather patterns without additional data.
- **When to Use:** Real-time cloud rendering where performance is critical and precomputed textures are available.
- **When Not to Use:** When procedural noise without textures is required, or when high-fidelity volumetric scattering is needed without approximation.

## Background
The method originates from the "realtime-volumetric-cloudscapes" project by erickTornero [S1, unofficial]. It is also associated with Jan Wedekind's work on procedural volumetric clouds [S2, unofficial]. The technique is used in real-time cloud rendering to generate dense, volumetric cloud fields efficiently.

## Sources
[S1] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
[S2] Procedural Volumetric Clouds · Jan Wedekind, https://www.wedesoft.de/software/2023/05/03/volumetric-clouds/, fetched 2026-10-28 (unofficial)
