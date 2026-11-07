# Projecting Detail Coefficients

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
    orientation_quat: Quaternion (w, x, y, z) for volume orientation
    zoom_factor: Float, default 1.0 (voxels per pixel)
    lower_threshold: Intensity value below which is background
    upper_threshold: Optional intensity value above which is max intensity
    viewport_size: Tuple (width, height)
    """
    # 1. Construct Model-View-Projection (MVP) matrix
    # Build model-view matrix from orientation and zoom
    mv_matrix = build_model_view_matrix(orientation_quat, zoom_factor)
    # Combine with standard projection matrix
    mvp_matrix = multiply_matrices(get_projection_matrix(), mv_matrix)
    
    # 2 & 3. Handle intensity iteration and upper threshold optimization
    if upper_threshold is not None:
        max_voxels = []
        for level_data in volume_levels:
            for v in level_data:
                if v.intensity >= upper_threshold:
                    max_voxels.append(v)
        if max_voxels:
            draw_point_primitives(max_voxels, mvp_matrix)
            
    # 4. Render each level to decreasing quad sizes
    for level_idx, level_data in enumerate(volume_levels):
        scale = 1.0 / (2 ** level_idx)
        quad_width = int(viewport_size[0] * scale)
        quad_height = int(viewport_size[1] * scale)
        
        level_voxels = [v for v in level_data if v.intensity >= lower_threshold]
        draw_point_primitives(level_voxels, mvp_matrix, (quad_width, quad_height))

# Standard helper definitions
def build_model_view_matrix(quat, zoom):
    # Converts quaternion to 3x3 rotation matrix, applies uniform scaling
    R = quaternion_to_rotation_matrix(quat)
    return [[R[i][j] * zoom for j in range(3)] for i in range(3)]

def get_projection_matrix():
    # Standard orthographic projection matrix
    return [[1,0,0,0], [0,1,0,0], [0,0,1,0], [0,0,0,1]]

def multiply_matrices(A, B):
    return [[sum(A[i][k]*B[k][j] for k in range(len(B))) for j in range(len(B[0]))] for i in range(len(A))]

def draw_point_primitives(voxels, mvp, target_size=None):
    for v in voxels:
        pos = [v.x, v.y, v.z, 1.0]
        proj = multiply_matrices(mvp, [[pos[i]] for i in range(4)])
        sx = proj[0][0] * (target_size[0] if target_size else 1.0)
        sy = proj[1][0] * (target_size[1] if target_size else 1.0)
        rasterize_point(sx, sy, v.intensity)

def quaternion_to_rotation_matrix(q):
    # Standard quaternion-to-rotation-matrix conversion
    w, x, y, z = q
    return [[1-2*(y**2+z**2), 2*(x*y-w*z), 2*(x*z+w*y)],
            [2*(x*y+w*z), 1-2*(x**2+z**2), 2*(y*z-w*x)],
            [2*(x*z-w*y), 2*(y*z+w*x), 1-2*(x**2+y**2)]]

def rasterize_point(x, y, intensity):
    # Rasterizes a point primitive at screen coordinates (x, y)
    # with the given intensity value
    pass
```

**Parameters:**
- Zoom factor: Default 1 voxel per pixel [S1].
- Lower threshold: User-defined or minimum intensity in volume [S1].
- Upper threshold: Optional, for maximum intensity optimization [S1].

**Edge Cases:**
- If no upper threshold is defined, the iteration proceeds normally from the lowest to the highest intensity [S1].
- The method uses point primitives without rounding or anti-aliasing, which may affect visual smoothness at non-integer zoom factors [S1].

## Subtopics

### [Coefficient Quantization Error Estimate](3334-coefficient-quantization-error-estimate.md)
Calculates the expected $L^2$ error from rounding detail coefficients, balancing value and positional quantization.

### [Quantized Knot Position and Coefficient Computation](4958-quantized-knot-position-and-coefficient-computation.md)
Computes quantized knots and coefficients for piecewise polynomial approximation.

## Operating characteristics
- **Time Complexity:** Not given in the sources. The method iterates over intensity values and renders $L+1$ levels, implying complexity proportional to the number of voxels and decomposition levels [S1].
- **Memory Complexity:** Requires storing $L+1$ separate volumes for the decomposition [S1].
- **Performance:** The method accelerates rendering by leveraging graphics hardware and multiresolution data [S1]. Load balancing across multiple GPUs is supported for large volumes [S1].
- **Strengths:** Efficient for large volumes, supports interactive adjustment of orientation and zoom, simplifies rendering by using point primitives [S1].
- **Limitations:** Uses orthogonal projection in examples, though generalizable; lacks anti-aliasing for point primitives [S1].

## Background
The method was presented in the paper "Multiresolution MIP Rendering of Large Volumetric Data Accelerated on Graphics Hardware" [S1]. It is part of a broader approach to multiresolution volume rendering, specifically focusing on the projection of wavelet detail coefficients. The technique is designed to handle large volumetric datasets by exploiting the hierarchical nature of wavelet decompositions and accelerating the rendering process using graphics hardware [S1].

## Sources
[S1] Multiresolution MIP Rendering of Large Volumetric Data Accelerated on Graphics Hardware, https://doi.org/10.2312/vissym/eurovis07/243-250, fetched 2026-10-26
