# Hardware Shading via Pixel Texture

Also called: Quantized Normal Shading, Volume Ray Tracing via 2D Texture Packing

## Overview
Hardware Shading via Pixel Texture is a technique that integrates surface shading into hardware-accelerated volume rendering by encoding quantized surface normals into a 2D pixel texture. It solves the problem of adding geometric shading to texture-memory based volume rendering, which previously lacked efficient shading mechanisms [S2]. Unlike standard 3D texture sampling which suffers from redundancy and truncation errors when interpolating normals, this method uses nearest-neighbor interpolation on a dedicated 2D texture to retrieve lighting information without artifacts [S1, S2].

## How it works
The method extends the standard ray casting pipeline by introducing a secondary shading pass or integrated shading logic that relies on pre-computed or dynamically generated normal data stored in 2D texture memory.

**Inputs and Outputs:**
*   **Input:** 3D volume data (density/radiance), camera parameters, and a 2D pixel texture containing quantized surface normals.
*   **Output:** Final composited image with shaded appearance.

**Data Structures:**
*   **Shading Texture:** A 2D texture where each texel stores a quantized surface normal.
    *   **Layout:** The normal vector $\mathbf{n} = (n_x, n_y, n_z)$ is quantized to 16 bits total.
    *   **Encoding:** The first 8 bits of the quantized normal are stored in the Red channel ($R$), and the second 8 bits are stored in the Green channel ($G$) [S2]. The Blue and Alpha channels are unused for normal data.

**Algorithm Steps:**

1.  **Normal Quantization:**
    Surface normals are computed for the volume data. Each component of the normal vector is quantized into a 16-bit integer representation. The specific bit-packing scheme assigns the first 8 bits to the Red channel and the next 8 bits to the Green channel of the 2D shading texture [S2].

2.  **Shading Texture Generation:**
    The 2D pixel texture containing the quantized normals is generated. This is typically done on a per-view basis in software to account for view-dependent normal orientations, although hardware generation is noted as feasible [S2]. The performance cost of this software generation is minimal [S2].

3.  **Ray Marching and Sampling:**
    Volume ray tracing is performed for all sample points in a single call. All sample points are packed into a 2D texture structure [S1]. A quad of the same size as the packed texture is drawn to trigger the pixel shader [S1].
    *   Inside the pixel shader, the ray marching loop executes.
    *   Density and radiance values are retrieved from 3D textures.
    *   The ray is terminated if it exits the volume [S1].

4.  **Shading Lookup:**
    During the compositing or shading phase, the quantized normal is retrieved from the 2D pixel texture.
    *   **Interpolation Mode:** Nearest-neighbor interpolation is strictly used for the pixel texture lookup [S2]. This avoids the redundancy and truncation errors associated with hardware trilinear interpolation of normals [S1, S2].
    *   The lookup is equivalent to a 16-bit 1D nearest neighbor operation [S2].

5.  **Lighting Calculation:**
    The retrieved quantized normal is used to compute lighting. The method treats the 2D pixel texture lookup as equivalent to an 8-bit per-axis 2D nearest neighbor texture lookup [S2]. The lighting calculation applies standard illumination models using the decoded normal.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the shading integration within the fragment shader

def decode_normal_from_texture(tex_color):
    """
    Decodes the 16-bit quantized normal from the 2D pixel texture.
    tex_color: vec4 from texture lookup (R, G, B, A)
    Returns: vec3 normal (normalized)
    """
    # Extract 8-bit integers from Red and Green channels
    # Note: Exact bit-shifting depends on the specific quantization scheme
    # described in [S2], but logically it splits the 16-bit value.
    bits_rg = tex_color.r * 255.0  # Scale to 0-255
    bits_g  = tex_color.g * 255.0  # Scale to 0-255
    
    # Reconstruct the 16-bit integer value
    # Assuming first 8 bits are R, next 8 are G
    quantized_value = (bits_rg << 8) | bits_g
    
    # Convert back to normalized vector components
    # This step assumes a specific mapping from 16-bit int to 3D vector
    # which is not fully detailed in sources, but implies a standard
    # quantization to [-1, 1] or [0, 1] range.
    # [G] Standard quantization: val / 32767.0 - 1.0 or similar.
    # Since the source says "first 8 bits... second 8 bits", it implies
    # a specific encoding. We assume a standard reconstruction.
    
    # Placeholder for exact bit-to-vector conversion logic not fully specified
    # in the provided text, but required for implementation.
    normal = reconstruct_vector_from_16bit(quantized_value)
    return normalize(normal)

def fragment_shader():
    """
    Main fragment shader logic for Hardware Shading via Pixel Texture.
    """
    # 1. Perform standard volume ray marching to get density/color
    #    (See [S1] for ray marching details: intersect RBFs, alpha blend)
    density, radiance = perform_ray_marching()
    
    # 2. Terminate if ray exits volume (handled in marching loop)
    if ray_exited_volume:
        discard()

    # 3. Lookup quantized normal from 2D shading texture
    #    Must use NEAREST_NEIGHBOR filtering
    shading_texel = texture(shading_texture, uv, LOD=0, compare_func=NONE)
    
    # 4. Decode normal
    normal = decode_normal_from_texture(shading_texel)
    
    # 5. Compute lighting
    #    [G] Standard Phong or Blinn-Phong model using the decoded normal
    light_color = compute_lighting(normal, view_dir, light_dir)
    
    # 6. Composite shaded color
    final_color = radiance * light_color
    
    return final_color
```

**Parameters:**
*   **Interpolation Mode:** Nearest-neighbor (critical for shading texture) [S2].
*   **Quantization:** 16-bit total (8 bits per channel for R/G) [S2].
*   **Texture Generation:** Per-view basis in software [S2].

**Edge Cases and Pitfalls:**
*   **Truncation Errors:** Using standard hardware interpolation (linear/trilinear) on the normal texture causes redundancy and truncation errors. Nearest-neighbor is mandatory [S1, S2].
*   **View Dependency:** The shading texture must be regenerated per-view if normals are view-dependent, incurring a minimal software cost [S2].

## Subtopics

### [Quantized Normal Shading via Pixel Texture](6549-quantized-normal-shading-via-pixel-texture.md)
A variant that explicitly focuses on the quantization scheme and the avoidance of interpolation artifacts by using a dedicated 2D texture lookup for lighting calculations. It distinguishes itself by ensuring quantized normal data is retrieved without interpolation artifacts.

### [Line Bundles](7701-line-bundles.md)
An extension that applies the splatting concept to composite collections of antialiased lines in a back-to-front manner, visualizing 3D vector fields. It builds on this method by extending the concept of splats to convey directionality.

## Operating characteristics
*   **Performance Cost:** The addition of shading increases rendering quality with negligible cost [S2]. The software generation of the shading texture has minimal performance impact [S2].
*   **Strengths:** Adds geometric shading to texture-memory based volume rendering, which previously lacked this feature [S2]. Avoids interpolation artifacts associated with hardware normal interpolation [S1, S2].
*   **Limitations:** Requires a separate 2D texture for normals. The shading texture generation is currently done in software per-view, though hardware generation is noted as possible [S2].
*   **When to Use:** When geometric shape information is critical for interpreting volume structures and hardware-accelerated 3D texturing is already in use [S2].

## Background
The method was introduced in the context of improving texture-memory based volume rendering. Previous work in this area often lacked shading, which is a fundamental component for expressing the shape of structures in the volume [S2]. The technique was developed to dramatically increase rendering quality with negligible cost [S2]. It is associated with implementations on hardware such as the Octane2 and GeForce3, utilizing multi-texturing pipelines [S2].

## Sources
[S1] Gradient‐based Interpolation and Sampling for Real‐time Rendering of Inhomogeneous, Single‐scattering Media, https://doi.org/10.1111/j.1467-8659.2008.01343.x, fetched 2026-10-27
[S2] PDF Interactive Volume Rendering Using Multi‐Dimensional Transfer Functions ..., https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
