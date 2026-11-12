# Hardware-Accelerated Gaussian Rasterization

Also called: GauRast, Hardware Architecture Pipeline, Hardware-algorithm co-design for Gaussian rasterization

## Overview
Hardware-Accelerated Gaussian Rasterization is a hardware-algorithm co-design that augments standard triangle rasterizers with dedicated units for Gaussian projection and blending, achieving an order of magnitude speedup in the rasterization stage [S1, unofficial]. It solves the bottleneck of processing millions of 3D Gaussians on general-purpose hardware by isolating specific computational phases into fixed-function pipelines [S2]. Unlike software-based differentiable rasterizers [Sibling: 3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md) which rely on GPU kernels, this method uses a four-stage hardware pipeline to handle culling, projection, sorting, and blending in parallel [S2].

## How it works
**Inputs:** A set of $N$ 3D Gaussians defined by mean position $\mu$, covariance matrix $\Sigma$, opacity $\alpha$, and spherical harmonic coefficients for color [G].
**Outputs:** A 2D image buffer with alpha-blended colors [G].

**Data Structures:**
*   **Gaussian Attributes:** Stored in a global buffer [G].
*   **Tile Memory:** Each 16×16 pixel tile is assigned to an independent SRAM bank [S2].
*   **Codebook:** A lookup table used to retrieve precomputed values based on tile indices [S2].
*   **Sorting Buffer:** Holds Gaussian indices sorted by depth per tile [S2].

**Pipeline Stages:**
The architecture consists of four sequential stages [S2]:

1.  **Stage 0: Near-plane Culling (Point-based)**
    *   Eliminates parameter access for invisible regions using an AABB depth-interval test [S2].
    *   This stage processes the largest number of Gaussian points and is isolated into a dedicated stage to optimize computational resources [S2].

2.  **Stage 1: Projection (Point-based)**
    *   Projects 3D Gaussians onto the 2D image plane with zero-Jacobian skipping, using a 6×1 multiplier-accumulator array [S2].
    *   Computes the 2D covariance matrix $\Sigma'$ and mean $\mu'$ via the perspective projection Jacobian [G].

3.  **Stage 2: Sorting (Tile-based)**
    *   Sorts projected Gaussians by depth within each tile using a four-way sub-sorter parallelism design [S2].
    *   Each sub-sorter processes 256 data elements at a time [S2].
    *   The sorter outputs the current maximum depth value in a linear sorting delay, aligning with the sequential alpha-blending requirement [S2].

4.  **Stage 3: Rasterization (Tile-based)**
    *   Performs alpha-blending, processing one point at a time aligned with the sorting output [S2].
    *   Each 16×16 tile is assigned to an independent SRAM bank with a fixed entry depth and address offset controller [S2].
    *   The index is used to look up values in the codebook [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hardware_rasterization(G, C, W, H):
    # Stage 0: Near-plane Culling (Point-based)
    # Not given in the sources: exact frustum intersection logic [G]
    visible = [g for g in G if g.depth > C.near_plane and intersects_frustum(g, C)]

    # Stage 1: Projection (Point-based)
    # Not given in the sources: exact Jacobian computation [G]
    projected = []
    for g in visible:
        if not is_zero_jacobian(g, C):
            projected.append(project_gaussian(g, C))

    # Stage 2: Sorting (Tile-based)
    tile_size = 16
    tiles = {}
    for g in projected:
        key = (g.screen_x // tile_size, g.screen_y // tile_size)
        tiles.setdefault(key, []).append(g)
    # Four-way sub-sorter parallelism, 256 elements per sub-sorter [S2]
    sorted_tiles = {k: sorted(gs, key=lambda x: x.depth) for k, gs in tiles.items()}

    # Stage 3: Rasterization (Tile-based)
    image = zeros((H, W, 3))
    alpha_buf = zeros((H, W))
    for key, sorted_gs in sorted_tiles.items():
        cb = lookup_codebook(key)  # Not given in the sources: codebook lookup implementation [G]
        for g in sorted_gs:
            y_min, y_max = max(0, g.y - g.radius), min(H, g.y + g.radius)
            x_min, x_max = max(0, g.x - g.radius), min(W, g.x + g.radius)
            for y in range(y_min, y_max):
                for x in range(x_min, x_max):
                    alpha = compute_alpha(g, x, y)  # Not given in the sources: exact alpha evaluation [G]
                    if alpha > 0.0:
                        old_a = alpha_buf[y, x]
                        new_a = old_a + alpha * (1.0 - old_a)
                        image[y, x] = (image[y, x] * old_a + g.color * alpha * (1.0 - old_a)) / new_a if new_a > 0 else g.color
                        alpha_buf[y, x] = new_a
    return image
```

**Parameters:**
*   Tile Size: 16×16 pixels [S2].
*   Sub-sorter Capacity: 256 elements [S2].
*   Multiplier-Accumulator Array: 6×1 [S2].

**Edge Cases:**
*   **Zero Jacobian:** Must be detected and skipped to avoid invalid projections [S2].
*   **Depth Precision:** Sorting must handle floating-point depth precision correctly to ensure correct occlusion [G].

## Subtopics

### Compressed 3D Gaussian Splatting Renderer
A renderer for compressed scenes using GPU sorting and rasterization, enabling real-time novel view synthesis on low-end devices and integration with polygonal rendering [S3]. It renders the scene using hardware rasterization instead of the specialized compute pipeline used in original 3D Gaussian Splatting [S3]. It differs from the original compute pipeline by using hardware rasterization, allowing seamless integration into VR/AR environments or games alongside polygon models [S3].

## Operating characteristics
*   **Performance:** Achieves an order of magnitude speedup in the rasterization stage compared to software implementations [S1, unofficial].
*   **Hardware:** Designed as a fixed-function accelerator, likely on FPGA or ASIC [S2].
*   **Memory:** Uses independent SRAM banks per tile to reduce memory contention [S2].
*   **Scalability:** The point-based preprocessing scales with the number of Gaussians $N$, while the tile-based rendering scales with image resolution [S2].
*   **Limitations:** Fixed tile size (16×16) may not be optimal for all resolutions or aspect ratios [S2]. The hardware design is specific to the Gaussian Splatting algorithm and not general-purpose [S2].

## Background
This method was proposed in "A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting" [S2]. It represents a shift from software-based rasterization to hardware-algorithm co-design, aiming to overcome the limitations of GPU-based rendering for real-time applications [S1, unofficial]. The design draws inspiration from traditional triangle rasterization pipelines but adapts them for the unique properties of 3D Gaussians [S2].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
[S3] Compressed 3D Gaussian Splatting for Accelerated Novel View Synthesis, https://doi.org/10.1109/cvpr52733.2024.00985, fetched 2026-10-26
