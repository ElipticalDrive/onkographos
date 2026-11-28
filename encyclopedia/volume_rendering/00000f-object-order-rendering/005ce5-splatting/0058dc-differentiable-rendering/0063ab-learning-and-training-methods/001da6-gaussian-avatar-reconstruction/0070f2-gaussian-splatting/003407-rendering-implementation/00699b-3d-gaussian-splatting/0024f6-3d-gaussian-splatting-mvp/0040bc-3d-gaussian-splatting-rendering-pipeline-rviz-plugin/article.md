# 3D Gaussian Splatting Rendering Pipeline (RViz Plugin)
Also called: SplatCloud rendering

## Overview
This article describes the OpenGL-based rendering pipeline for visualizing 3D Gaussian Splatting data within the ROS RViz environment [S1, unofficial]. It solves the problem of efficiently rendering millions of 2D-projected Gaussian splats as a single instanced draw call while handling depth sorting and view-dependent shading [S1, unofficial]. It differs from pure software implementations by leveraging GPU-based sorting and texture buffer objects (TBOs) for data storage, and from other RViz plugins by supporting both exact back-to-front sorting and Order-Independent Transparency (WBOIT) approximations [S1, unofficial].

## How it works
The pipeline renders a cloud of 3D Gaussians by projecting them into 2D screen space, sorting them by depth, and accumulating their colors with transparency [S1, unofficial].

### Inputs and Outputs
*   **Input**: A set of $N$ 3D Gaussians defined by position, orientation (quaternion), scale, opacity, and spherical harmonic (SH) coefficients for color [S1, unofficial].
*   **Output**: A 2D image frame rendered by the RViz viewport [S1, unofficial].

### Data Structures
The plugin uses specific GPU memory layouts to minimize bandwidth and decouple sorting from rendering [S1, unofficial].

1.  **Base Splat Data (TBO)**:
    *   Stored in a Texture Buffer Object (TBO) bound to texture unit 0 immediately before the draw call via the `notifyRenderSingleObject` callback [S1, unofficial].
    *   Layout: 2 `uvec4` texels per splat [S1, unofficial].
    *   Size: 32 bytes per splat [S1, unofficial].
    *   Content: Packed `SplatGPU` data including position, covariance/rotation, scale, and opacity [S1, unofficial].

2.  **Spherical Harmonics Data (TBO)**:
    *   Stored in a separate RGBA16F TBO [S1, unofficial].
    *   Number of coefficients per splat: $C = (d_{active} + 1)^2 - 1$, where $d_{active}$ is the active SH degree [S1, unofficial].
    *   Coefficients are interleaved RGB, coefficient-major [S1, unofficial].
    *   The DC term (degree 0) is always present [S1, unofficial].
    *   Higher-order terms are stored if available in the source data [S1, unofficial].

3.  **Sorted Indices (VBO)**:
    *   A dedicated Vertex Buffer Object (`index_vbo_`) storing per-instance sorted indices [S1, unofficial].
    *   This decouples the sorting process from the rendering pipeline, allowing the sort backend to be swapped without reloading base splat data [S1, unofficial].
    *   Indices are sorted back-to-front for exact transparency or used for WBOIT weighting [S1, unofficial].

4.  **Splat Centers Cache**:
    *   A `std::vector` (`centers_cache_`) storing splat centers [S1, unofficial].
    *   Used to allow swapping the sort backend (CPU/CUDA) without reloading the full base splat data [S1, unofficial].

### Rendering Steps

1.  **Data Upload**:
    *   Convert raw PLY data (positions, quaternions, scales, opacity, SH coefficients) into the compact TBO layout [S1, unofficial].
    *   Quaternions are normalized. Scales are exponentiated from log-space. Opacity is mapped to `uint8` [S1, unofficial].
    *   Upload the base splat data to the primary TBO [S1, unofficial].

2.  **Sorting**:
    *   The `SplatCloud` class manages a worker thread that drives the sort backend [S1, unofficial].
    *   Sorting can be performed on CPU or GPU (CUDA) [S1, unofficial].
    *   The CUDA backend automatically falls back to CPU if no GPU device is available [S1, unofficial].
    *   The sorter outputs a list of indices representing the back-to-front order of splats [S1, unofficial].
    *   These indices are uploaded to the `index_vbo_` [S1, unofficial].

3.  **Draw Call**:
    *   Bind the base splat TBO to texture unit 0 [S1, unofficial].
    *   Bind the sorted indices VBO.
    *   Execute an instanced draw call using a quad VBO (4 vertices, ±2 corners), an index buffer (6 indices for two triangles), and the instance index VBO [S1, unofficial].
    *   The vertex shader uses the instance index to fetch the sorted splat ID, then fetches the corresponding splat data from the TBO [S1, unofficial].

4.  **Fragment Shading**:
    *   The fragment shader evaluates the 2D Gaussian profile [S1, unofficial].
    *   If the fragment's alpha is below `alpha_threshold_` (default 0.05), it is discarded [S1, unofficial].
    *   Color is computed using the SH coefficients based on the view direction and `active_sh_degree_` (default 0, max constrained by data) [S1, unofficial].

5.  **Transparency Accumulation**:
    *   Two modes are supported, configurable at runtime [S1, unofficial]:
        *   **Sorted**: Standard back-to-front alpha blending.
        *   **WBOIT**: Order-Independent Transparency approximation using a weight function based on depth and alpha [S1, unofficial].

### Formulas

**Spherical Harmonics Coefficient Count**:
$$ N_{coeffs} = (d_{active} + 1)^2 - 1 $$
Where $d_{active}$ is the active SH degree [S1, unofficial].

**WBOIT Weight Function**:
The exact weight function formula is not provided in the sources [S1, unofficial]. The accumulation uses configurable parameters for depth discrimination and alpha suppression [S1, unofficial].

**WBOIT Alpha Discard**:
Fragments with alpha below `wboit_alpha_discard_` (default 0.01, range [0.0, 0.1]) are discarded in the accumulation pass [S1, unofficial].

### Parameters

| Parameter | Type | Default | Range/Notes |
| :--- | :--- | :--- | :--- |
| `alpha_threshold_` | float | 0.05 | Opacity threshold for discarding splat fragments [S1, unofficial]. |
| `active_sh_degree_` | int | 0 | SH degree for view-dependent color. Max constrained by data [S1, unofficial]. |
| `clip_min_` | vec3 | [-10, -10, -10] | AABB lower bound for ROI clipping [S1, unofficial]. |
| `clip_max_` | vec3 | [10, 10, 10] | AABB upper bound for ROI clipping [S1, unofficial]. |
| `wboit_weight_scale_` | float | 5.0 | Depth-discrimination strength [S1, unofficial]. |
| `wboit_weight_exponent_` | float | 2.0 | Alpha-suppression power [S1, unofficial]. |
| `wboit_alpha_discard_` | float | 0.01 | Fragment alpha cutoff in WBOIT pass [S1, unofficial]. |

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_splat_cloud(splat_cloud, camera, viewport):
    # 1. Prepare Sorter
    sorter = splat_cloud.get_sorter()
    if sorter is None:
        return  # Sorting disabled

    # 2. Trigger Sort (Async/Threaded)
    # The sorter uses centers_cache_ to sort indices
    sorted_indices = sorter.sort(splat_cloud.centers_cache_)
    
    # 3. Upload Indices
    index_vbo = splat_cloud.index_vbo_
    index_vbo.upload(sorted_indices)
    
    # 4. Setup Rendering State
    program = splat_cloud.program
    
    # Bind Base Splat Data TBO to Texture Unit 0
    bind_texture_buffer(program.uniforms["splat_data"], 0, splat_cloud.base_tbo)
    
    # Bind SH Data TBO
    bind_texture_buffer(program.uniforms["sh_data"], 1, splat_cloud.sh_tbo)
    
    # Set Parameters
    program.uniforms["alpha_threshold"] = splat_cloud.alpha_threshold_
    program.uniforms["active_sh_degree"] = splat_cloud.active_sh_degree_
    
    # Set WBOIT parameters if mode is WBOIT
    if splat_cloud.transparency_mode == 'WBOIT':
        program.uniforms["wboit_weight_scale"] = splat_cloud.wboit_weight_scale_
        program.uniforms["wboit_weight_exponent"] = splat_cloud.wboit_weight_exponent_
        program.uniforms["wboit_alpha_discard"] = splat_cloud.wboit_alpha_discard_
    else:
        # Enable standard blending for Sorted mode
        enable_blending()
        set_blend_func(SRC_ALPHA, ONE_MINUS_SRC_ALPHA)

    # 5. Draw Call
    # Quad VBO (Source 0, 4 verts), Index VBO (Source 1, N floats), IBO (6 indices)
    bind_vertex_array(splat_cloud.quad_vbo)
    bind_index_buffer(splat_cloud.ibo)
    bind_instance_buffer(splat_cloud.index_vbo)
    
    # Set up vertex attributes for quad positions and instance indices
    setup_vertex_attribute(0, 2, FLOAT, 0, 0)  # Quad positions
    setup_vertex_attribute(1, 1, FLOAT, 0, 0)  # Instance indices
    
    draw_elements_instanced(
        TRIANGLES,
        6,  # IBO size
        UNSIGNED_INT,
        0,
        len(sorted_indices)  # N instances
    )
```

### Edge Cases and Pitfalls
*   **Precision**: Opacity is stored as `uint8` in the TBO, which may limit dynamic range compared to float32 [S1, unofficial].
*   **Boundaries**: ROI clipping is handled via `clip_min_` and `clip_max_`. Splats outside this AABB are not rendered [S1, unofficial].
*   **SH Degree**: `active_sh_degree_` defaults to 0 and is constrained by the maximum available in the loaded data to prevent out-of-bounds reads [S1, unofficial].
*   **Sort Backend**: If CUDA is selected but no GPU is available, the system falls back to CPU sorting, which may be significantly slower for large $N$ [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Sorting is $O(N \log N)$ or $O(N)$ depending on the backend [G]. Rendering is $O(N)$ for the draw call [G].
*   **Memory Complexity**: $O(N)$ for TBOs and VBOs. 32 bytes per splat for base data plus SH coefficients [S1, unofficial].
*   **Performance**: Depends heavily on the sort backend. GPU sorting (CUDA) is significantly faster for large clouds [G]. WBOIT mode may be slower than sorted mode due to additional accumulation passes or complex fragment shader logic, but provides order-independent results [G].
*   **Strengths**: Efficient instanced rendering, decoupled sorting, configurable transparency [S1, unofficial].
*   **Limitations**: Requires OpenGL support. WBOIT is an approximation. CPU sorting is slow for large datasets [G].

## Background
This plugin is part of the `splat-ros` ecosystem, providing a visualization tool for 3D Gaussian Splatting data within ROS [S1, unofficial]. It bridges the gap between offline reconstruction (e.g., from Gaussian Splatting training) and interactive 3D visualization in robotics workflows [G].

## Sources
[S1] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
