# MixRT

Also called: Mixed Neural Representations For Real-Time NeRF Rendering

## Overview
MixRT is a hybrid neural representation designed to enable real-time Neural Radiance Field (NeRF) rendering on edge devices by decomposing the scene into three complementary components: a low-quality triangle mesh for coarse geometry, a view-dependent displacement map for surface calibration, and a compressed NeRF model stored in an Instant-NGP hash table for density and color [S1]. It distinguishes itself from other hybrid approaches by explicitly leveraging distinct graphics hardware units—rasterizers, texture mapping units, and SIMD units—via WebGL APIs to bypass the computational bottlenecks of dense ray-marching, achieving real-time frame rates on edge devices [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A low-quality triangle mesh, a view-dependent displacement map, and a compressed NeRF model in Instant-NGP format [S1].
*   **Outputs:** Rendered color and density for each pixel along camera rays [S1].

**Data Structures:**
1.  **Triangle Mesh:** Standard format containing geometric vertex coordinates, texture coordinates, and polygonal face elements. Per-vertex appearance parameters are not stored [S1]. The mesh is post-processed with vertex order optimization to improve cache hit rates for neighboring triangles [S1].
2.  **View-Dependent Displacement Map:** A texture map storing spherical harmonics (SH) coefficients and scale factors, accessed via texture coordinates associated with the mesh vertices [S1].
3.  **Compressed NeRF Model:** An Instant-NGP representation consisting of a 3D embedding grid stored as a compact 1D hash table, followed by a small Multi-Layer Perceptron (MLP) [S1].
    *   **Hash Table:** Configured with a shrunk number of levels and an enlarged table size to optimize for rendering speed [S1].
    *   **MLP:** 2 hidden layers with 64 units each [S1].

**Scene Configuration:**
The unbounded scene domain is mapped into a finite sphere using a contraction function for the NeRF model components, while the low-quality mesh and displacement map remain in their original unbounded space to leverage optimized rasterization pipelines common in graphics hardware [S1].

**Rendering Pipeline (Numbered Steps):**
1.  **Mesh Rasterization:** Use rasterizer hardware to rasterize the low-quality triangle mesh and determine the ray-mesh intersection point $p$ [S1].
2.  **Displacement Calibration:** Leverage texture mapping units to access the displacement map using the texture coordinates associated with $p$. Retrieve the SH coefficients and scale, then compute the calibrated point $p_{cali}$ by applying the displacement to $p$ [S1].
3.  **Feature Retrieval:** Process $p_{cali}$ using SIMD units to query the 3D embedding grid (stored as a hash table) and retrieve embeddings for the eight closest vertices to $p_{cali}$ [S1].
4.  **Interpolation:** Perform trilinear interpolation of the eight retrieved embeddings to obtain the final feature vector for $p_{cali}$ [S1].
5.  **Color and Density Prediction:** Feed the interpolated embeddings into the small MLP (2 layers, 64 hidden units) to obtain the corresponding density and view-dependent color [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_pixel(camera_ray, mesh, displacement_map, hash_table, mlp):
    # Step 1: Mesh Rasterization
    # Standard ray-triangle intersection returning point p and UV coordinates
    p, tex_coords = ray_triangle_intersection(camera_ray, mesh)

    # Step 2: Displacement Calibration
    # Retrieve spherical harmonics (SH) coefficients and scale from displacement map
    sh_coeffs, scale = sample_texture_map(displacement_map, tex_coords)
    # Compute calibrated point p_cali using SH basis functions and scale
    # Note: Exact SH basis evaluation is not specified in the sources [G]
    p_cali = p + scale * evaluate_sh_displacement(sh_coeffs, camera_ray.direction)

    # Step 3 & 4: Feature Retrieval and Interpolation
    # Query hash table for 8 closest vertices to p_cali
    neighbor_indices = hash_table.query_8_neighbors(p_cali)
    neighbor_embeddings = [hash_table.get_embedding(idx) for idx in neighbor_indices]
    # Trilinear interpolation of the 8 embeddings
    features = trilinear_interpolate(neighbor_embeddings, p_cali)

    # Step 5: MLP Prediction
    # MLP: 2 layers, 64 hidden units
    density, color = mlp.forward(features)

    return color, density

def ray_triangle_intersection(ray, mesh):
    # Standard ray-triangle intersection returning point and UV coordinates
    # Not given in the sources.
    pass

def sample_texture_map(texture_map, uv):
    # Standard texture sampling returning SH coefficients and scale
    # Not given in the sources.
    pass

def evaluate_sh_displacement(sh_coeffs, direction):
    # Evaluate spherical harmonics basis functions for the given direction
    # Not given in the sources.
    pass

def trilinear_interpolate(embeddings, point):
    # Standard trilinear interpolation of 8 voxel embeddings
    # Not given in the sources.
    pass
```

**Parameters and Typical Values:**
*   **Mesh Complexity:** Approximately 15 MB storage size [S1].
*   **MLP Architecture:** 2 layers, 64 hidden units per layer [S1].
*   **Hash Table Configuration:** Number of levels is shrunk; hash table size is enlarged [S1].
*   **Resolution:** 1280x720 [S1].

**Edge Cases and Pitfalls:**
*   **Real-Time Failure:** Directly combining a low-quality mesh with a standard Instant-NGP hash table retains storage efficiency but fails to achieve real-time rendering speeds due to bottlenecks in the number of hash table levels [S1]. The hash table structure must be modified (shrinking levels, enlarging size) to boost speed [S1].
*   **Geometry Quality:** High-quality geometry (many triangles) is not necessary for photorealistic rendering if supported by the displacement map and hash table color field [S1].

## Operating characteristics
*   **Performance:** Achieves over 30 FPS at 1280x720 resolution on a MacBook M1 Pro laptop [S1].
*   **Quality:** Achieves 0.2 PSNR higher than state-of-the-art methods in indoor scenes of the Unbounded-360 dataset [S1].
*   **Storage:** Consumes less than 80% of the storage size of state-of-the-art methods (e.g., BakedSDF uses >300 MB, MixRT uses ~15 MB) [S1].
*   **Scaling:** The method is designed for edge devices and leverages WebGL APIs, making it compatible with cross-platform graphics libraries and existing computer graphics toolchains [S1].
*   **Strengths:** Maximizes utilization of diverse hardware resources (Rasterizer, Texture Mapping Units, SIMD Units); small storage footprint; real-time speed on consumer hardware [S1].
*   **Limitations:** Relies on the quality of the low-quality mesh for coarse geometry; performance depends on WebGL implementation efficiency on the target device [S1].

## Background
MixRT addresses the limitation of NeRF's slow rendering speed on edge devices, challenging the convention that high-quality geometry (substantial triangles) is necessary for photorealistic rendering [S1]. The method builds on the observation that a low-quality mesh combined with a hash table color field can achieve higher PSNR than a high-quality mesh with 5x more triangles while consuming only 0.26x the storage size [S1]. It further refines Instant-NGP by modifying its hash table structure to align with WebGL frameworks, enabling real-time rendering on devices equipped with browsers [S1].

## Sources
[S1] MixRT: Mixed Neural Representations For Real-Time NeRF Rendering, https://doi.org/10.1109/3dv62453.2024.00087, fetched 2026-10-27
