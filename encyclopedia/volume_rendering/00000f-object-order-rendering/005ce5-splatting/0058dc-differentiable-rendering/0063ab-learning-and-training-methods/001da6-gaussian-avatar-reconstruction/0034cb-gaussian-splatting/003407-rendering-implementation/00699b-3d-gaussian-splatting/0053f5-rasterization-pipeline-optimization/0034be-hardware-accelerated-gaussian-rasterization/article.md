# Hardware-Accelerated Gaussian Rasterization

Also called: GauRast, Hardware Architecture Pipeline, Hardware-algorithm co-design for Gaussian rasterization

## Overview
Hardware-Accelerated Gaussian Rasterization is a hardware-algorithm co-design that augments standard triangle rasterizers with dedicated units for Gaussian projection and blending, achieving an order of magnitude speedup in the rasterization stage [S1, unofficial][S2]. It solves the bottleneck of rendering millions of 3D Gaussians on general-purpose GPUs by offloading specific geometric operations to fixed-function hardware. It differs from general compute-pipeline approaches by isolating preprocessing (point-based) from rendering (tile-based) into a four-stage pipeline, and from sibling methods like [Speedy-Splat Rendering Pipeline](3543-speedy-splat-rendering-pipeline.md) by its specific focus on dedicated silicon units for projection and sorting rather than purely algorithmic software optimizations.

## How it works
The method implements a four-stage hardware pipeline [S2]. The first two stages constitute preprocessing (point-based), and the last two constitute rendering (tile-based) [S2].

**Inputs and Outputs:**
*   **Input:** 3D Gaussian parameters (position, covariance, opacity, spherical harmonics coefficients) and camera matrices.
*   **Output:** Final 2D image pixels.

**Data Structures:**
*   **Tile Assignment:** The image is divided into $16 \times 16$ pixel tiles [S2]. Each tile is assigned to an independent SRAM bank with a fixed entry depth and address offset controller [S2].
*   **Codebook:** An index is used to look up values stored in a codebook within the SRAM [S2].
*   **Sorting Buffer:** The sorting unit outputs the current maximum value in a linear sorting delay, aligning with the alpha-blending process [S2].

**Pipeline Stages:**

1.  **Stage 0: Near-plane Culling**
    *   Eliminates parameter access for invisible regions using an Axis-Aligned Bounding Box (AABB) depth-interval test [S2].
    *   This stage is isolated to handle the largest number of Gaussian points efficiently [S2].

2.  **Stage 1: Projection**
    *   Projects 3D Gaussians onto the 2D image plane [S2].
    *   Uses a 6x1 multiplier-accumulator array [S2].
    *   Implements zero-Jacobian skipping to optimize computation [S2].

3.  **Stage 2: Gaussian Sorting**
    *   Sorts Gaussians per tile to enable correct front-to-back alpha compositing [S2].
    *   Uses a four-way sub-sorter parallelism based on a comparison-free sorter design [S2].
    *   Each sub-sorter processes 256 data elements at a time [S2].
    *   The design outputs the current maximum value in a linear sorting delay, which aligns with the rasterization unit's need to process one point at a time [S2].

4.  **Stage 3: Rasterization**
    *   Performs alpha-blending, processing one point at a time aligned with the sorting output [S2].
    *   Uses the tile-based model where each tile operates on its assigned SRAM bank [S2].

**Alpha Compositing:**
The final pixel color $C(p)$ is computed via front-to-back blending [S1, unofficial]:
$$ C(p) = \sum_{i \in \mathcal{N}} c_i \alpha_i(p) \prod_{j=1}^{i-1} (1 - \alpha_j(p)) $$
where $\mathcal{N}$ is the set of surviving Gaussians in the tile, sorted by depth [S1, unofficial]. The opacity $\alpha_i(p)$ is defined as:
$$ \alpha_i(p) = \sigma_i g_i(p) $$
where $\sigma_i$ is the Gaussian opacity and $g_i(p)$ is the projected 2D Gaussian value at pixel $p$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Hardware Pipeline Simulation
# Stage 0: Near-plane Culling
def stage_0_cull(gaussians, camera_near_plane):
    """
    Eliminates Gaussians outside the near-plane using AABB depth test.
    """
    valid_gaussians = []
    for g in gaussians:
        # AABB depth-interval test
        if g.min_depth > camera_near_plane:
            valid_gaussians.append(g)
    return valid_gaussians

# Stage 1: Projection
def stage_1_project(gaussians, camera_matrix):
    """
    Projects 3D Gaussians to 2D. Uses 6x1 MAC array and skips zero Jacobian.
    """
    projected_gaussians = []
    for g in valid_gaussians:
        # 3D to 2D projection with covariance transformation
        # Skip if Jacobian determinant is zero
        if g.jacobian_determinant != 0:
            g_2d = project_3d_to_2d(g, camera_matrix)
            projected_gaussians.append(g_2d)
    return projected_gaussians

# Stage 2: Sorting
def stage_2_sort(projected_gaussians, tile_size=16):
    """
    Sorts Gaussians per tile using comparison-free sub-sorters.
    Outputs max value in linear delay.
    """
    tiles = create_tiles(projected_gaussians, tile_size)
    sorted_tiles = []
    for tile in tiles:
        # Four-way sub-sorter parallelism, 256 elements per sub-sorter
        sorted_gaussians = comparison_free_sort(tile.gaussians)
        sorted_tiles.append(sorted_gaussians)
    return sorted_tiles

# Stage 3: Rasterization
def stage_3_rasterize(sorted_tiles, image_width, image_height):
    """
    Performs alpha-blending per tile using SRAM codebook lookup.
    """
    image = zeros((image_height, image_width, 3))
    for tile in sorted_tiles:
        # Retrieve tile from SRAM bank
        tile_data = lookup_sram(tile.index)
        # Process one point at a time aligned with sorting output
        for g in tile.sorted_gaussians:
            # Alpha blending
            alpha = g.opacity * gaussian_value(g, pixel)
            color = g.color
            # Update image buffer
            update_pixel_alpha_blend(image, pixel, color, alpha)
    return image
```

## Subtopics

### Compressed 3D Gaussian Splatting Renderer
A renderer for compressed scenes using GPU sorting and rasterization, enabling real-time novel view synthesis on low-end devices and integration with polygonal rendering [S3]. It renders the scene using hardware rasterization instead of the specialized compute pipeline used in original 3D Gaussian Splatting [S3]. This differs from the original compute pipeline by using hardware rasterization, allowing seamless integration into VR/AR environments or games alongside polygon models [S3].

### LS-Gaussian Hardware Architecture
A dedicated hardware accelerator designed to accelerate 3DGS by enabling early stages to process subsequent frames while later stages execute previous ones, improving parallelism and pipeline efficiency [S4]. It enhances the Culling and Conversion Unit (CCU) by replacing GSCore’s dual OBB Intersection Test Units (OIUs) with a square root and logarithmic operator to support the two-stage accurate intersection test [S4]. The Viewpoint Transformation Unit (VTU) performs sparse rendering by retrieving pixel colors and maximum depths from the reference frame and performing three matrix multiplications: mapping 2D pixels to 3D space, transforming the point cloud, and reprojecting onto the new camera plane [S4]. The VTU Tile Classification uses a counter array to track validly projected pixels per tile; if the count exceeds 56 (of total pixels in tile), the tile is interpolated, otherwise it is forwarded to the Volume Rendering Unit (VRU) for full re-rendering [S4]. The architecture includes CCU, Load Distribution Unit (LDU), VTU, VRU, Sorting Unit, and Interpolation Unit [S4].

## Operating characteristics
*   **Performance:** Achieves an order of magnitude speedup in the rasterization stage compared to standard compute pipelines [S1, unofficial].
*   **Hardware:** Implemented as a dedicated hardware accelerator, often built upon existing graphics cores (e.g., GSCore) with enhanced units [S4].
*   **Complexity:** The pipeline isolates near-plane culling to handle the largest number of points, optimizing computational resources [S2].
*   **Limitations:** Data dependencies in the multiplier-accumulator arrays limit the ability to fully parallelize operations, as certain computations must wait for completion of others [S2].
*   **Use Case:** Suitable for real-time rendering on constrained hardware or when integration with standard polygonal rasterization is required [S3].

## Background
The concept of hardware-accelerated Gaussian rasterization emerged as a response to the high computational cost of rendering 3D Gaussian Splatting on general-purpose GPUs. Early works like GauRast proposed augmenting standard triangle rasterizers with dedicated units for Gaussian projection and blending [S1, unofficial]. Subsequent developments, such as LS-Gaussian, focused on streaming mechanisms and specific hardware enhancements like square root and logarithmic operators to improve pipeline efficiency and reduce stalls [S4]. The approach represents a shift from purely algorithmic optimizations to hardware-algorithm co-design [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
[S3] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
[S4] No Redundancy, No Stall: Lightweight Streaming 3D Gaussian Splatting for Real-time Rendering, https://doi.org/10.1109/iccad66269.2025.11240950, fetched 2026-10-30
