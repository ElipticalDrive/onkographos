# Hardware Shading via Pixel Texture

Also called: Quantized Normal Shading, Volume Ray Tracing via 2D Texture Packing

## Overview
Hardware Shading via Pixel Texture is a method for adding surface shading to texture-memory based volume rendering by encoding quantized surface normals into a 2D pixel texture. It solves the problem of poor shape perception in standard volume rendering, which often lacks explicit shading cues [S2]. It sets itself apart from sibling methods like [3D Pixel Texture Classification](6055-3d-pixel-texture-classification.md) by specifically targeting the shading component using a 16-bit quantization scheme and nearest-neighbor interpolation to avoid hardware interpolation redundancy and truncation [S2, S3].

## How it works
The method builds on [Hardware-Accelerated Volume Rendering](5616-hardware-accelerated-volume-rendering.md). It extends the standard ray casting loop by integrating a shading step that utilizes a pre-computed 2D pixel texture containing quantized normals.

**Inputs and Outputs:**
*   **Input:** 3D scalar volume data $V$, camera parameters, and a shading volume containing pre-computed normals.
*   **Output:** A 2D image with shaded volume rendering.

**Data Structures:**
*   **Shading Pixel Texture:** A 2D texture where each texel stores a quantized normal vector.
    *   **Layout:** 16-bit quantization.
    *   **Channel Packing:** The first 8 bits of the quantized normal are stored in the Red channel, and the second 8 bits are stored in the Green channel [S2, S3].
    *   **Interpolation:** Nearest-neighbor interpolation is used for lookups to prevent redundancy and truncation errors associated with linear interpolation of quantized values [S2, S3].

**Steps:**
1.  **Normal Quantization:** Compute surface normals for the volume data. Encode these normals using a 16-bit quantization scheme [S2, S3].
2.  **Texture Packing:** Pack the quantized normals into a 2D pixel texture. The first 8 bits of the normal are assigned to the Red channel, and the second 8 bits to the Green channel [S2, S3].
3.  **Shading Texture Generation:** Generate the shading pixel texture on a per-view basis. This can be done in software with minimal performance cost, or in hardware by rendering quantized normals as points with hardware lighting and copying the result to the pixel texture [S2, S3].
4.  **Volume Ray Tracing:** Perform volume ray tracing for all sample points in a single call. This is achieved by packing all sample points into a 2D texture and drawing a quad of the same size to trigger the pixel shader [S1].
5.  **Shading Integration:** During the fragment shader execution, retrieve the quantized normal from the shading pixel texture using nearest-neighbor interpolation. Use this normal to compute lighting (e.g., diffuse shading) [S2, S3].
6.  **Compositing:** Apply the standard front-to-back compositing equation, modulating the color with the computed shading [S1, S2, S3].
    $$
    C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i
    $$
    $$
    \alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
    $$
7.  **Termination:** Terminate the tracing of a ray if it exits the volume or if accumulated opacity exceeds a threshold [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Fragment Shader for Hardware Shading via Pixel Texture

# Inputs
# - ray_origin, ray_direction: Camera ray parameters
# - volume_texture: 3D texture for scalar values V
# - shading_texture: 2D pixel texture for quantized normals
# - transfer_function: Function mapping V to color C and opacity alpha

# Constants
# - EXIT_THRESHOLD: Opacity threshold for early termination (e.g., 0.95)
# - STEP_SIZE: Distance between samples along the ray

# Helper: Decode quantized normal from 2D texture
def decode_normal(tex_coord):
    # Sample the 2D pixel texture using nearest-neighbor interpolation
    # Red channel contains first 8 bits, Green channel contains second 8 bits
    r = texture_nearest(shading_texture, tex_coord).r
    g = texture_nearest(shading_texture, tex_coord).g
    
    # Reconstruct 16-bit quantized normal value
    # Note: Exact reconstruction depends on the specific quantization scheme
    # Assuming a simple linear mapping for demonstration
    quantized_normal = r * 256 + g
    
    # Convert back to normalized vector component (example)
    # This step depends on the specific encoding used in [S2, S3]
    normal_component = quantized_normal / 65535.0
    
    # Return a placeholder normal vector; actual implementation requires
    # full 3D normal reconstruction which may involve multiple texture lookups
    # or a more complex encoding scheme not fully detailed in sources.
    # [G] Assuming a simplified 1D normal representation for the sake of
    # complete pseudocode, as the sources describe a 16-bit encoding
    # that effectively acts as a 1D nearest neighbor lookup.
    return vec3(normal_component, 0.0, 0.0) # Placeholder

# Main Ray Marching Loop
color = vec3(0.0)
alpha = 0.0

# Determine ray entry and exit points (from parent method)
t_min, t_max = get_ray_volume_intersection(ray_origin, ray_direction)

t = t_min
while t < t_max and alpha < EXIT_THRESHOLD:
    # Sample point in volume
    sample_point = ray_origin + t * ray_direction
    
    # Sample scalar value
    v = texture(volume_texture, sample_point).r
    
    # Apply transfer function
    c, a = transfer_function(v)
    
    # Compute shading
    # [G] The exact lighting model is not specified, but standard diffuse
    # lighting is implied by the use of normals.
    normal = decode_normal(sample_point.xy) # Simplified UV mapping
    light_dir = normalize(vec3(0.0, 0.0, 1.0)) # Placeholder light direction
    diffuse = max(dot(normal, light_dir), 0.0)
    shaded_color = c * diffuse
    
    # Front-to-back compositing
    # C' = C' + (1 - alpha') * alpha * C
    # alpha' = alpha' + (1 - alpha') * alpha
    color = color + (1.0 - alpha) * a * shaded_color
    alpha = alpha + (1.0 - alpha) * a
    
    t += STEP_SIZE

# Output final color
frag_color = vec4(color, alpha)
```

**Parameters:**
*   **Quantization Scheme:** 16-bit encoding for normals [S3].
*   **Interpolation:** Nearest-neighbor for shading texture lookups [S2, S3].
*   **Exit Threshold:** Typically 0.95 or 1.0 for early ray termination [S1].

**Edge Cases and Pitfalls:**
*   **Hardware Flexibility:** Some hardware implementations (e.g., SGI Octane2) are not flexible enough to support rendering quantized normals as points with hardware lighting and copying to a pixel texture directly during rasterization [S3].
*   **Interpolation Redundancy:** Using linear interpolation for quantized normals can lead to redundancy and truncation errors; nearest-neighbor is required [S2, S3].
*   **View Dependency:** The shading pixel texture must be regenerated on a per-view basis if the view point changes, as the normals are view-dependent [S2, S3].

## Subtopics

### Quantized Normal Shading via Pixel Texture
A shading method that encodes surface normals into a 2D pixel texture to avoid hardware interpolation redundancy and truncation, using nearest-neighbor interpolation. It is chosen when high-quality shading is needed with minimal performance cost, and the hardware supports 2D pixel textures. It differs from standard shading by explicitly handling the quantization and packing of normals into a 2D texture format. Link: [Quantized Normal Shading via Pixel Texture](6055-3d-pixel-texture-classification.md) (Note: No separate article exists, so this is a subtopic of the current method).

### Line Bundles
A technique relying on graphics hardware to create texture by rendering and compositing many semitransparent line segments to produce an anisotropic volume rendering result. It is chosen for visualizing flow fields where directionality is important. It differs from standard volume rendering by explicitly rendering line segments to convey direction. Link: [Line Bundles](6503-gpu-ray-casting-for-light-field-display.md) (Note: No separate article exists, so this is a subtopic of the current method).

## Operating characteristics
*   **Time Complexity:** The shading texture generation is performed on a per-view basis, but the cost is minimal [S2, S3]. The ray marching loop complexity is linear with respect to the number of samples along the ray, similar to standard volume rendering [S1].
*   **Memory Complexity:** Requires additional memory for the 2D shading pixel texture [S2, S3].
*   **Measured Performance:** The performance cost of generating the shading pixel texture is minimal [S2, S3]. The method increases rendering quality with negligible increase in rendering cost [S2].
*   **Strengths:** Adds shading to texture-memory based volume rendering, increasing quality with negligible cost [S2]. Avoids hardware interpolation redundancy and truncation [S3].
*   **Limitations:** Requires per-view regeneration of the shading texture if the view point changes [S2, S3]. Some hardware implementations may not support the required operations [S3].
*   **When to Use:** When shading is needed for volume rendering and the hardware supports 2D pixel textures with nearest-neighbor interpolation [S2, S3].
*   **When Not to Use:** When the hardware does not support the required pixel texture operations or when per-view regeneration is too costly [S3].

## Background
The method was developed as part of interactive volume rendering research using multi-dimensional transfer functions [S2, S3]. It was implemented on SGI Octane 2 with V series graphics cards and nVidia GeForce3 series graphics adaptors [S3]. The approach addresses the lack of shading in previous texture-memory based volume rendering work [S2]. The concept of packing sample points into a 2D texture for volume ray tracing was also explored in the context of gradient-based interpolation and sampling for real-time rendering of inhomogeneous, single-scattering media [S1].

## Sources
[S1] Gradient‐based Interpolation and Sampling for Real‐time Rendering of Inhomogeneous, Single‐scattering Media, https://doi.org/10.1111/j.1467-8659.2008.01343.x, fetched 2026-10-27
[S2] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
[S3] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
[S4] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
