# MixRT

Also called: Mixed Neural Representations For Real-Time NeRF Rendering

## Overview
MixRT is a hybrid neural representation designed to enable real-time Neural Radiance Field (NeRF) rendering on edge devices by leveraging standard graphics hardware pipelines [S1]. It solves the trade-off between rendering quality, speed, and storage by combining three components: a low-quality triangle mesh for coarse geometry, a view-dependent displacement map for surface calibration, and a compressed NeRF model stored in an Instant-NGP hash table for density and color [S1]. Unlike pure MLP-based NeRFs or high-fidelity baked representations, MixRT offloads ray intersection to rasterizers, surface refinement to texture mapping units, and feature extraction to SIMD units, achieving over 30 FPS at 1280x720 resolution with storage sizes less than 80% of state-of-the-art methods [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of calibrated camera poses and images; a pre-optimized low-quality mesh; a view-dependent displacement map; and a compressed NeRF model (Instant-NGP format).
*   **Outputs:** A rendered RGB image for a given camera viewpoint.

**Data Structures:**
1.  **Low-Quality Mesh:**
    *   Format: Standard triangle mesh.
    *   Fields: Geometric vertex coordinates, texture coordinates, and polygonal face elements (indices).
    *   Constraint: Per-vertex appearance parameters are **not** stored [S1].
    *   Optimization: Vertices are post-processed by vertex order optimization to maximize cache hit rates for neighboring triangle access [S1].
2.  **View-Dependent Displacement Map:**
    *   Format: Texture map (2D image).
    *   Content: Spherical Harmonics (SH) coefficients and scale factors.
    *   Access: Indexed by texture coordinates derived from the ray-mesh intersection point [S1].
3.  **Compressed NeRF Model (Instant-NGP Format):**
    *   Structure: A 3D embedding grid stored as a compact 1D hash table [S1].
    *   Configuration: The number of levels is shrunk, and the hash table size is enlarged to boost rendering speed [S1].
    *   MLP Head: A small Multi-Layer Perceptron with 2 layers and 64 hidden units each [S1]. (Contrast with vanilla NeRF's 10 layers/256 units [S1]).

**Rendering Pipeline Steps:**
The rendering process maps the scene domain and processes rays through hardware-accelerated stages [S1]:

1.  **Scene Configuration:**
    *   The unbounded scene domain is mapped into a finite sphere using the contraction function from [4] for the NeRF model [S1].
    *   The low-quality mesh and displacement map remain in their original, unbounded space to leverage optimized rasterization pipelines [S1].

2.  **Ray-Mesh Intersection (Rasterizer Hardware):**
    *   For each pixel, a ray is cast.
    *   The rasterizer hardware performs mesh rasterization to determine the intersection point $p$ on the low-quality mesh [S1].

3.  **Surface Calibration (Texture Mapping Units):**
    *   Texture coordinates are derived from the intersection point $p$.
    *   Texture mapping units access the displacement map using these coordinates to retrieve SH coefficients and scale [S1].
    *   The calibrated point $p_{cali}$ is computed by applying the displacement to $p$ [S1].

4.  **Feature Extraction (SIMD Units):**
    *   The calibrated point $p_{cali}$ is processed by SIMD units [S1].
    *   The 3D coordinates of $p_{cali}$ are hashed to determine the table index in the Instant-NGP hash table [S1].
    *   Embeddings for the eight closest vertices of the 3D grid are retrieved from the hash table [S1].
    *   Trilinear interpolation is performed on these eight embeddings to obtain the final feature vector for $p_{cali}$ [S1].

5.  **Density and Color Prediction (MLP):**
    *   The interpolated embeddings are fed into the small MLP (2 layers, 64 hidden units) [S1].
    *   The MLP outputs the volume density $\sigma$ and view-dependent color $\mathbf{c}$ [S1].

6.  **Volume Rendering:**
    *   Standard volume rendering integration is performed using the density and color to produce the final pixel color [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_mixrt(camera_ray, mesh, displacement_map, hash_table, mlp_head):
    """
    Renders a single ray using the MixRT representation.
    
    Args:
        camera_ray: Ray origin and direction.
        mesh: Triangle mesh with vertices, faces, and texture coords.
        displacement_map: Texture map containing SH coefficients and scale.
        hash_table: Instant-NGP 1D hash table storing 3D grid embeddings.
        mlp_head: Small MLP (2 layers, 64 hidden units).
        
    Returns:
        color: Rendered RGB color.
    """
    # Step 1: Ray-Mesh Intersection via Rasterizer
    # The rasterizer hardware finds the intersection point p on the mesh.
    # This step replaces explicit ray-triangle intersection tests.
    p = rasterize_mesh_intersection(camera_ray, mesh)
    
    # Step 2: Surface Calibration via Texture Mapping
    # Retrieve texture coordinates corresponding to point p on the mesh.
    tex_coords = get_texture_coords(p, mesh)
    
    # Access displacement map to get SH coefficients and scale.
    # This leverages hardware texture sampling units.
    sh_coeffs, scale = sample_displacement_map(tex_coords, displacement_map)
    
    # Compute the calibrated point p_cali.
    # The exact displacement formula is not specified in sources, 
    # but it adjusts p based on view-dependent SH data [S1].
    p_cali = apply_displacement(p, sh_coeffs, scale)
    
    # Step 3: Feature Extraction via SIMD/Hash Table
    # Hash the 3D coordinates of p_cali to find the table index.
    # This step leverages SIMD units for parallel hashing and lookup.
    table_index = hash_coordinates(p_cali)
    
    # Retrieve embeddings for the 8 closest vertices in the 3D grid.
    # The hash table stores embeddings at grid vertices.
    embeddings = get_8_nearest_embeddings(p_cali, table_index, hash_table)
    
    # Trilinear interpolation of the 8 embeddings.
    features = trilinear_interpolate(embeddings, p_cali)
    
    # Step 4: MLP Prediction
    # Feed features into the small MLP.
    density, color = mlp_head(features)
    
    # Step 5: Volume Rendering (Standard NeRF Integration)
    # Integrate density and color along the ray segment.
    # Note: MixRT primarily uses the mesh intersection for the primary 
    # surface, but density is still used for volume rendering if 
    # transmittance is not zero.
    # For simplicity, assuming the mesh defines the primary surface 
    # and density handles transparency/occlusion.
    alpha = 1.0 - exp(-density * step_size)
    final_color = color * alpha
    
    return final_color
```

**Parameters and Typical Values:**
*   **Mesh Complexity:** Approximately 15 MB storage size [S1].
*   **MLP Architecture:** 2 layers, 64 hidden units per layer [S1].
*   **Hash Table Configuration:** Number of levels is shrunk; hash table size is enlarged [S1].
*   **Resolution:** 1280x720 [S1].
*   **Hardware:** WebGL-based framework on edge devices (e.g., MacBook M1 Pro) [S1].

**Edge Cases and Pitfalls:**
*   **Direct Combination Failure:** Directly combining a low-quality mesh with a standard Instant-NGP hash table retains storage efficiency but fails to achieve real-time rendering speeds [S1]. The hash table structure must be modified (shrinking levels, enlarging size) to meet performance targets [S1].
*   **Mesh Quality vs. Quality:** A low-quality mesh with a hash table color field achieves ~0.3 higher PSNR than a high-quality mesh with 5x more triangles, while consuming only 0.26x storage size [S1]. However, the mesh must still provide a reasonable coarse geometry for the rasterizer to function effectively [S1].
*   **Contraction Function:** The NeRF model uses a contraction function to map the unbounded scene to a finite sphere, while the mesh and displacement map remain in original space. This mismatch requires careful handling of coordinate systems during the calibration step [S1].

## Operating characteristics
*   **Performance:** Achieves over 30 FPS at 1280x720 resolution on a MacBook M1 Pro laptop [S1].
*   **Quality:** Achieves 0.2 PSNR higher than state-of-the-art methods in indoor scenes of the Unbounded-360 dataset [S1].
*   **Storage:** Consumes less than 80% of the storage size of state-of-the-art methods (e.g., BakedSDF [38] uses >300 MB, MixRT uses ~15 MB for the mesh component) [S1].
*   **Hardware Utilization:** Leverages rasterizers, texture mapping units, and SIMD units via WebGL APIs, making it suitable for edge devices and cross-platform compatibility [S1].
*   **Scaling:** The hash table configuration (shrinking levels, enlarging size) is critical for scaling rendering speed [S1].
*   **Strengths:** Real-time rendering on edge devices, high storage efficiency, good rendering quality.
*   **Limitations:** Relies on WebGL and standard graphics hardware; may not leverage the full power of high-end GPUs (like RTX 3090Ti) as effectively as native implementations [S1].

## Background
MixRT was proposed to address the slow rendering speed of NeRF on edge devices, which has hampered practical applications in immersive interactions [S1]. It challenges the convention that high-quality geometry (meshes with substantial triangles) is necessary for photorealistic rendering [S1]. The method builds on Instant-NGP [24] for compressed NeRF representation and uses a contraction function from [4] for scene mapping. It is designed to be compatible with existing computer graphics toolchains, such as collision detection [17] [S1].

## Sources
[S1] MixRT: Mixed Neural Representations For Real-Time NeRF Rendering, https://doi.org/10.1109/3dv62453.2024.00087, fetched 2026-10-27
