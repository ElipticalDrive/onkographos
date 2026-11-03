# Ray Marching with Shadow Mapping

Also called: Deep Opacity Maps, OSM, Opacity Shadow Map Volume Rendering

## Overview
Ray Marching with Shadow Mapping is a technique for efficient self-shadowing in volume rendering that uses an opacity shadow map to determine occlusion by the volume itself [S1]. It solves the problem of expensive per-view-ray light sampling by precomputing transmittance values from the light's perspective, allowing the renderer to multiply interpolated values from this map by colors from the transfer function during the main pass [S2]. Unlike [Shadow Volume Rendering](1556-shadow-volume-rendering.md), which explicitly constructs a volumetric shadow buffer, this method uses a 2D depth map combined with a 3D array of opacity images sampled from the light source's perspective [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volume density function $density(\mathbf{p})$, absorption coefficient $k_a$, scattering coefficient $k_s$, light direction (azimuth $\alpha$, elevation $\beta$), shadow volume range $[z_{min}, z_{max}]$, step size $\Delta z$, number of layers/slices $N$.
*   **Outputs:** A 3D texture (opacity shadow map) storing transmittance values, and a final rendered image with self-shadowed illumination.

**Data Structures:**
The opacity shadow map consists of:
1.  A depth (offset) image storing the distance to the cloud start from the light source [S2].
2.  A 3D texture array storing transmittance values. The dimensions are $W \times H \times N$, where $W, H$ are the shadow map resolution and $N$ is the number of depth slices [S2].
    *   *Constraint:* All layers must be stored in a single channel (e.g., Red) to allow correct bilinear/trilinear interpolation. Storing different layers in different color channels (e.g., Layer 1 in Red, Layer 2 in Green) prevents correct interpolation [S1].
    *   *MRT Limit:* If using Multiple Render Targets (MRT) to write layers, the minimum count is 8. If $N > 8$, multiple passes are required [S1].

**Parameters:**
*   $z_{min}, z_{max}$: Define the volume range for slicing the shadow map. If $z_{min}$ is too large, the shadow map slice starts inside the volume, making the side facing the light incorrectly too dark [S1].
*   $k = k_{Schlick}$: Anisotropy parameter for Schlick's approximation phase function [S1].
*   $\alpha$: Azimuth angle controlling directional light direction around the y-axis ($0^\circ$ is center-right, $90^\circ$ is behind) [S1].
*   $\beta$: Elevation angle controlling directional light direction around -z ($0^\circ$ is center-right, $90^\circ$ is top) [S1].
*   Step jitter: A small random offset added to the step size to hide visible banding caused by low layer counts, at the cost of adding noise [S1].

**Steps:**

1.  **Render Opacity Shadow Map (Light Pass):**
    1.  Render the scene from the light source's perspective. Use an orthographic camera for directional lights and a perspective camera for point lights [S1].
    2.  Ray march with even steps of $\Delta z$ through an encapsulating volume from $z_{min}$ to $z_{max}$ [S1].
    3.  For each sample position, compute the density and update the accumulated transmittance using Beer's Law:
        $$ T_{new} = T_{old} \cdot \exp(-\rho \cdot (k_a + k_s) \cdot \Delta z) $$
        where $\rho$ is the current density, and $k_a, k_s$ are absorption and scattering coefficients [S1].
    4.  Store the transmittance values for each depth layer into the 3D texture. If $N \le 8$, use MRT to write all layers in one pass. Otherwise, use multiple passes [S1].

2.  **Main Rendering Pass (View Pass):**
    1.  Initialize transmittance $T = 1.0$ and illumination $I = \vec{0}$ [S1].
    2.  Loop for $i$ from $0$ to $stepCount$:
        a.  Advance sample position: $\mathbf{p} \leftarrow \mathbf{p} + \mathbf{d} \cdot \Delta z$, where $\mathbf{d}$ is the march direction [S1].
        b.  Sample current density $\rho = density(\mathbf{p})$ [S1].
        c.  Update volume transmittance:
            $$ T \leftarrow T \cdot \exp(-\rho \cdot (k_a + k_s) \cdot \Delta z) $$
        d.  Compute in-scattering. First, determine the shadow factor from the opacity shadow map:
            i.  Transform $\mathbf{p}$ to shadow map space: $\mathbf{p}_{light} = M_{light} \cdot \mathbf{p}$ [S1].
            ii. Normalize depth $z$ within the shadow volume range:
                $$ z = \frac{-\mathbf{p}_{light}.z - z_{min}}{z_{max} - z_{min}} $$
            iii. Compute UV coordinates:
                $$ uv = \frac{\mathbf{p}_{light}.xy}{W} + \vec{0.5} $$
                *(Note: Source [S1] formula uses `lightCamSize / 2.0`, which is equivalent to normalizing by width $W$ if the camera is centered, but standard shadow map UV is $[0,1]$)* [S1].
            iv. Sample the 3D texture:
                $$ shadowFactor = texture(opacityShadowMap, vec3(uv, z)).r $$
        e.  Compute in-scattering:
            $$ I_{in} = L_{ambient} + L_{direct} \cdot phase(\mathbf{v}, -\mathbf{l}) \cdot shadowFactor $$
            where $phase(\mathbf{v}, -\mathbf{l})$ is the Schlick phase function [S1].
        f.  Compute out-scattering:
            $$ I_{out} = k_s \cdot \rho $$
        g.  Accumulate illumination:
            $$ I \leftarrow I + T \cdot I_{in} \cdot I_{out} \cdot \Delta z $$
    3.  Return $\vec{4}(I, T)$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader for Light Pass (Generating Opacity Shadow Map)
# Uses MRT to write up to 8 layers per pass
def render_opacity_shadow_map(light_cam_matrix, shadow_volume_range, slice_count, density_func, absorption_coef, scattering_coef):
    extinction_coef = absorption_coef + scattering_coef
    step_size = (shadow_volume_range.y - shadow_volume_range.x) / slice_count
    
    # Initialize layer transmittances
    layers = [0.0] * 8
    
    # Get local position in camera space
    # Note: In GLSL, this is typically done via vertex attributes or uniforms
    local_position = get_local_position() 
    
    total_density = 0.0
    
    for i in range(slice_count):
        # Sample density at current depth slice
        # The sample position is offset from the origin along the light direction
        sample_pos = local_position - (i * step_size) * vec3(0.0, 0.0, 1.0) # Assuming light is along -Z in camera space
        current_density = density_func(sample_pos)
        total_density += current_density
        
        # Beer's Law for transmittance
        # T = exp(-sigma_t * density * distance)
        # Note: The source code accumulates density and applies step_size once per layer
        # This implies density is treated as a constant over the step or averaged
        transmittance = exp(-total_density * step_size * extinction_coef)
        
        if i < 8:
            layers[i] = transmittance
            
    # Write to MRTs
    # outColor0 = vec4(layers[0], 0, 0, 0)
    # outColor1 = vec4(layers[1], 0, 0, 0)
    # ...
    return layers

# Fragment Shader for Main Pass (Ray Marching with Shadow Lookup)
def raymarch(sample_position, march_direction, step_count, step_size, light_cam_matrix, shadow_volume_range, opacity_shadow_map, ambient_light, direct_light, scattering_coef, absorption_coef):
    extinction_coef = absorption_coef + scattering_coef
    transmittance = 1.0
    illumination = vec3(0.0)
    
    for i in range(step_count):
        # Advance sample position
        sample_position += march_direction * step_size
        
        # Sample density
        current_density = density_func(sample_position)
        
        # Update transmittance
        transmittance *= exp(-current_density * extinction_coef * step_size)
        
        # Compute shadow factor from opacity shadow map
        shadow_map_position = (light_cam_matrix * vec4(sample_position, 1.0)).xyz
        z = (-shadow_map_position.z - shadow_volume_range.x) / (shadow_volume_range.y - shadow_volume_range.x)
        uv = shadow_map_position.xy / light_cam_size / 2.0 + vec2(0.5)
        shadow_factor = texture(opacity_shadow_map, vec3(uv, z)).r
        
        # Compute in-scattering
        # phase(v, -l) is Schlick's approximation
        in_scattering = ambient_light + direct_light * phase_function(v, -l, k) * shadow_factor
        
        # Compute out-scattering
        out_scattering = scattering_coef * current_density
        
        # Accumulate illumination
        current_light = in_scattering * out_scattering
        illumination += transmittance * current_light * step_size
        
    return vec4(illumination, transmittance)
```

## Subtopics

### [Shadow Volume Rendering](1556-shadow-volume-rendering.md)
Shadow Volume Rendering is a specialized ray marching routine that computes only the accumulated alpha (transmittance) along a ray to generate shadow maps or occlusion values for volume scenes. It precomputes a shadow volume to store the amount of light arriving at each sample after attenuation by intervening material, allowing the renderer to multiply interpolated values from this map by colors from the transfer function during the main pass. Unlike [Ray Marching with Shadow Mapping](1415-ray-marching-with-shadow-mapping.md), which uses an opacity shadow map for occlusion, this method explicitly constructs a volumetric shadow buffer.

### [Volume Shadow Calculation via Light Strength Buffer](1914-volume-shadow-calculation-via-light-strength-buffer.md)
This method adds shadows to mixtures of geometric and volumetric data by storing shadow information in a 3D light strength buffer instead of a 2D shadow depth buffer. It sets itself apart from its sibling [Ray Marching with Shadow Mapping](1415-ray-marching-with-shadow-mapping.md) by computing a fractional light strength at every point in space, which allows it to correctly render penumbras and semi-transparent shadows without the aliasing artifacts associated with Williams' 2D shadow depth buffer.

### [Light Injection Pass](2759-light-injection-pass.md)
The Light Injection Pass is a post-processing technique that adds volumetric light scattering (such as sun shafts and lamp glow) to a rendered scene by marching rays from the camera through the volume to sample light sources and their occlusion. It solves the problem of visualizing light interacting with participating media (air/fog) in real-time without full path tracing, relying on pre-existing shadow maps for occlusion. Unlike [Shadow Volume Rendering](1556-shadow-volume-rendering.md) which uses geometric volumes, or [Volume Shadow Calculation via Light Strength Buffer](1914-volume-shadow-calculation-via-light-strength-buffer.md) which may use different accumulation buffers, this method specifically targets direct illumination scattering using a ray-marching approach over a depth buffer, executed as a distinct post-process step.

### Deep Shadow Maps
Deep Shadow Maps are a shadow mapping technique where each pixel stores a curve representing volume density as a function of distance, rather than just the depth to the nearest object [S3]. The algorithm computes a shadow map per light in the scene, storing a curve in each pixel that represents how transmission through the volume changes as one moves through it [S3]. Unlike standard shadow maps which store depth to the nearest object, deep shadow maps store volume density information [S3]. It is mentioned for completeness and historical reasons, with similarities to baking lighting into voxels [S3]. Developed in 2000 by Tom Lokovic and Eric Veach from Pixar [S3].

## Operating characteristics
*   **Time Complexity:** The light pass requires $O(W \cdot H \cdot N)$ samples, where $W, H$ are shadow map dimensions and $N$ is the number of layers. The main pass requires $O(R \cdot S)$ samples, where $R$ is the number of rays and $S$ is the number of steps per ray. The shadow lookup is $O(1)$ via texture sampling.
*   **Memory Complexity:** The opacity shadow map requires $O(W \cdot H \cdot N)$ memory. For a $512 \times 512$ map with 7 layers, this is significant but manageable [S2].
*   **Measured Performance:** In an example using 28.5 FPS on an AMD Ryzen 7 4700U at 640x480 resolution, 3 deep opacity maps with a 7x512x512 transmittance array were rendered per frame [S2].
*   **Strengths:** More efficient than sampling light rays towards the light source for each sample of the view ray [S2]. Avoids the need for per-view-ray light sampling [S2].
*   **Limitations:** Visible banding can occur due to the low number of layers [S1]. Storing layers in different color channels prevents correct interpolation [S1]. If $z_{min}$ is too large, shadows are incorrect [S1].
*   **When to use:** When self-shadowing is required for volumes and performance is a concern compared to full light sampling.
*   **When not to use:** When high-quality penumbras are required (consider [Volume Shadow Calculation via Light Strength Buffer](1914-volume-shadow-calculation-via-light-strength-buffer.md)) or when the volume geometry is highly irregular and non-uniform sampling is needed (consider [Deep Opacity Maps](1415-ray-marching-with-shadow-mapping.md) variant).

## Background
The technique was developed as an alternative to per-view-ray light sampling for volume self-shadowing. It builds on the concept of shadow mapping, extending it to record transparency (transmittance) along different depths from the light's perspective [S1]. The term "Opacity Shadow Maps" is used to describe this specific implementation using a 3D texture of transmittance values [S1]. Related historical techniques include Deep Shadow Maps by Tom Lokovic and Eric Veach (2000) [S3].

## Sources
[S1] Volumetric Rendering - Computer Graphics Learning, https://cglearn.eu/pub/advanced-computer-graphics/volumetric-rendering, fetched 2026-10-28 (unofficial)
[S2] Procedural Volumetric Clouds · Jan Wedekind, https://www.wedesoft.de/software/2023/05/03/volumetric-clouds/, fetched 2026-10-28 (unofficial)
[S3] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
