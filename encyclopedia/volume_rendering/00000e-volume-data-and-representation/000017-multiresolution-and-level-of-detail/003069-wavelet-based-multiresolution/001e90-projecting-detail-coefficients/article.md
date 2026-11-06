# Projecting Detail Coefficients
Also called: Multi-level Detail Coefficient Projection, Wavelet MIP Projection

## Overview
Projecting Detail Coefficients is a rendering technique for multiresolution volume data that projects wavelet detail coefficients and approximations directly to screen-space quads of decreasing resolution [S1]. It solves the problem of efficiently rendering large volumetric datasets by leveraging the hierarchical structure of wavelet decompositions, where lower levels represent coarser approximations [S1]. Unlike sibling methods that may reconstruct volumes or use tree traversal for ray casting, this method renders each level of the decomposition as a separate geometric primitive, scaling the output quad size inversely with the resolution level [S1].

## How it works
**Inputs:**
- A multiresolution volume dataset decomposed into $L+1$ levels: detail levels $d_0, \dots, d_{L-1}$ and the final approximation $f_L$ [S1].
- Volume orientation (quaternion) and zoom factor (default 1 voxel per pixel) [S1].
- Intensity thresholds: a lower threshold (background) and an optional upper threshold (maximum intensity) [S1].

**Outputs:**
- A composite image formed by projecting voxels from all levels [S1].

**Data Structures:**
- The volume is stored as $L+1$ separate volumes, one for each level of the decomposition [S1].
- Each volume contains voxels corresponding to the detail coefficients ($d_i$) or the approximation ($f_L$) at that level [S1].

**Steps:**
1. **Matrix Construction:** Build a model-view matrix from the volume orientation quaternion and the zoom factor. Combine this with the projection matrix to form a 4x4 modelview-projection matrix. Pass this matrix to the vertex shader [S1].
2. **Intensity Iteration:** Iterate over the intensity values present in the volume, starting from the lowest intensity (or user-defined lower threshold) up to the highest [S1].
3. **Upper Threshold Optimization:** If an upper intensity threshold exists above which all voxels have maximum intensity, project all such voxels in a single call to `glDrawArrays` using point primitives. This step skips rounding and anti-aliasing for these high-intensity values [S1].
4. **Level Rendering:** Render each level $i$ (from $0$ to $L$) to a quad:
   - Level 0 is rendered to a quad with the same dimensions as the viewport [S1].
   - Level 1 is rendered to a quad with half the width and height of the viewport [S1].
   - Level $k$ is rendered to a quad with dimensions scaled by $1/2^k$ relative to the viewport [S1].
   - The rendering process for each level follows the same projection logic as described in steps 2-3, applied to the specific detail coefficients or approximation of that level [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def project_detail_coefficients(volume_levels, orientation_quat, zoom_factor, 
                                lower_threshold, upper_threshold, viewport_size):
    """
    volume_levels: List of L+1 volumes [d0, d1, ..., dL-1, fL]
    orientation_quat: Quaternion for volume orientation
    zoom_factor: Float, default 1.0 (voxels per pixel)
    lower_threshold: Intensity value below which is background
    upper_threshold: Optional intensity value above which is max intensity
    viewport_size: Tuple (width, height)
    """
    # Step 1: Construct Model-View-Projection Matrix
    model_view_matrix = build_model_view_matrix(orientation_quat, zoom_factor)
    projection_matrix = build_projection_matrix() # Assuming standard perspective/orthographic
    mvp_matrix = multiply_matrices(projection_matrix, model_view_matrix)
    
    # Pass MVP to vertex shader
    set_shader_uniform("mvp_matrix", mvp_matrix)
    
    # Step 2 & 3: Handle Upper Threshold Optimization
    if upper_threshold is not None:
        max_intensity_voxels = get_voxels_above_threshold(volume_levels, upper_threshold)
        if max_intensity_voxels:
            draw_point_primitives(max_intensity_voxels)
            # Remove these voxels from further processing or mark as processed
            # Note: The source implies a single call for all max intensity voxels
            # across the volume, likely requiring a pre-filtered list or shader logic.
            # For clarity, we assume a helper function retrieves these.
    
    # Step 4: Render each level
    for level in range(len(volume_levels)):
        current_volume = volume_levels[level]
        
        # Determine quad size for this level
        # Level 0: viewport size
        # Level 1: half size
        # Level k: 1/2^k size
        scale_factor = 1.0 / (2 ** level)
        quad_width = int(viewport_size[0] * scale_factor)
        quad_height = int(viewport_size[1] * scale_factor)
        
        # Render the current level's volume to the quad
        # The rendering iterates over intensity values from lower_threshold upwards
        render_level_to_quad(current_volume, lower_threshold, quad_width, quad_height)

def render_level_to_quad(volume, lower_threshold, quad_width, quad_height):
    """
    Renders a single level's volume data to a quad of specified size.
    Iterates over intensity values.
    """
    # Get voxels within the volume that are above the lower threshold
    # and below the upper threshold (if applicable and not already drawn)
    voxels_to_render = get_voxels_in_range(volume, lower_threshold)
    
    # Project these voxels to the quad
    # The vertex shader handles the projection using the MVP matrix
    draw_voxels_as_points(voxels_to_render, quad_width, quad_height)
```

**Parameters:**
- Zoom factor: Default 1 voxel per pixel [S1].
- Lower threshold: User-defined or minimum intensity in volume [S1].
- Upper threshold: Optional, for maximum intensity optimization [S1].

**Edge Cases:**
- If no upper threshold is defined, all voxels are processed in the iteration loop [S1].
- Precision issues may arise if the zoom factor does not align with pixel boundaries, though the method uses point primitives without anti-aliasing [S1].

## Subtopics

### [Coefficient Quantization Error Estimate](3334-coefficient-quantization-error-estimate.md)
Calculates the expected $L^2$ error from rounding detail coefficients, balancing value and positional quantization.

### [Quantized Knot Position and Coefficient Computation](4958-quantized-knot-position-and-coefficient-computation.md)
Computes quantized knots and coefficients for piecewise polynomial approximation.

## Operating characteristics
- **Time Complexity:** Not explicitly given, but the method involves iterating over intensity values and rendering $L+1$ levels, suggesting complexity proportional to the number of voxels and levels [S1].
- **Memory Complexity:** Requires storing $L+1$ separate volumes for the decomposition [S1].
- **Performance:** The method accelerates rendering by leveraging graphics hardware and multiresolution data [S1]. Load balancing across multiple GPUs is supported for large volumes [S1].
- **Strengths:** Efficient for large volumes, supports interactive adjustment of orientation and zoom, simplifies rendering by using point primitives [S1].
- **Limitations:** Uses orthogonal projection in examples, though generalizable; lacks anti-aliasing for point primitives [S1].

## Background
The method was presented in the paper "Multiresolution MIP Rendering of Large Volumetric Data Accelerated on Graphics Hardware" by [S1]. It is part of a broader approach to multiresolution volume rendering, specifically focusing on the projection of wavelet detail coefficients. The technique is designed to handle large volumetric datasets by exploiting the hierarchical nature of wavelet decompositions and accelerating the rendering process using graphics hardware [S1].

## Sources
[S1] Multiresolution MIP Rendering of Large Volumetric Data Accelerated on Graphics Hardware, https://doi.org/10.2312/vissym/eurovis07/243-250, fetched 2026-10-26
