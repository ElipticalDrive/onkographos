# Half-Angle Slice Rendering

Also called: Half-Angle Shadowing

## Overview
Half-Angle Slice Rendering is a technique for computing volumetric shadows in particle-based media by rendering the volume as a series of slices perpendicular to a "half-angle" vector between the view and light directions [S1, unofficial]. It solves the problem of efficiently accumulating shadowing information while blending the volume, allowing both operations to occur simultaneously using a single 2D shadow buffer [S1, unofficial]. Unlike voxel-based volume rendering methods that require 3D textures and complex ray marching, this method relies on sorting particles and 2D blending, making it suitable for real-time particle systems [S1, unofficial].

## How it works
The method approximates the volume as a collection of point sprites (particles). The core innovation is aligning the slicing direction with the half-angle vector to ensure that the front-to-back order from the camera's perspective is roughly consistent with the front-to-back order from the light's perspective. This allows the accumulation of light transmittance (shadowing) to happen in the same passes as the color blending [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Particle system: Array of particles, each with position $\mathbf{p}_i$, color $\mathbf{c}_i$, and opacity $\alpha_i$.
    *   View direction $\mathbf{v}$ (normalized vector from camera to scene).
    *   Light direction $\mathbf{l}$ (normalized vector from scene to light source).
    *   Rendering parameters: `numSlices` (integer), `lightBufferResolution` (e.g., $256 \times 256$).
*   **Outputs:**
    *   Final image buffer with volumetrically shaded particles.
    *   Light buffer (2D texture) storing accumulated transmittance/shadowing for the next slice.

### Data Structures
1.  **Particle Array:** Structured as an array of objects containing `pos` (float3), `color` (float3), and `alpha` (float).
2.  **Sort Keys:** Array of floats, `sort_keys`, where `sort_keys[i] = dot(halfVector, particle[i].pos)`.
3.  **Render Targets:**
    *   `image_buffer`: Standard color buffer with an alpha channel.
    *   `light_buffer`: 2D texture (typically low resolution, e.g., $256 \times 256$) with an alpha channel to store accumulated opacity. The alpha channel is critical for the "under" blending operator [S1, unofficial].

### Algorithm Steps

1.  **Calculate Half-Angle Vector:**
    Determine the direction perpendicular to the slices. The direction depends on the relative orientation of the view and light vectors to ensure correct front-to-back rendering from the light's perspective [S1, unofficial].
    $$
    \text{if } \mathbf{v} \cdot \mathbf{l} > 0: \quad \mathbf{h} = \text{normalize}(\mathbf{v} + \mathbf{l}), \quad \text{flipped} = \text{false}
    $$
    $$
    \text{else}: \quad \mathbf{h} = \text{normalize}(-\mathbf{v} + \mathbf{l}), \quad \text{flipped} = \text{true}
    $$
    Here, $\mathbf{v}$ is the view vector, $\mathbf{l}$ is the light vector, and $\mathbf{h}$ is the half-angle vector [S1, unofficial].

2.  **Sort Particles:**
    Project each particle's position onto $\mathbf{h}$ to generate sorting keys. Sort the particles based on these keys. This orders particles from front-to-back (or back-to-front) relative to the half-angle direction [S1, unofficial].
    $$
    \text{sort\_keys}[i] = \mathbf{h} \cdot \mathbf{p}_i
    $$
    Sort the particle array such that particles with smaller `sort_keys` are rendered first if `flipped` is false, or last if `flipped` is true (depending on implementation details of the "under" operator requirement) [S1, unofficial]. Note: The source code implies sorting by key and then iterating through slices [S1, unofficial].

3.  **Initialize Buffers:**
    Clear `light_buffer` and `image_buffer` [S1, unofficial].

4.  **Iterate Slices:**
    Divide the sorted particle list into `numSlices` batches. For each slice $i$ from $0$ to `numSlices` - 1:
    a.  **Render to Image Buffer:** Draw the current batch of particles to the `image_buffer`. Use the "under" blending operator. This step composites the current slice's color onto the existing accumulated color, using the `light_buffer` from the *previous* slice to modulate the lighting (shadowing) [S1, unofficial].
    b.  **Render to Light Buffer:** Draw the *same* batch of particles to the `light_buffer`. This updates the accumulated transmittance for the next slice. The blending function here accumulates opacity [S1, unofficial].

5.  **Particle Shading:**
    For each particle, render as a point sprite. The pixel shader should:
    a.  Calculate radial falloff for the sprite shape.
    b.  Pre-multiply color by alpha: $\mathbf{c}_{out} = \mathbf{c}_{in} \cdot \alpha_{in}$.
    c.  Return `float4(c_out, alpha_in)` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def half_angle_slice_rendering(particles, view_vector, light_vector, num_slices, light_buffer_res):
    # 1. Calculate half-angle vector
    dot_product = dot(normalize(view_vector), normalize(light_vector))
    if dot_product > 0.0:
        half_vector = normalize(view_vector + light_vector)
        flipped = False
    else:
        half_vector = normalize(-view_vector + light_vector)
        flipped = True

    # 2. Calculate sorting keys
    sort_keys = []
    for p in particles:
        key = dot(half_vector, p.pos)
        sort_keys.append(key)

    # 3. Sort particles
    # Create indices and sort based on keys
    indices = range(len(particles))
    sorted_indices = argsort(sort_keys)
    sorted_particles = [particles[i] for i in sorted_indices]

    # 4. Initialize buffers
    light_buffer = create_texture(light_buffer_res, format='RGBA')
    image_buffer = create_render_target(format='RGBA')
    clear(light_buffer)
    clear(image_buffer)

    # 5. Iterate through slices
    batch_size = len(sorted_particles) // num_slices
    for i in range(num_slices):
        start_idx = i * batch_size
        end_idx = start_idx + batch_size
        current_batch = sorted_particles[start_idx:end_idx]

        # Draw slice to image buffer (using previous light buffer state for shadowing)
        set_blend_mode('UNDER') # one_minus_dest_alpha, one
        render_to(image_buffer, current_batch, light_buffer)

        # Draw slice to light buffer (accumulate opacity for next slice)
        set_blend_mode('ACCUMULATE') # Specific blend for light buffer accumulation
        render_to(light_buffer, current_batch, None)

    return image_buffer

def render_to(target, batch, light_tex):
    for p in batch:
        # Vertex shader: project position, check opacity culling if applicable
        # Pixel shader:
        #   alpha = p.alpha * radial_falloff()
        #   color = p.color * alpha
        #   if light_tex is not None:
        #       shadow = texture_lookup(light_tex, p.proj_pos)
        #       color = color * shadow
        #   return float4(color, alpha)
        pass
```

### Parameters
*   `numSlices`: Number of slices to divide the volume into. Typical value: 32 to 128 [S1, unofficial].
*   `lightBufferResolution`: Resolution of the 2D shadow texture. Typical value: $256 \times 256$ or less [S1, unofficial].

### Edge Cases and Pitfalls
*   **View/Light Orientation:** If the camera points towards the light (dot product negative), the view vector must be negated before calculating the half-angle vector to ensure correct front-to-back rendering from the light's perspective [S1, unofficial].
*   **Blending Operator:** Front-to-back rendering requires the "under" blending operator (`one_minus_dest_alpha`, `one`) instead of the standard "over" operator. This requires a destination alpha channel in the render target [S1, unofficial].
*   **Solid Object Intersections:** If solid objects intersect the particle volume, the light buffer will not contain correct shadowing for those objects because it only stores light reaching the final slice of particles. Correct results require sorting solid objects along the half-angle axis as well or using linear approximations [S1, unofficial].
*   **Opacity Culling Artifacts:** Culling particles based on accumulated opacity at the center of the point sprite can cause popping artifacts because it does not account for visible edges of the particle sprite. This optimization cannot be used when rendering back-to-front (when camera points towards light) [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by particle sorting ($O(N \log N)$) and rendering $N$ particles across `numSlices` passes. Sorting can be performed on GPU (e.g., CUDA radix sort) to avoid CPU bottlenecks [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for particle storage and $O(R^2)$ for the light buffer, where $R$ is the light buffer resolution. No 3D voxel storage is required [S1, unofficial].
*   **Performance:** Fill-rate bound due to extensive frame buffer blending. In tests, 65,000 particles with CUDA sorting, 1280x1024 window (2x down-sampling), and 256x256 light buffer achieved >100 FPS on a GeForce 9800 GTX+ [S1, unofficial].
*   **Strengths:** Simple implementation, low memory footprint compared to voxel methods, real-time capable on older hardware [S1, unofficial].
*   **Limitations:** Approximate shadows (diffuse nature of volume limits high-frequency shadow detail), artifacts from opacity culling, incorrect shadowing for intersecting solid objects [S1, unofficial].
*   **When to use:** Particle-based volumes (smoke, fire, clouds) where performance is critical and high-fidelity volumetric ray marching is not feasible [S1, unofficial].
*   **When not to use:** Volumes with sharp shadow boundaries, scenes with complex solid object intersections within the volume, or when high-resolution shadow maps are required [S1, unofficial].

## Background
The technique was presented by NVIDIA as an extension of the volume shading technique by Joe Kniss in *GPU Gems 1* [S1, unofficial]. It was designed to add volumetric shadowing to particle systems efficiently. The method leverages the fact that for many participating media, shadows are diffuse, allowing for low-resolution shadow buffers and simplified blending operations [S1, unofficial].

## Sources
[S1] PDF Volumetric Particle Shadows - Nvidia, https://developer.download.nvidia.com/assets/cuda/files/smokeParticles.pdf, fetched 2026-10-30 (unofficial)
