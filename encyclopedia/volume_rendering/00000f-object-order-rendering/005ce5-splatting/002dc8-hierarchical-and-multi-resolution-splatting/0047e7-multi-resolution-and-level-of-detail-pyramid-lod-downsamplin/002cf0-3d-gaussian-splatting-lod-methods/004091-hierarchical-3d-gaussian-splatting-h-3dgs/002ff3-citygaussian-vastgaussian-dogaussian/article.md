# CityGaussian / VastGaussian / DoGaussian

Also called: City-Gaussian, CityGaussian / Hierarchical 3DGS, CityGaussian V2, DoGaussian, Hierarchical 3DGS

## Overview
CityGaussian, VastGaussian, and DoGaussian are methods for large-scale urban reconstruction that partition scenes into overlapping blocks, train independent 3D Gaussian Splatting (3DGS) models per block, and merge them at rendering time using a Level-of-Detail (LoD) hierarchy [S1, S3, S4, unofficial][S6]. They solve the memory and computational bottlenecks of standard 3DGS in city-scale scenes by enabling streaming and interactive exploration through spatial partitioning and distance-based detail selection [S2, S3, unofficial][S6]. Unlike sibling methods like [Octree-GS](7341-octree-gs.md) which use anchor-based neural decoders, or [Iterative Gaussian Synopsis](9289-iterative-gaussian-synopsis.md) which compresses a global model, these methods rely on block-based training and merging with explicit LoD pyramids [S3, unofficial][S6].

## How it works
**Inputs:** A set of calibrated images covering a large-scale scene (e.g., city-scale).
**Outputs:** A set of block-specific 3DGS models and an LoD hierarchy for rendering.

**Data Structures:**
1.  **Block Partition:** The scene is divided into spatially overlapping or disjoint blocks (tiles) [S1, S3, S4, unofficial].
2.  **Block Model:** Each block contains a set of 3D Gaussians $\{g_i\}$ with parameters: position $\mathbf{p}$, covariance $\mathbf{\Sigma}$ (or rotation $\mathbf{R}$ and scale $\mathbf{s}$), opacity $\alpha$, and spherical harmonics (SH) coefficients for color [S1, unofficial][S6].
3.  **LoD Pyramid:** For each block, multiple detail levels are generated, typically by applying compression strategies (e.g., LightGaussian) to reduce point count, feature dimensions, and precision [S6].

**Steps:**
1.  **Partitioning:** Divide the scene into blocks. VastGaussian uses spatially disjoint regions with feathered overlap to handle varying lighting [S3, unofficial]. CityGaussian uses overlapping tiles [S3, unofficial].
2.  **Independent Training:** Train a separate 3DGS model for each block independently [S1, S3, S4, unofficial]. DoGaussian extends this to distributed training across multiple nodes [S3, unofficial].
3.  **LoD Generation (Optional/Variant):** Apply compression to create multiple detail levels per block. This involves reducing the number of Gaussians, lowering SH band, and quantizing parameters [S6].
4.  **Rendering (Merge & Select):**
    *   **Distance Calculation:** Calculate the distance from the camera to each block.
    *   **LoD Selection:** Select the appropriate detail level for each block based on the camera distance [S4, unofficial][S6].
    *   **Frustum Culling:** Determine if a block intersects the camera frustum.
    *   **Fusion:** Concatenate Gaussians from selected blocks and detail levels for rasterization [S6].

**Formulas:**
*   **Block Bounds Estimation (CityGS):** To avoid floaters, bounds $\mathbf{p}_{min}^j$ and $\mathbf{p}_{max}^j$ for block $j$ are estimated using Median Absolute Deviation (MAD) [S6]:
    $$
    MAD_j = \mathrm{median}\left( \left| \mathbf{p}_k^j - \mathrm{median}\left( \mathbf{p}_k^j \right) \right| \right)
    $$
    $$
    \mathbf{p}_{min}^j = \max\left( \min\left( \mathbf{p}_k^j \right), \mathrm{median}\left( \mathbf{p}_k^j \right) - n_{MAD} \times MAD_j \right)
    $$
    $$
    \mathbf{p}_{max}^j = \min\left( \max\left( \mathbf{p}_k^j \right), \mathrm{median}\left( \mathbf{p}_k^j \right) + n_{MAD} \times MAD_j \right)
    $$
    where $\mathbf{p}_k^j$ are the positions of Gaussians in block $j$, and $n_{MAD}$ is a hyperparameter [S6].

*   **LoD Selection:** The detail level for a block is determined by the minimum distance from the block's eight corners to the camera center [S6].

*   **Frustum Culling:** Project block corners to screen space, create a bounding box, and calculate Intersection-over-Union (IoU) with the screen area. Include the block if IoU indicates intersection or if the camera is inside the block [S6].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_city_gaussian(camera, scene_blocks, lod_levels):
    """
    Render a large-scale scene using block-based partitioning and LoD.
    
    Args:
        camera: Camera object with position, orientation, and projection matrix.
        scene_blocks: List of Block objects. Each Block has:
            - gaussians: List of Gaussian objects (pos, cov, alpha, sh).
            - lod_levels: List of compressed Gaussian sets (LOD 0, 1, 2...).
            - bounds: Estimated bounding box (p_min, p_max).
        lod_levels: List of detail levels. Each level is a list of Gaussians.
    
    Returns:
        Image buffer.
    """
    image = empty_buffer()
    visible_gaussians = []
    
    for block in scene_blocks:
        # 1. Estimate Block Bounds (if not pre-computed)
        # Use MAD to filter out floaters
        p_positions = [g.pos for g in block.gaussians]
        median_pos = median(p_positions)
        mad = median([abs(p - median_pos) for p in p_positions])
        p_min = max(min(p_positions), median_pos - n_MAD * mad)
        p_max = min(max(p_positions), median_pos + n_MAD * mad)
        
        # 2. Frustum Culling
        corners = get_box_corners(p_min, p_max)
        projected_corners = project_to_screen(corners, camera)
        screen_bbox = bounding_box(projected_corners)
        iou = intersection_over_union(screen_bbox, camera.screen_area)
        
        if iou > threshold or camera_inside_block(camera, p_min, p_max):
            # 3. LoD Selection
            # Calculate min distance from block corners to camera
            min_dist = min(distance(camera.pos, corner) for corner in corners)
            lod_index = select_lod_index(min_dist, block.lod_thresholds)
            
            # Get Gaussians for selected LOD
            selected_gaussians = block.lod_levels[lod_index]
            
            # 4. Add to visible list
            visible_gaussians.extend(selected_gaussians)
    
    # 5. Rasterization
    # Sort visible_gaussians by depth (standard 3DGS sorting)
    visible_gaussians.sort(key=lambda g: depth(g.pos, camera))
    
    # Render using standard 3DGS rasterizer
    image = rasterize(visible_gaussians, camera)
    
    return image
```

**Edge Cases and Pitfalls:**
*   **Floaters:** Outlier Gaussians (floaters) can distort block bounds. Using MAD for bound estimation mitigates this [S6].
*   **Overlap Artifacts:** Overlapping blocks may cause double rendering or seams. Feathered overlap or careful merging is required [S3, unofficial].
*   **Precision:** Lower detail levels use quantized parameters (e.g., 8-bit for $\alpha$, INT6 for SH), which may introduce visible artifacts if compression is too aggressive [S6].

## Subtopics

### Proxy-GS
Proxy-GS uses a coarse proxy mesh to guide Gaussian placement, significantly reducing the number of Gaussians required for background regions [S1, unofficial]. It sets itself apart by leveraging geometric priors from a mesh to optimize Gaussian density, rather than relying solely on image-based optimization. It is chosen when a coarse mesh is available and background regions are large and static [S1, unofficial].

### CityGS Level-of-Detail (LoD)
CityGS Level-of-Detail (LoD) is a rendering optimization technique that generates multiple detail levels of Gaussians and selects them per block based on distance to the camera, reducing rasterizer load [S6]. It sets itself apart by using the LightGaussian compression strategy to create levels with different compression rates ($r_0, r_1, r_2$), reducing points, feature dimensions, and precision [S6]. It is chosen for real-time rendering of very large scenes where memory bandwidth is a bottleneck [S6].

## Operating characteristics
*   **Time Complexity:** Training is linear in the number of blocks and Gaussians per block, but parallelizable across blocks [S1, S3, unofficial]. Rendering complexity depends on the number of visible Gaussians, which is reduced by LoD selection [S6].
*   **Memory Complexity:** VRAM usage is proportional to the number of visible Gaussians at the selected LoD, significantly lower than loading all Gaussians [S3, unofficial][S6].
*   **Measured Performance:** CityGaussian enables interactive exploration of square-kilometer scenes [S3, unofficial]. DoGaussian supports distributed training across nodes [S3, unofficial].
*   **Strengths:** Scalability to city-scale scenes, parallelizable training, reduced VRAM usage.
*   **Limitations:** Potential seams at block boundaries, complexity in managing overlapping blocks, dependency on block partitioning quality [S1, S3, unofficial].
*   **When to use:** Large-scale urban reconstruction, autonomous driving simulation, scenes with significant scale variation.
*   **When not to use:** Small-scale scenes where standard 3DGS is sufficient, scenes with high dynamic content requiring global consistency (unless combined with methods like StreetGaussians) [S1, S3, unofficial].

## Background
CityGaussian, VastGaussian, and DoGaussian originated from the need to scale 3D Gaussian Splatting to city-scale environments. VastGaussian was introduced in CVPR'24, partitioning scenes into spatially-disjoint regions with decoupled appearance handling [S3, unofficial]. CityGaussian and its V2 introduced hierarchical LoD across square kilometers with depth regularization and visibility-aware training [S3, unofficial]. DoGaussian extended this to distributed training across nodes [S3, unofficial]. These methods build on the principles of Hierarchical 3D Gaussian Splatting (H-3DGS) [10279-hierarchical-3d-gaussian-splatting-h-3dgs.md] and address the memory bottlenecks of standard 3DGS in large scenes [S1, S3, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S4] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
[S5] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
[S6] CityGaussian: Real-Time High-Quality Large-Scale Scene Rendering with Gaussians, https://doi.org/10.1007/978-3-031-72640-8_15, fetched 2026-10-26
