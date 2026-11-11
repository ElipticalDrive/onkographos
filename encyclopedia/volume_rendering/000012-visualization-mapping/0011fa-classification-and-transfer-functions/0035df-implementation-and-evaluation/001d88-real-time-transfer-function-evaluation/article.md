# Real-time transfer function evaluation

Also called: Engel, Kraus & Ertl Technique, Texture-based volume rendering with GPU illumination

## Overview
This method enables real-time modification of volume classification by storing pre-evaluated transfer function results in 2D textures, allowing the GPU to classify voxels and compute primary color and opacity instantly [S1, unofficial]. It distinguishes itself from CPU-based illumination approaches (e.g., Westermann & Sevenich) by performing dynamic lighting calculations on the GPU using fragment programs, thereby supporting interactive lighting changes [S1, unofficial].

## How it works
**Inputs:**
*   Volume dataset (scalar values).
*   Transfer function definition (mapping scalar values to color/opacity).
*   Lighting parameters (view vector, light positions, material properties).

**Outputs:**
*   Rendered image with dynamic illumination.

**Data Structures:**
1.  **2D Texture (Transfer Function Lookup):** Stores the result of the transfer function evaluated for a range of input scalar values. The texture dimensions correspond to the input value range (e.g., 256x1 for 1D TF, or higher dimensions if multi-dimensional). Each texel contains the pre-computed primary color (RGB) and opacity (A) [S1, unofficial].
2.  **Compressed RGB Texture (Gradients):** Stores the pre-calculated gradient vectors for each volume element. Gradients are compressed into an RGB texture format, likely mapping the 3D gradient vector $(g_x, g_y, g_z)$ to RGB channels, possibly normalized or quantized [S1, unofficial].
3.  **3D Texture (Classification - Variant Specific):** In the PCbTF variant, the classification of each voxel is stored as a 3D texture in the frame buffer [S2].
4.  **3D RGB Texture (User Picked Colors - Variant Specific):** Stores colors and opacities picked by the user in the PCbTF variant [S2].

**Steps:**
1.  **Pre-calculate Gradients:** Compute the gradient vector for every voxel in the volume dataset. Compress these gradients into a compressed RGB texture format [S1, unofficial].
2.  **Evaluate Transfer Function:** Evaluate the transfer function for the entire range of possible input scalar values. Store these results (color and opacity) in a 2D texture [S1, unofficial].
3.  **Initialize Rendering:** Load the volume data, the 2D transfer function texture, and the compressed gradient texture into GPU memory [S1, unofficial].
4.  **Fragment Program Execution:** For each pixel (or sample) in the ray casting process:
    a.  Sample the volume data to get the scalar value $v$.
    b.  Use $v$ to index into the 2D transfer function texture to retrieve the primary color $C_{prim}$ and opacity $\alpha$ [S1, unofficial].
    c.  Sample the compressed gradient texture to retrieve the gradient vector $\mathbf{g}$ [S1, unofficial].
    d.  Calculate the final color value using a fragment program (e.g., Cg). This calculation incorporates uniform parameters modifiable interactively from the application, including diffuse and specular lighting contributions based on $\mathbf{g}$ and the view vector [S1, unofficial].
5.  **Composite:** Accumulate the final color and opacity along the ray using standard volume rendering integration [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-processing (CPU or GPU compute)
def precompute_gradients(volume_data):
    """
    Calculate gradient for each voxel and store in compressed RGB texture.
    Assumes volume_data is a 3D array of scalar values.
    """
    width, height, depth = volume_data.shape
    gradient_texture = create_texture(width, height, depth, format=RGB)
    
    for x in range(width):
        for y in range(height):
            for z in range(depth):
                # Calculate gradient using central differences
                gx = (volume_data[x+1, y, z] - volume_data[x-1, y, z]) / 2.0
                gy = (volume_data[x, y+1, z] - volume_data[x, y-1, z]) / 2.0
                gz = (volume_data[x, y, z+1] - volume_data[x, y, z-1]) / 2.0
                
                # Compress gradient into RGB format
                # Note: Exact compression scheme (e.g., normalization, quantization) 
                # is not specified in sources [S1, unofficial]. [G]
                gradient_texture.set_pixel(x, y, z, (gx, gy, gz))
    return gradient_texture

def precompute_transfer_function(tf_function, input_range):
    """
    Evaluate TF for input range and store in 2D texture.
    """
    # Assuming 1D TF for simplicity, mapped to 2D texture width
    width = input_range.max - input_range.min
    height = 1
    tf_texture = create_texture(width, height, format=RGBA)
    
    for i in range(width):
        value = input_range.min + i
        color, opacity = tf_function(value)
        tf_texture.set_pixel(i, 0, (color.r, color.g, color.b, opacity))
    return tf_texture

# Rendering (GPU Fragment Program)
def render_fragment(volume_sample, tf_texture, gradient_texture, lighting_params):
    """
    Calculate final color for a single sample.
    """
    # 1. Get primary color and opacity from TF texture
    scalar_value = volume_sample.scalar
    # Map scalar_value to texture coordinate
    tf_coord = map_scalar_to_tf_coord(scalar_value, tf_texture)
    primary_color, opacity = tf_texture.sample(tf_coord)
    
    # 2. Get gradient from gradient texture
    gradient = gradient_texture.sample(volume_sample.position)
    
    # 3. Calculate lighting
    # Lighting parameters (view vector, light positions, material) are uniforms
    final_color = calculate_lighting(
        primary_color=primary_color,
        gradient=gradient,
        view_vector=lighting_params.view_vector,
        light_positions=lighting_params.light_positions,
        material=lighting_params.material
    )
    
    return final_color, opacity

def calculate_lighting(primary_color, gradient, view_vector, light_positions, material):
    """
    Calculate diffuse and specular contributions.
    """
    # Normalize gradient to get normal vector
    normal = normalize(gradient)
    
    total_diffuse = 0.0
    total_specular = 0.0
    
    for light in light_positions:
        light_direction = normalize(light.position - camera.position) # Simplified
        # Diffuse: N . L
        diffuse_factor = max(dot(normal, light_direction), 0.0)
        total_diffuse += diffuse_factor * material.diffuse
        
        # Specular: (R . V)^shininess
        # R = 2 * (N . L) * N - L
        reflection = 2.0 * dot(normal, light_direction) * normal - light_direction
        specular_factor = pow(max(dot(reflection, view_vector), 0.0), material.shininess)
        total_specular += specular_factor * material.specular
        
    final_color = primary_color * (material.ambient + total_diffuse) + total_specular * material.specular_color
    return final_color
```

**Parameters:**
*   **Gradient Storage:** Compressed into an RGB texture format [S1, unofficial].
*   **Illumination:** Dynamic, performed on the GPU using Cg [S1, unofficial].
*   **Hardware (PCbTF Variant):** Intel Xeon CPU 3.60GHz, 3GB memory, Nvidia GeForce GTX 285 graphic card [S2].
*   **Software Libraries (PCbTF Variant):** OpenGL and FLTK for interface, Cg for rendering [S2].

**Edge Cases and Pitfalls:**
*   **Reproducibility:** The original work's results could not be reproduced because the plane reduction optimization was not implemented due to the complexity of their approximation towards two-texture interpolation [S1, unofficial].
*   **Precision:** Compressing gradients into RGB textures may introduce quantization errors, affecting lighting accuracy [G].
*   **Boundary Conditions:** Gradient calculation at volume boundaries requires careful handling (e.g., clamping, mirroring) to avoid artifacts [G].

## Subtopics

### [List-based Pre-classification](1930-list-based-pre-classification.md)
List-based pre-classification is an optimization for interactive transfer function changes in volume rendering that avoids rebuilding index buffers on the CPU by pre-sorting voxel indices into iso-value buckets, allowing fast pointer merging. Unlike naive pre-classification which requires rebuilding and reloading the index buffer on every transfer function change (causing CPU-GPU bandwidth bottlenecks), this method uses pre-sorted buffers and pointer merging for interactive rates.

### GPU-Accelerated Volume Rendering with PCbTF
A rendering implementation that accelerates volume rendering and transfer function specification using graphics hardware, treating TF design as a mapping from nD vectors to color and opacity values [S2]. It stores the classification of each voxel as a 3D texture in the frame buffer and recomputes/redraws the new texture on the screen via the GPU whenever the Transfer Function changes [S2]. It calculates a separate 3D RGB texture to store the colors and opacities picked by the user [S2].

## Operating characteristics
*   **Performance:** Achieves 4 to 6 frames per second with GPU-based illumination and shading [S1, unofficial].
*   **Complexity:** Time complexity is dominated by the ray casting and fragment program execution per frame. Pre-computation of gradients and TF evaluation is done once or when parameters change significantly.
*   **Memory:** Requires storage for the volume data, 2D TF texture, compressed gradient texture, and potentially 3D classification/color textures in variants.
*   **Strengths:** Enables dynamic lighting and real-time TF modification, which is not possible with pre-computed CPU illumination [S1, unofficial].
*   **Limitations:** Performance is limited by GPU fill rate and fragment program complexity. The original plane reduction optimization was not implemented, potentially impacting performance on large volumes [S1, unofficial].
*   **When to Use:** When interactive lighting changes and real-time TF adjustment are required.
*   **When Not to Use:** When maximum image quality is prioritized over speed, or when hardware does not support the required GPU features (e.g., Cg, 3D textures).

## Background
The technique was proposed by Engel, Kraus, and Ertl. It is discussed in the context of texture mapping-based volume rendering [S1, unofficial]. The approach leverages the computational power of modern graphics cards to perform illumination and shading on the GPU, contrasting with earlier methods that calculated illumination on the CPU [S1, unofficial]. The PCbTF variant was implemented to accelerate volume rendering and TF specification using graphics hardware [S2].

## Sources
[S1] Texture Mapping Based Volume Rendering, https://www.ciroduran.com/files/Tesis.pdf, fetched 2026-10-29 (unofficial)
[S2] Multi-dimensional Reduction and Transfer Function Design using Parallel ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC4536824/, fetched 2026-10-27
