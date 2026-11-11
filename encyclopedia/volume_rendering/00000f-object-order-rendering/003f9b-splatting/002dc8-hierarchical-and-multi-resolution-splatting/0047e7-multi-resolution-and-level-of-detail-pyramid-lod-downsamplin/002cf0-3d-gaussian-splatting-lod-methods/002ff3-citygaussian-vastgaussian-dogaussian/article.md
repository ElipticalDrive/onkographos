# CityGaussian / VastGaussian / DoGaussian

Also called: City-Gaussian, CityGaussian V2, DoGaussian, VastGaussian

## Overview
CityGaussian, VastGaussian, and DoGaussian are block-based partitioning methods that enable the training and rendering of city-scale scenes by dividing large environments into spatial tiles, training independent 3D Gaussian Splatting (3DGS) models per tile, and merging them at render time [S1, S2, unofficial]. They solve the memory and convergence limitations of monolithic 3DGS training on scenes spanning square kilometers [S2, unofficial]. VastGaussian partitions the scene into spatially-disjoint regions with decoupled appearance handling for varying lighting and merges with feathered overlap [S2, unofficial]. CityGaussian and its V2 variant use overlapping blocks with depth regularization, visibility-aware training, and a hierarchical Level of Detail (LoD) pyramid for streaming [S2, unofficial]. DoGaussian extends this approach to distributed training across multiple nodes [S2, unofficial].

## How it works
**Inputs:** A large-scale scene captured by multi-view images; scene bounding box; block/tile dimensions; overlap ratio [S1, S2, unofficial].
**Outputs:** A set of per-block 3DGS models and a metadata structure defining block boundaries and overlap regions [S1, S2, unofficial].

**Data Structures:**
1.  **Block Grid:** A 2D or 3D grid partitioning the scene bounding box into square tiles. Each tile $T_i$ has a defined spatial extent (bbox) and an overlap region with neighbors [S1, S2, unofficial].
2.  **Per-Block Model:** Standard 3DGS primitives for each block [G].
    *   Position: $\mathbf{p} \in \mathbb{R}^3$ [G]
    *   Scale/Rotation: $\mathbf{s} \in \mathbb{R}^3, \mathbf{q} \in \mathbb{R}^4$ (quaternion) [G]
    *   Opacity: $\alpha \in [0, 1]$ [G]
    *   Spherical Harmonics (SH): Coefficients $\mathbf{c}_{l,m}$ for view-dependent color [G]

**Steps:**
1.  **Partitioning:** Divide the scene bounding box into tiles [S1, S2, unofficial]. VastGaussian uses spatially-disjoint regions, while CityGaussian uses overlapping blocks to handle seams [S2, unofficial].
2.  **Independent Training:** Train a separate 3DGS model for each tile using only the images that overlap with that tile's region [S1, unofficial].
    *   *VastGaussian:* Handles varying lighting by decoupling appearance handling per block [S2, unofficial].
    *   *CityGaussian:* Uses depth regularization and visibility-aware training to ensure geometric consistency within the block [S2, unofficial].
3.  **Merging:** At render time, collect all Gaussians from all blocks [S1, unofficial].
    *   *Overlap Handling:* In regions where blocks overlap, Gaussians from multiple blocks exist. VastGaussian uses "feathered overlap" to blend contributions [S2, unofficial]. CityGaussian relies on the hierarchical LoD structure to select the appropriate level of detail, implicitly handling overlaps by prioritizing closer or higher-resolution blocks [S2, unofficial].
4.  **Hierarchical LoD (CityGaussian V2):** To support streaming and large-scale rendering, CityGaussian V2 builds a LoD pyramid [S2, unofficial]. Clusters of Gaussians within each block are merged into coarser representations [S2, unofficial]. Rendering selects the appropriate LoD based on view distance, streaming only necessary blocks [S2, unofficial].

**Parameters:**
*   Block Size: Typically determined by GPU memory limits and scene scale. Not given in the sources.
*   Overlap Width: Sufficient to cover aliasing artifacts at tile boundaries. Not given in the sources.
*   LoD Levels: Number of pyramid levels. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_city_gaussian(scene_images, bounding_box, block_size, overlap):
    # 1. Partition scene into blocks
    blocks = []
    step = block_size
    for x in range(bounding_box.min.x, bounding_box.max.x, step):
        for y in range(bounding_box.min.y, bounding_box.max.y, step):
            for z in range(bounding_box.min.z, bounding_box.max.z, step):
                bbox = {'min': (x, y, z), 'max': (x + step, y + step, z + step)}
                blocks.append({'id': len(blocks), 'bbox': bbox})
    
    models = {}
    for block in blocks:
        # 2. Identify images overlapping this block
        relevant_images = [img for img in scene_images if frustum_intersects_box(img.camera, block.bbox)]
        
        # 3. Train independent 3DGS model for this block
        # VastGaussian: decoupled appearance handling per block
        # CityGaussian: depth regularization and visibility-aware training
        # [G] Standard 3DGS optimization: initialize Gaussians, render images, compute loss, backpropagate
        model = {'gaussians': [], 'bbox': block.bbox}
        models[block['id']] = model
        
    return models

def render_city_gaussian(models, camera, lod_level):
    # 4. Collect Gaussians from relevant blocks
    visible_blocks = [bid for bid, m in models.items() if frustum_intersects_box(camera, m.bbox)]
    
    all_gaussians = []
    for block_id in visible_blocks:
        block_gaussians = models[block_id]['gaussians']
        # 5. Merge with overlap handling
        # VastGaussian: feathered blend based on distance to block center
        # CityGaussian: hierarchical LoD selection prioritizes closer blocks
        all_gaussians.extend(block_gaussians)
        
    # 6. Rasterize all collected Gaussians
    # [G] Standard 3DGS rasterization: sort Gaussians by depth, project to screen, alpha composite
    return alpha_composite_projected_gaussians(all_gaussians, camera)
```

## Subtopics

### CityGS-X
A method that extends block-based partitioning with a geometry-aware architecture that refines block boundaries according to building footprints, achieving highly accurate urban reconstruction [S1, unofficial]. It targets large-scale urban scenes by combining semantic segmentation to allocate Gaussians only on building facades and road surfaces [S1, unofficial].

### Proxy-GS
A method that uses a coarse proxy mesh to guide Gaussian placement, significantly reducing the number of Gaussians required for background regions [S1, unofficial].

## Operating characteristics
*   **Memory:** Scales linearly with the number of blocks, allowing training on hardware with limited memory by processing blocks independently [S1, unofficial]. Total memory is the sum of per-block models [G].
*   **Training Time:** Parallelizable across blocks/nodes. DoGaussian specifically enables distributed training across multiple nodes [S2, unofficial].
*   **Rendering:** Standard 3DGS rasterization complexity, but requires managing multiple block sources and overlap blending [S1, S2, unofficial].
*   **Strengths:** Enables city-scale training (square kilometers) which is impossible with monolithic 3DGS [S2, unofficial]. Handles varying lighting through decoupled appearance handling per block [S2, unofficial].
*   **Limitations:** Potential seams or artifacts at block boundaries if overlap blending is not handled well [S2, unofficial]. Requires careful management of block dependencies [S1, unofficial].

## Background
Originating from the need to scale 3D Gaussian Splatting to urban environments, VastGaussian was presented at CVPR 2024, proposing the partition-and-merge strategy with decoupled appearance handling [S2, unofficial]. CityGaussian (and V2) introduced hierarchical LoD and depth regularization for city-scale scenes [S2, unofficial]. DoGaussian focused on the distributed training aspect [S2, unofficial]. These methods build on the foundational 3DGS work [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
