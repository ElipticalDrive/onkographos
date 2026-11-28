# Half-Angle Slice Rendering

Also called: Half-Angle Shadowing, Half-Angle Slicing Volume Rendering, Half-Angle Slicing for Volumetric Shadows

## Overview
Half-Angle Slice Rendering is a technique for computing volumetric shadows and local illumination in participating media, specifically designed for particle systems and voxel-based atmospheric data [S1][S2, unofficial]. It solves the problem of efficiently accumulating shadowing information while rendering volume slices by projecting the volume onto a single 2D shadow buffer [S2, unofficial]. Unlike methods that require complex 3D texture lookups or separate light-space rendering passes, this method calculates a "half-angle" vector between the view and light directions, allowing the same set of slices to be composited into an eye buffer (for final image accumulation) and a light buffer (for shadow attenuation) [S1][S2, unofficial]. It is distinct from voxel-based volume rendering approaches that require 3D textures or octrees, as it relies only on particle sorting and a single 2D shadow buffer [S2, unofficial].

## How it works
The method operates on a set of particles or a voxel grid, rendering them as a series of slices perpendicular to a calculated half-angle vector [S1][S2, unofficial]. It requires an eye buffer for final image accumulation and a light buffer (the single 2D shadow buffer) for storing transmittance and accumulated opacity [S1][S2, unofficial]. The light buffer must include an alpha channel to store accumulated opacity for the "under" blending operator [S2, unofficial].

### Inputs and Outputs
*   **Inputs:** Particle positions $\mathbf{p}_i$, view direction $\mathbf{v}$, light direction $\mathbf{l}$, number of slices $N_{slices}$, particle count $N_{particles}$ [S2, unofficial].
*   **Outputs:** Final color in Eye Buffer, Transmittance/Shadow map in Light Buffer [S1][S2, unofficial].

### Data Structures
1.  **Particle List:** Array of structures containing position $\mathbf{p}_i$ and color/opacity attributes [S2, unofficial].
2.  **Sort Keys:** Array of floats $k_i = \mathbf{h} \cdot \mathbf{p}_i$, where $\mathbf{h}$ is the half-angle vector [S2, unofficial].
3.  **Render Targets:**
    *   `image_buffer`: Color + Alpha [S2, unofficial].
    *   `light_buffer`: Color (or single channel) + Alpha [S1][S2, unofficial].

### Algorithm Steps

1.  **Calculate Half-Angle Vector:**
    Determine the direction perpendicular to the slices. Let $\mathbf{v}$ be the normalized view vector (from eye to scene) and $\mathbf{l}$ be the normalized light vector (from scene to light) [S2, unofficial].
    $$ \mathbf{h} = \begin{cases} \text{normalize}(\mathbf{v} + \mathbf{l}) & \text{if } \mathbf{v} \cdot \mathbf{l} > 0 \\ \text{normalize}(-\mathbf{v} + \mathbf{l}) & \text{if } \mathbf{v} \cdot \mathbf{l} \le 0 \end{cases} $$
    The source contains a typo (`> 1.0`); the correct condition for normalized vectors is `> 0.0` [G]. If $\mathbf{v} \cdot \mathbf{l} \le 0$, the camera points towards the light, requiring a flip in rendering order [S2, unofficial].

2.  **Sort Particles:**
    Compute the projection of each particle onto $\mathbf{h}$:
    $$ k_i = \mathbf{h} \cdot \mathbf{p}_i $$
    Sort particles based on $k_i$ in ascending order (front-to-back relative to $\mathbf{h}$) [S2, unofficial].

3.  **Initialize Buffers:**
    Clear `image_buffer` and `light_buffer` to zero [S2, unofficial].

4.  **Pre-pass (Atmospheric Inscattering Estimate):**
    Render a single slice halfway through the volume extent (covering the entire image) to estimate atmospheric inscattering between the eye and the volume [S1]. This step is specific to voxel-based atmospheric rendering [S1].

5.  **Iterative Slicing:**
    Divide the sorted particles into $N_{slices}$ batches [S2, unofficial]. For each slice index $i$ from $0$ to $N_{slices}-1$:
    
    a. **Render to Eye Buffer:**
       Draw the current batch of particles into `image_buffer` [S2, unofficial].
       *   **Blending:** Use the "under" operator: `src = one_minus_dest_alpha`, `dst = one` [S2, unofficial].
       *   **Color Pre-multiplication:** The pixel shader must pre-multiply color by alpha: $\mathbf{c}_{out} = \mathbf{c}_{in} \cdot \alpha_{in}$ [S2, unofficial].
       *   **Shadowing:** Sample the `light_buffer` from the *previous* slice pass (or initial clear for $i=0$) to attenuate the incoming light for the current slice [S2, unofficial].
       
    b. **Render to Light Buffer:**
       Draw the *same* batch of particles into `light_buffer` [S2, unofficial].
       *   **Blending:** Use the "under" operator to accumulate opacity/attenuation [S2, unofficial].
       *   This pass calculates the amount of light arriving at the next slice [S2, unofficial].

6.  **Post-pass (Background Atmosphere):**
    Approximate the atmosphere behind the volume by calculating inscattering from the plane halfway through the volume onward and compositing it into the image [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def half_angle_slice_rendering(particles, view_dir, light_dir, num_slices, image_buffer, light_buffer):
    """
    Renders volumetric shadows using half-angle slicing.
    
    Args:
        particles: List of dicts with 'pos' (vec3), 'color' (vec3), 'alpha' (float).
        view_dir: Normalized view vector (eye to scene).
        light_dir: Normalized light vector (scene to light).
        num_slices: Number of slices to divide particles into.
        image_buffer: Render target for final image.
        light_buffer: Render target for shadow attenuation.
    """
    # 1. Calculate Half-Angle Vector
    dot_prod = math.dot(view_dir, light_dir)
    if dot_prod > 0.0:
        half_vec = math.normalize(view_dir + light_dir)
        flipped = False
    else:
        half_vec = math.normalize(-view_dir + light_dir)
        flipped = True

    # 2. Sort Particles
    sort_keys = []
    for p in particles:
        key = math.dot(half_vec, p['pos'])
        sort_keys.append((key, p))
    
    sort_keys.sort(key=lambda x: x[0])
    sorted_particles = [p for _, p in sort_keys]

    # 3. Clear Buffers
    image_buffer.clear()
    light_buffer.clear()

    # 4. Pre-pass (Optional, for voxel atmospheres)
    # render_halfway_slice(image_buffer) 

    # 5. Iterative Slicing
    num_particles = len(sorted_particles)
    batch_size = num_particles // num_slices
    
    for i in range(num_slices):
        start_idx = i * batch_size
        end_idx = start_idx + batch_size
        if i == num_slices - 1:
            end_idx = num_particles
            
        current_batch = sorted_particles[start_idx:end_idx]
        
        # 5a. Render to Eye Buffer
        render_slice_to_buffer(current_batch, image_buffer, light_buffer, prev_light_pass=True)
        
        # 5b. Render to Light Buffer
        render_slice_to_buffer(current_batch, light_buffer, None, prev_light_pass=False)

    # 6. Post-pass
    # composite_background_atmosphere(image_buffer)

def project_to_screen(pos, view_proj_matrix):
    """Standard view-projection transform [G]."""
    clip = view_proj_matrix @ pos
    ndc = clip[:3] / clip[3]
    return ndc[:2]

def render_slice_to_buffer(batch, target_buffer, shadow_buffer, prev_light_pass):
    """
    Draws a batch of particles to a target buffer.
    """
    for p in batch:
        screen_pos = project_to_screen(p['pos'], view_proj_matrix)
        
        if shadow_buffer and prev_light_pass:
            attenuation = shadow_buffer.sample(screen_pos)
            final_color = p['color'] * attenuation
        else:
            final_color = p['color']
            
        alpha = p['alpha']
        final_color = final_color * alpha
        
        # 'Under' blend: Out = Src + Dest * (1 - SrcAlpha)
        target_buffer.blend(screen_pos, final_color, alpha)
```

### Parameters
*   **numSlices:** Number of slices dividing the volume. Typical value: 32 to 128 [S2, unofficial].
*   **light_buffer resolution:** Resolution of the 2D shadow texture. Typical value: 256x256 or less [S2, unofficial].
*   **batchSize:** Derived from `numParticles / numSlices` [S2, unofficial].

### Edge Cases and Pitfalls
*   **Camera-Light Orientation:** When the camera points towards the light ($\mathbf{v} \cdot \mathbf{l} < 0$), the view vector must be negated to ensure correct front-to-back rendering from the light's perspective [S2, unofficial].
*   **Blending Operator:** Front-to-back rendering requires the "under" blending operator (`one_minus_dest_alpha`, `one`) instead of the standard "over" operator. This necessitates a destination alpha channel in the render target [S2, unofficial].
*   **Solid Object Intersection:** If solid objects intersect the particle volume, the light buffer will not contain correct shadowing for those objects because it only stores light reaching the final slice [S2, unofficial].
*   **Opacity Culling Artifacts:** Culling particles based on accumulated opacity at the center of the point sprite can cause popping artifacts because it ignores visible edges of the sprite [S2, unofficial].
*   **Culling Limitations:** Opacity culling cannot be used when rendering back-to-front (when the camera points towards the light) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by sorting particles ($O(N \log N)$) and rendering $N_{slices}$ passes. Each pass renders $N_{particles}/N_{slices}$ particles. Total draw calls: $N_{slices}$ [S2, unofficial].
*   **Memory Complexity:** Requires storage for particle data, sort keys, an eye buffer, and a single 2D light/shadow buffer [S1][S2, unofficial].
*   **Performance:** Fill-rate bound due to extensive frame buffer blending [S2, unofficial]. On a GeForce 9800 GTX+, rendering 65,000 particles with 128 slices, a 1280x1024 window (with 2x down-sampling), and a 256x256 light buffer achieves >100 FPS [S2, unofficial].
*   **Strengths:** Requires only a single 2D shadow buffer, avoiding the storage requirements of 3D textures or octrees [S2, unofficial]. Efficient for particle systems [S2, unofficial].
*   **Limitations:** Approximate shadowing; quality depends on the number of slices. Fill-rate intensive. Cannot handle complex occlusion from solid objects intersecting the volume correctly without extensions [S2, unofficial].

## Background
The method was presented by Joe Kniss in *GPU Gems 1* [S2, unofficial] and further detailed in NVIDIA's technical documentation on volumetric particle shadows [S2, unofficial]. It is based on the half-angle slicing scheme presented by Kniss et al. [KPH* 03] [S1]. The technique is widely used for rendering atmospheric phenomena and particle-based smoke/clouds where real-time performance is critical [S1][S2, unofficial].

## Sources
[S1] Efficient Rendering of Atmospheric Phenomena, https://doi.org/10.2312/egwr/egsr04/375-386, fetched 2026-10-26
[S2] PDF Volumetric Particle Shadows - Nvidia, https://developer.download.nvidia.com/assets/cuda/files/smokeParticles.pdf, fetched 2026-10-30 (unofficial)
