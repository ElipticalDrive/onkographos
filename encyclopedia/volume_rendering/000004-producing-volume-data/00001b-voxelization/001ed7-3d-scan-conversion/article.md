# 3D Scan-conversion
Also called: Polygon-to-voxel conversion, Voxelization with geometric primitives

## Overview
3D scan-conversion is a rendering method that converts geometric primitives (polygons) into a volumetric dataset, combines them with existing volumetric data using volume matting operators, and renders the resulting composite ensemble using standard volume rendering techniques [S1, unofficial]. It sets itself apart from hybrid ray tracing by pre-filtering and sampling geometry into the voxel grid, allowing the use of standard ray-casting for the final image while avoiding aliasing artifacts through analytic anti-aliasing [S1, unofficial]. Unlike rasterization-based voxelization which maps to screen space, this method maps directly to the 3D volume grid coordinates [S1, unofficial].

## How it works
**Inputs:**
1. A volumetric dataset (scalar field) defined on an $N \times N \times N$ voxel array.
2. A set of embedded geometric primitives (polygons).
3. A viewing position and ray configuration.

**Outputs:**
1. A final rendered image.
2. (Intermediate) A composite volumetric dataset containing color and opacity for every voxel.

**Data Structures:**
*   **Volumetric Buffer:** An $N \times N \times N$ array storing scalar values.
*   **Color/Opacity Buffers:** Three $N \times N \times N$ arrays (or a single structure with 4 components) storing the color $c_v$ and opacity $o_v$ for the volumetric data, and $c_p, o_p$ for the polygons.
*   **Light Strength Buffer:** A 3D buffer storing fractional light strength for shadow calculations (optional, for shadowed scenes) [S1, unofficial].

**Steps:**

1.  **Classify Volumetric Data:** Shade and classify the volumetric data to yield a color $c_v$ and opacity $o_v$ for each voxel [S1, unofficial]. This typically involves trilinear interpolation of scalar values to determine material properties at sample points [S1, unofficial].

2.  **Shade and Filter Polygons:** For each polygon in the geometric set:
    a.  **Shade:** Compute the color of the polygon. The source notes homogeneous color polygons, but continuous shading is noted as under investigation [S1, unofficial].
    b.  **Filter:** Apply a 3D edge filtering algorithm to bandlimit the polygon data prior to sampling. The specific filter used is a 3D version of the algorithm by Feibush et al. [1980] [S1, unofficial].
    c.  **Convolution Kernel:** Use a constant weight ball kernel with a diameter of 2 voxels [S1, unofficial].
    d.  **Sample:** Sample the filtered polygon at the resolution of the volumetric data to yield a color $c_p$ and opacity $o_p$ for each voxel it affects [S1, unofficial].

3.  **Combine (Volume Matting):** Combine the volumetric color/opacity ($c_v, o_v$) and polygon color/opacity ($c_p, o_p$) using volume matting operators to yield a composite color $c_c$ and opacity $o_c$ for each voxel [S1, unofficial].
    *   **Ordering:** Matting is not commutative. The order in which polygons are scan-converted affects the appearance of polygon-polygon intersections. Polygons may also be matted over or under volumetric data [S1, unofficial].

4.  **Render Composite:** Ray trace, resample, and composite the composite dataset ($c_c, o_c$) without further consideration of the geometric data [S1, unofficial].
    *   **Sampling Rate:** Since the polygons have already been filtered, an image sampling rate of one ray per pixel usually suffices to prevent aliasing artifacts, even along polygonal edges [S1, unofficial].

5.  **Shadow Calculation (Optional):** If shadows are required, use a two-pass approach [Williams, 1978] storing shadow information in a 3D light strength buffer instead of a 2D shadow depth buffer [S1, unofficial]. Compute fractional light strength at every point in space [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_3d_scan_conversion(volumetric_data, polygons, view_params):
    """
    Renders a mixture of volumetric data and geometric primitives using 3D scan-conversion.
    
    Args:
        volumetric_data: 3D array of scalar values (N x N x N)
        polygons: List of polygon objects (vertices, normals, color)
        view_params: Camera position, orientation, and resolution
        
    Returns:
        final_image: Rendered 2D image
    """
    N = volumetric_data.shape[0]
    
    # Step 1: Classify Volumetric Data
    # Yield color c_v and opacity o_v for each voxel
    c_v = classify_volumetric(volumetric_data)
    o_v = classify_volumetric_opacity(volumetric_data)
    
    # Initialize composite buffers
    c_c = zeros_like(c_v)
    o_c = zeros_like(o_v)
    
    # Step 2 & 3: Process Polygons
    # Note: Order matters due to non-commutative matting
    for polygon in polygons:
        # Shade polygon
        p_color = shade_polygon(polygon)
        
        # Filter polygon using 3D edge filter (Feibush et al.)
        # Kernel: Constant weight ball, 2 voxels diameter
        p_filtered = apply_3d_edge_filter(polygon, kernel_radius=1.0) # Diameter 2 implies radius 1 in voxel units
        
        # Sample filtered polygon into voxel grid
        # Yield color c_p and opacity o_p for affected voxels
        p_color_grid, p_opacity_grid = sample_polygon_to_voxels(p_filtered, N)
        
        # Combine using volume matting operators
        # This step is not commutative
        c_c, o_c = volume_matting_combine(c_c, o_c, p_color_grid, p_opacity_grid)
        
    # Step 4: Render Composite
    # Ray trace the composite dataset (c_c, o_c)
    final_image = ray_cast_volume(c_c, o_c, view_params)
    
    return final_image

def classify_volumetric(scalar_field):
    """
    Classifies scalar field into color.
    Uses trilinear interpolation from 8 surrounding voxels for sample points.
    """
    # Implementation depends on specific transfer function
    pass

def apply_3d_edge_filter(polygon, kernel_radius):
    """
    Applies 3D edge filtering to polygon.
    """
    # Feibush et al. algorithm implementation
    pass

def sample_polygon_to_voxels(filtered_polygon, N):
    """
    Samples the filtered polygon into the N x N x N voxel grid.
    """
    # Rasterize filtered polygon into 3D grid
    pass

def volume_matting_combine(c_c, o_c, c_p, o_p):
    """
    Combines existing composite (c_c, o_c) with new polygon data (c_p, o_p).
    Uses volume matting operators [Drebin et al., 1988].
    """
    # Standard alpha compositing or matting formula
    # o_new = o_c + o_p * (1 - o_c)
    # c_new = (c_c * o_c + c_p * o_p * (1 - o_c)) / o_new
    # Handle division by zero if o_new is 0
    pass
```

**Parameters:**
*   **Filter:** 3D version of the edge filtering algorithm by Feibush et al. [1980] [S1, unofficial].
*   **Convolution Kernel:** Constant weight ball, 2 voxels in diameter [S1, unofficial].
*   **Sampling Rate:** One ray per pixel is usually sufficient [S1, unofficial].

**Pitfalls:**
*   **Matting Order:** Matting is not commutative. The order in which polygons are scan-converted affects the appearance of polygon-polygon intersections [S1, unofficial].
*   **Volumetric Interaction:** Polygons may be matted over or under volumetric data, affecting appearance [S1, unofficial].
*   **Aliasing:** If polygonal data is not sufficiently bandlimited prior to sampling, aliasing artifacts will appear [S1, unofficial].

## Subtopics

### [Hybrid Ray Tracer vs 3D Scan-Conversion](2036-hybrid-ray-tracer-vs-3d-scan-conversion.md)
This variant compares two distinct approaches for rendering mixtures of geometric primitives and volumetric data: hybrid ray tracing and 3D scan-conversion. Both methods are reasonably efficient and produce aliasing-free images, but they differ significantly in computational cost, image quality characteristics, and versatility. The choice between them depends on whether the application prioritizes the physical accuracy of ray-traced lighting and shadows or the structural clarity provided by scan-converted geometric backdrops.

## Operating characteristics
*   **Time Complexity:** Not explicitly given. Requires shading, filtering, and sampling each polygon into the volume grid, followed by standard volume ray casting.
*   **Memory Complexity:** Requires storage for the volumetric data, color/opacity buffers for the volume and polygons, and potentially a 3D light strength buffer for shadows. The 3D buffer is noted to require significantly more memory than a 2D shadow depth buffer [S1, unofficial].
*   **Performance:** Designed to be efficient. The pre-filtering of polygons allows for standard sampling rates (one ray per pixel) without aliasing [S1, unofficial].
*   **Strengths:** Produces images free from aliasing artifacts; allows use of standard volume rendering techniques for the final composite; convenient for adding shadows via 3D light strength buffers [S1, unofficial].
*   **Limitations:** Matting is non-commutative, requiring careful ordering of polygons; high memory usage for 3D shadow buffers; continuous shading is noted as under investigation, implying potential limitations with current implementations [S1, unofficial].
*   **When to Use:** When structural clarity of geometric backdrops is prioritized over physical accuracy of lighting; when aliasing-free rendering of mixed data is required without the cost of full hybrid ray tracing.
*   **When Not to Use:** When physical accuracy of ray-traced lighting and shadows is the primary goal (use Hybrid Ray Tracer instead); when memory is severely constrained (due to 3D buffers).

## Background
Originating from research in medical imaging and molecular graphics, this method was presented in a study comparing hybrid ray tracing and 3D scan-conversion for rendering mixtures of geometric and volumetric data [S1, unofficial]. It builds on volume matting operators [Drebin et al., 1988] and edge filtering algorithms [Feibush et al., 1980]. The method was developed to provide an efficient alternative to hybrid ray tracing while maintaining high image quality and avoiding aliasing artifacts [S1, unofficial].

## Sources
[S1] DTIC ADA208085: Rendering Mixtures of Geometric and Volumetric Data, https://archive.org/details/DTIC_ADA208085, fetched 2026-10-26 (unofficial)
[S2] 100-million-voxel volumetric display, https://doi.org/10.1117/12.480930, fetched 2026-11-01
