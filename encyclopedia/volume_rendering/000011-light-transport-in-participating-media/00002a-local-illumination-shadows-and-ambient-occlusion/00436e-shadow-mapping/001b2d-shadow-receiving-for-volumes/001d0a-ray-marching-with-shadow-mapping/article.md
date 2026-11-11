# Ray Marching with Shadow Mapping

Also called: Deep Opacity Maps, OSM, Opacity Shadow Map Volume Rendering, Scene Object Integration with Shadow Mapping

## Overview
Ray Marching with Shadow Mapping is a technique for computing self-shadowing in participating media by precomputing transmittance values from the light's perspective into a 3D texture, which is then sampled during the view-dependent ray march [S2, unofficial][S3, unofficial]. It solves the problem of efficiently determining how much light reaches a sample point through a volume without performing a secondary ray march toward the light for every sample [S3, unofficial]. It differs from [Shadow Volume Rendering](1556-shadow-volume-rendering.md) by using a 2D-depth-based 3D texture stack rather than a full 3D geometric volume, and from [Deep Shadow Maps](2166-deep-shadow-maps.md) by storing discrete transmittance layers rather than hierarchical density curves [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volume density function $D(x)$, extinction coefficient $\sigma_t$, scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$, light direction, light camera matrix, shadow volume range $[z_{min}, z_{max}]$, step count, step size.
*   **Outputs:** Illuminated volume color (RGB) and final transmittance (A) per pixel.

**Data Structures:**
1.  **Opacity Shadow Map:** A 3D texture storing transmittance values.
    *   Dimensions: $W \times H \times L$, where $W, H$ are the shadow map resolution and $L$ is the number of layers (samples along the light ray).
    *   Format: Single channel (R) or multiple channels if MRT is not used for layer separation. [S2, unofficial]
    *   Note: Storing different layers in different color channels (e.g., Layer 1 in R, Layer 2 in G) prevents correct bilinear/trilinear interpolation during sampling [S2, unofficial].
2.  **Light Camera:** Orthographic for directional lights, perspective for point lights [S2, unofficial].

**Preprocessing (Shadow Map Generation):**
1.  Render the volume from the light source's perspective.
2.  Ray march along the light direction (Z-axis in light space) from $z_{min}$ to $z_{max}$ with constant step size $\Delta z$.
3.  For each layer $i$ from $0$ to $L-1$:
    *   Calculate position: $pos_i = pos_{start} + i \cdot \Delta z$.
    *   Sample density: $d_i = D(pos_i)$.
    *   Accumulate optical depth and compute transmittance using Beer's Law.
    *   Store transmittance in the 3D texture at coordinate $(u, v, i/L)$.

**Main Rendering Pass (Ray Marching):**
1.  For each pixel, cast a view ray.
2.  Initialize `transmittance` = 1.0, `illumination` = $(0,0,0)$.
3.  Loop for $i$ from $0$ to `stepCount`:
    *   Advance sample position: $x = x_{start} + i \cdot \text{stepSize} \cdot \text{marchDirection}$.
    *   **Shadow Lookup:**
        *   Transform $x$ to light space: $x_{light} = M_{light} \cdot x$.
        *   Normalize depth to $[0, 1]$: $z = \frac{-x_{light}.z - z_{min}}{z_{max} - z_{min}}$. [S2, unofficial]
        *   Normalize UV: $uv = \frac{x_{light}.xy}{\text{lightCamSize}} \cdot 0.5 + 0.5$. [S2, unofficial]
        *   Sample transmittance: $T_{shadow} = \text{texture}(\text{opacityShadowMap}, (uv, z)).r$. [S2, unofficial]
    *   **Lighting Calculation:**
        *   Sample density: $d = D(x)$.
        *   Compute extinction: $\sigma_t = \sigma_a + \sigma_s$.
        *   Update volume transmittance: $T_{vol} = T_{vol} \cdot \exp(-\sigma_t \cdot d \cdot \text{stepSize})$.
        *   Compute in-scattering: $L_{in} = L_{ambient} + L_{direct} \cdot \Phi(v, -l) \cdot T_{shadow}$.
        *   Compute out-scattering: $L_{out} = \sigma_s \cdot d$.
        *   Accumulate illumination: $I = I + T_{vol} \cdot L_{in} \cdot L_{out} \cdot \text{stepSize}$.
4.  Return $(I, T_{vol})$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing: Generate Opacity Shadow Map
def generate_opacity_shadow_map(light_cam_matrix, shadow_volume_range, slice_count, density_func, extinction_coef):
    """
    Generates a 3D texture storing transmittance layers.
    shadow_volume_range: (z_min, z_max)
    slice_count: Number of layers (L)
    """
    z_min, z_max = shadow_volume_range
    step_size = (z_max - z_min) / slice_count
    
    # Initialize 3D texture of size (W, H, slice_count)
    shadow_map = Texture3D(width=512, height=512, depth=slice_count)
    
    for layer in range(slice_count):
        # Calculate position along the light ray for this layer
        # In light space, we march along Z. 
        # Note: The source code implies marching backwards from z_min or forwards.
        # Based on [S2, unofficial], we accumulate density.
        
        current_optical_depth = 0.0
        
        # We need to store transmittance at each layer. 
        # Transmittance T(z) = exp(-integral_0^z sigma_t * D(x) dx)
        # However, the source code [S2, unofficial] accumulates total density and applies Beer's law 
        # for the *current* step's contribution to the layer's value? 
        # Actually, the source code calculates layer[i] as exp(-total_density * step * coef).
        # This implies layer[i] stores the transmittance from the start (z_min) to layer i.
        
        total_density = 0.0
        for k in range(slice_count):
            # Position for layer k
            pos_z = z_min + k * step_size
            # Assuming density function takes world or light space pos. 
            # Here we assume density_func returns density at the specific 3D point.
            # For simplicity, we assume we are marching through a volume defined in light space.
            pos = vec3(0, 0, pos_z) # Simplified 1D march for density sampling logic
            d = density_func(pos)
            total_density += d
            
            # Transmittance from start to this point
            t = exp(-total_density * step_size * extinction_coef)
            
            # Store in texture. 
            # Note: [S2, unofficial] uses MRT to write multiple layers. 
            # Here we simulate writing to the 3D texture.
            shadow_map.set(layer, t)
            
    return shadow_map

# Main Rendering: Ray Marching with Shadow Map
def raymarch_with_shadow_map(view_ray_origin, view_ray_dir, step_count, step_size, 
                             light_cam_matrix, shadow_map, shadow_volume_range,
                             ambient_light, direct_light, scattering_coef, extinction_coef,
                             phase_func):
    """
    Renders the volume using precomputed shadow map.
    """
    z_min, z_max = shadow_volume_range
    light_cam_size = light_cam_matrix.get_width() # Or specific uniform
    
    transmittance = 1.0
    illumination = vec3(0.0)
    
    current_pos = view_ray_origin
    
    for i in range(step_count):
        # Advance sample position
        current_pos += view_ray_dir * step_size
        
        # 1. Shadow Lookup
        # Transform to light space
        light_pos = light_cam_matrix * vec4(current_pos, 1.0)
        
        # Normalize Z to [0, 1]
        # Note: Source [S2, unofficial] uses -shadowMapPosition.z. This assumes Z is negative in view space 
        # or the light camera has a specific projection.
        z = (-light_pos.z - z_min) / (z_max - z_min)
        
        # Clamp Z to avoid out-of-bounds sampling
        z = clamp(z, 0.0, 1.0)
        
        # Normalize UV
        # Source [S2, unofficial]: uv = shadowMapPosition.xy / lightCamSize / 2.0 + vec2(0.5)
        uv = light_pos.xy / light_cam_size / 2.0 + vec2(0.5)
        
        # Sample transmittance from 3D texture
        shadow_factor = shadow_map.sample(uv, z).r
        
        # 2. Volume Integration
        current_density = density_func(current_pos)
        
        # Update volume transmittance (Beer's Law)
        transmittance *= exp(-current_density * extinction_coef * step_size)
        
        # Compute In-Scattering
        # L_in = Ambient + Direct * Phase * Shadow
        # Note: phase_func(v, -l) computes phase function for view vector v and light direction -l
        phase = phase_func(view_ray_dir, -view_ray_dir) # Simplified direction assumption
        in_scattering = ambient_light + direct_light * phase * shadow_factor
        
        # Compute Out-Scattering
        out_scattering = scattering_coef * current_density
        
        # Accumulate Illumination
        current_light = in_scattering * out_scattering
        illumination += transmittance * current_light * step_size
        
    return vec4(illumination, transmittance)
```

**Parameters:**
*   $z_{min}, z_{max}$: Define the volume range for slicing the shadow map. If $z_{min}$ is too large, the shadow map slice starts inside the volume, making the side facing the light incorrectly dark [S2, unofficial].
*   $k = k_{Schlick}$: Anisotropy parameter for Schlick's approximation phase function [S2, unofficial].
*   $\alpha$ (azimuth), $\beta$ (elevation): Control directional light direction [S2, unofficial].
*   Step Size: Can be jittered with a small random offset to hide visible banding caused by low layer counts, at the cost of adding noise [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Banding:** Visible bands may occur due to a low number of layers in the opacity shadow map. Solution: Add a small random offset in each step of the ray marcher to create visual noise, which can be blurred if too noticeable [S2, unofficial].
*   **Channel Storage:** Storing different layer values in different color channels (e.g., Layer 1 in Red, Layer 2 in Green) prevents correct bilinear/trilinear interpolation during sampling. Store all layers in a single channel or use separate passes [S2, unofficial].
*   **Depth Range:** Incorrect $z_{min}$ causes incorrect shading [S2, unofficial].

## Subtopics

### [Shadow Volume Rendering](1556-shadow-volume-rendering.md)
Precomputes a 3D texture of transmittance from each voxel toward the light. Decouples shadow calculation from the view-dependent ray march, trading memory bandwidth for reduced compute during the final image pass. Choose when view-independent shadow accuracy is prioritized over memory usage.

### [Volume Shadow Calculation via Light Strength Buffer](1914-volume-shadow-calculation-via-light-strength-buffer.md)
Stores shadow information in a 3D light strength buffer instead of a 2D shadow depth buffer. Allows correct rendering of penumbras and semi-transparent object shadows. Choose when continuous light attenuation values are needed rather than binary depth comparisons.

### [Deep Shadow Maps](2166-deep-shadow-maps.md)
Stores a curve of volume density (or accumulated alpha) as a function of distance in each pixel. Offers a more compact representation than storing full alpha values per layer. Choose when memory is constrained and hierarchical approximation is acceptable.

### [Light Injection Pass](2759-light-injection-pass.md)
Post-processing technique for volumetric light scattering. Uses pre-existing shadow maps for occlusion. Choose for adding light shafts/glow effects in post-process rather than direct volume integration.

## Operating characteristics
*   **Time Complexity:** Preprocessing is $O(W \cdot H \cdot L)$ where $L$ is the number of layers. Rendering is $O(N)$ per pixel for $N$ steps, with $O(1)$ shadow lookup cost.
*   **Memory Complexity:** $O(W \cdot H \cdot L)$ for the 3D opacity shadow map. [S2, unofficial]
*   **Performance:** On an AMD Ryzen 7 4700U with 640x480 resolution, 3 deep opacity maps (512x512 offset, 7x512x512 transmittance array) achieved 28.5 FPS [S3, unofficial].
*   **Strengths:** Efficient self-shadowing without per-sample light ray marching. Supports interpolation for smooth shadows.
*   **Limitations:** Limited by the number of layers (MRT limits to 8+ layers per pass). Aliasing if interpolation is not used or if layers are sparse. Requires precomputation for static lights.

## Background
Originated as a method for efficient self-shadowing in volumetric rendering, specifically clouds and participating media [S3, unofficial]. It builds on standard shadow mapping by extending the depth buffer to store transmittance values along the light ray [S2, unofficial]. The technique is often referred to as "Deep Opacity Maps" in the context of cloud rendering [S3, unofficial].

## Sources
[S1] Gradient‐based Interpolation and Sampling for Real‐time Rendering of Inhomogeneous, Single‐scattering Media, https://doi.org/10.1111/j.1467-8659.2008.01343.x, fetched 2026-10-27
[S2] Volumetric Rendering - Computer Graphics Learning, https://cglearn.eu/pub/advanced-computer-graphics/volumetric-rendering, fetched 2026-10-28 (unofficial)
[S3] Procedural Volumetric Clouds · Jan Wedekind, https://www.wedesoft.de/software/2023/05/03/volumetric-clouds/, fetched 2026-10-28 (unofficial)
