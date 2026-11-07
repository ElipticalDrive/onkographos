# Hardware Shading via Pixel Texture

Also called: Quantized Normal Shading, Volume Ray Tracing via 2D Texture Packing

## Overview
Hardware Shading via Pixel Texture is a technique that integrates surface shading into texture-memory based volume rendering by encoding quantized surface normals into a 2D pixel texture [S2, S3]. It solves the problem of missing or low-quality shading in standard hardware-accelerated volume rendering, which often lacks the geometric information required for realistic lighting [S2]. Unlike standard 3D texture mapping which relies on hardware interpolation of scalar values, this method uses nearest-neighbor interpolation on a 2D texture to avoid the redundancy and truncation errors associated with interpolating normals in hardware [S3]. It sets itself apart from siblings like [3D Pixel Texture Classification](6055-3d-pixel-texture-classification.md) by specifically targeting the shading component via a separate pixel texture lookup rather than integrating shading into the primary classification volume [S2, S3].

## How it works
The method extends the standard 3D texture mapping pipeline [Parent] by introducing a shading volume and a 2D pixel texture lookup stage. The core innovation is the representation of surface normals in a format compatible with 2D texture hardware, allowing lighting calculations to be performed during the compositing pass.

### Inputs and Outputs
*   **Inputs:**
    *   Volumetric scalar field $V(\mathbf{p})$ stored in 3D texture memory [Parent].
    *   Quantized surface normals $\mathbf{n}$ for the volume, encoded into a 2D pixel texture [S2, S3].
    *   Camera parameters and view direction [Parent].
    *   Light position and material properties (diffuse/specular coefficients) [S3].
*   **Outputs:**
    *   Final composited image with shading applied [S2, S3].

### Data Structures
1.  **3D Volume Texture:** Stores the scalar data $V(\mathbf{p})$ and potentially transfer function parameters (e.g., $V, G, H$) [S3].
2.  **2D Pixel Texture (Shading Texture):** Stores the quantized normals.
    *   **Layout:** 2D array.
    *   **Channels:** Red ($R$) and Green ($G$) channels.
    *   **Encoding:** Each normal $\mathbf{n} = (n_x, n_y, n_z)$ is quantized to 16 bits. The first 8 bits are stored in $R$, and the second 8 bits are stored in $G$ [S2, S3].
    *   **Interpolation Mode:** Nearest-neighbor. Trilinear interpolation is explicitly avoided to prevent redundancy and truncation of the quantized normal values [S2, S3].

### Numbered Steps
1.  **Normal Quantization and Encoding:**
    *   Compute surface normals $\mathbf{n}$ for the volume data.
    *   Quantize $\mathbf{n}$ using a 16-bit scheme [S2, S3].
    *   Encode the quantized normal into the 2D pixel texture:
        $$ R = \text{first 8 bits of } \mathbf{n} $$
        $$ G = \text{second 8 bits of } \mathbf{n} $$
        $$ B = 0, A = 0 \text{ (unused)} $$
    *   Generate this shading pixel texture on a per-view basis in software [S2, S3].

2.  **Ray Generation and Marching:**
    *   Generate rays per-pixel using the bounding box geometry or camera matrices [Parent].
    *   Perform ray marching through the 3D volume texture.
    *   Sample scalar values $V(\mathbf{p})$ using hardware trilinear interpolation [Parent].
    *   Apply transfer function to determine opacity $\alpha_i$ and color $C_i$ for each sample [Parent].

3.  **Shading Lookup:**
    *   During the ray marching loop, at each sample point $\mathbf{p}$, determine the corresponding location in the 2D shading texture.
    *   Perform a nearest-neighbor lookup in the 2D pixel texture to retrieve the quantized normal $\mathbf{n}_{quant}$ [S2, S3].
    *   Decode $\mathbf{n}_{quant}$ back to a normal vector $\mathbf{n}$ [S3].

4.  **Lighting Calculation:**
    *   Compute the lighting contribution $L(\mathbf{n}, \mathbf{l}, \mathbf{v})$ using the retrieved normal $\mathbf{n}$, light direction $\mathbf{l}$, and view direction $\mathbf{v}$ [S3].
    *   The lighting model is typically a standard Phong or Blinn-Phong model, though the specific formula is not detailed in the sources [S3].

5.  **Compositing:**
    *   Combine the shaded color with the unshaded color or apply shading directly to the color $C_i$ before compositing.
    *   Perform front-to-back compositing using the over operator [Parent]:
        $$ C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i $$
        $$ \alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i $$
    *   If using the SGI Octane2 implementation, render the slice to a scratch buffer, modulate it with the shading volume, copy it to the pixel texture using `glCopyPixels()`, and then blend it into the frame buffer [S3].

6.  **Early Ray Termination (ERT):**
    *   Terminate the ray if the accumulated opacity $\alpha'$ exceeds a threshold (e.g., 0.95) or if the ray exits the volume [Parent, S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Hardware Shading via Pixel Texture
# Assumes standard fragment shader environment

# Inputs:
# - volume_tex: 3D texture containing scalar data V(p)
# - shading_tex: 2D texture containing quantized normals (R, G channels)
# - light_dir: Light direction vector
# - view_dir: View direction vector
# - material_diffuse, material_specular: Material properties

# Output:
# - final_color: RGB color

def fragment_shader():
    # 1. Ray Generation (from parent method)
    ray_origin, ray_direction = generate_ray()
    
    # 2. Initialize compositing variables
    accumulated_color = vec3(0.0)
    accumulated_alpha = 0.0
    
    # 3. Ray Marching Loop
    t = ray_origin.t_min
    while t < ray_origin.t_max and accumulated_alpha < 0.95:
        # Sample volume
        p = ray_origin + t * ray_direction
        scalar_value = texture(volume_tex, p).r
        
        # Classification (from parent method)
        alpha_sample, color_sample = classify(scalar_value)
        
        # Shading Lookup
        # Map 3D position p to 2D texture coordinates for shading
        # Note: The exact mapping from 3D volume to 2D shading texture 
        # is implementation-specific (e.g., projecting to a slice or using 
        # a pre-computed 2D representation of normals).
        # Based on S3, the shading texture is generated per-view.
        shading_coords = map_to_shading_texture(p)
        
        # Nearest-neighbor lookup for quantized normal
        quantized_normal_rgb = texture(shading_tex, shading_coords, filter=NEAREST)
        
        # Decode normal from 16-bit quantization
        # R = first 8 bits, G = second 8 bits
        normal = decode_normal(quantized_normal_rgb.r, quantized_normal_rgb.g)
        
        # Lighting Calculation
        # Standard lighting model (e.g., Phong)
        diffuse = max(dot(normal, light_dir), 0.0) * material_diffuse
        specular = pow(max(dot(reflect(-light_dir, normal), view_dir), 0.0), 32.0) * material_specular
        shading_factor = diffuse + specular + ambient
        
        # Apply shading to color sample
        shaded_color = color_sample * shading_factor
        
        # Compositing
        accumulated_color = accumulated_color + (1.0 - accumulated_alpha) * alpha_sample * shaded_color
        accumulated_alpha = accumulated_alpha + (1.0 - accumulated_alpha) * alpha_sample
        
        t += step_size
        
    return accumulated_color
```

### Parameters
*   **Quantization Scheme:** 16-bit encoding for normals [S3].
*   **Interpolation:** Nearest-neighbor for shading texture [S2, S3].
*   **Threshold for ERT:** Typically 0.95 or 1.0 [Parent].

### Edge Cases and Pitfalls
*   **Hardware Flexibility:** Some hardware implementations (e.g., SGI Octane2) do not support rendering quantized normals as points with hardware lighting and copying to a pixel texture directly during rasterization. A scratch buffer and `glCopyPixels()` are required [S3].
*   **Truncation and Redundancy:** Using hardware interpolation (trilinear) on quantized normals leads to redundancy and truncation errors. Nearest-neighbor interpolation is mandatory [S2, S3].
*   **Per-View Generation:** The shading pixel texture must be regenerated on a per-view basis if the view direction changes, as the normals are encoded relative to the view [S2, S3]. This adds a software overhead, though described as minimal [S2].

## Subtopics

### Quantized Normal Shading via Pixel Texture
A specific implementation of Hardware Shading via Pixel Texture that encodes surface normals into a 2D pixel texture to avoid hardware interpolation redundancy and truncation [S3]. It uses a 16-bit quantization scheme, encoding the first 8 bits of the normal into the Red channel and the second 8 bits into the Green channel of a 2D pixel texture [S3]. The shading texture is generated on a per-view basis in software [S3]. On hardware like the SGI Octane2, which does not support per-fragment pixel texture operations during rasterization, the method requires rendering the slice to a scratch buffer, using `glCopyPixels()` to pixel texture it, and then blending it into the frame buffer [S3]. A pitfall is that some hardware is not flexible enough to support rendering quantized normals as points with hardware lighting and copying to a pixel texture [S3].

### Line Bundles
A technique that relies on graphics hardware to create texture by rendering and compositing many semitransparent line segments to produce an anisotropic volume rendering result [S4, unofficial]. It is used for flow visualization, allowing the viewing of velocity magnitude while clearly evidencing directionality, which standard volume rendering may lack [S4, unofficial]. For each data point, pre-computed sets of rendered line segments are used in a splatting analogy [S4, unofficial]. The colors of these line segments are slightly perturbed to provide a colored pattern [S4, unofficial]. The line bundles are composited together in back-to-front order [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** The shading lookup adds a constant-time texture lookup per sample. The generation of the shading pixel texture is done in software on a per-view basis, which is described as having minimal performance cost [S2, S3].
*   **Memory Complexity:** Requires additional memory for the 2D shading pixel texture [S3].
*   **Measured Performance:** The method increases rendering quality with a negligible increase in rendering cost [S2].
*   **Strengths:** Adds realistic shading to volume rendering without significant performance penalty; avoids interpolation artifacts in normals [S2, S3].
*   **Limitations:** Requires per-view regeneration of the shading texture; hardware-dependent implementation details (e.g., need for scratch buffers on SGI Octane2) [S3].
*   **When to Use:** When shading is critical for visualizing the shape of structures in the volume and hardware interpolation of normals is insufficient [S2].
*   **When Not to Use:** When the hardware does not support the required pixel texture operations or when the view changes frequently, requiring frequent regeneration of the shading texture [S3].

## Background
The method was developed to address the lack of shading in previous texture-memory based volume rendering work [S2]. It was implemented on the SGI Octane 2 with V series graphics cards and the nVidia GeForce3 series graphics adaptor [S3]. The approach leverages the hardware's ability to handle 2D pixel textures, even if not directly on a per-fragment basis during rasterization on all platforms [S3]. The Line Bundles variant was developed by Crawfis, Shen, and Max for flow visualization [S4, unofficial].

## Sources
[S1] Gradient‐based Interpolation and Sampling for Real‐time Rendering of Inhomogeneous, Single‐scattering Media, https://doi.org/10.1111/j.1467-8659.2008.01343.x, fetched 2026-10-27
[S2] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://publications.sci.utah.edu/publications/jmk01/simian-paper.pdf, fetched 2026-10-27
[S3] PDF Interactive Volume Rendering Using Multi-Dimensional Transfer Functions ..., https://collections.lib.utah.edu/dl_files/d4/e4/d4e47bfc09e54eee9f532e4f22b17e9214f4c755.pdf, fetched 2026-10-27
[S4] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
