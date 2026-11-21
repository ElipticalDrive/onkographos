# Tile and Slide Framework

Also called: Snake-NeRF

## Overview
The Tile and Slide Framework (Snake-NeRF) is an out-of-core method that scales Neural Radiance Fields (NeRFs) to massive 3D scenes (e.g., 10km x 10km) by partitioning the volume into non-overlapping 3D tiles, each trained with its own independent NeRF [S1]. Unlike sibling methods that dispatch multiple NeRFs across multiple GPUs or require pre-training a global model, this framework operates on a single GPU device, eliminating the need to load all images or network weights simultaneously [S1]. It solves the problem of catastrophic forgetting and memory overflow in large-scale training by using a 2x2 sliding window progression and a segmented ray sampler to ensure seamless rendering across tile boundaries [S1].

## How it works
The method partitions the scene into a grid of independent NeRFs, ensuring that each tile learns only its local geometry and appearance, while a global color network handles view-dependent shading [S1].

### Inputs and Outputs
*   **Inputs:** A set of $N_{im}$ satellite images covering a Region of Interest (ROI) with area $A_{ROI}$ and spatial resolution $SR$ (pix/m). The total input size is defined as $N = N_{im} \times A_{ROI} \times SR^2$ pixels [S1].
*   **Outputs:** A set of non-overlapping NeRF models, one for each 3D tile, capable of synthesizing novel views within their respective domains [S1].

### Data Structures
The framework relies on the following data structures, building on the [Voxel and Sparse Grid Representations](13738-voxel-and-sparse-grid-representations.md) principles:

1.  **Tiling Grid:** A square grid aligned with East/North (UTM coordinates) [S1]. The scene is divided into non-overlapping 3D bounding boxes (tiles).
2.  **Per-Tile NeRF:** Each tile contains a dedicated NeRF model. The source notes that these NeRFs typically use multi-resolution hash tables to encode geometry combined with a global color network, similar to NeRF-XL [S1].
3.  **Image Cropping Buffer:** Images are cropped with overlap to ensure each NeRF receives all necessary pixels for its domain [S1].
4.  **Ray Coordinates:** Ray origins and directions are computed on-the-fly using Rational Polynomial Coefficients (RPCs) for each image, rather than stored in memory [S1].

### Numbered Steps

1.  **Scene Partitioning:** Divide the 3D scene of interest into a grid of non-overlapping 3D tiles aligned with UTM coordinates [S1].
2.  **Image Cropping:** For each tile, crop the input images with overlap to ensure that every pixel relevant to the tile's volume is included in the training data for that specific NeRF [S1].
3.  **Sliding Window Progression:** Implement a 2x2 3D tile progression strategy (sliding window). This ensures that during training, each ray is used at least once in a training batch with all the necessary NeRFs currently loaded in memory [S1].
4.  **Segmented Sampling:** Use a segmented sampler that samples per-bounding box rather than on the entire ray. This guarantees seamless rendering along rays that intersect multiple bounding boxes by handling the integration limits locally within each tile's domain [S1].
5.  **On-the-Fly Ray Computation:** Compute ray coordinates (origin and direction) on-the-fly from the RPCs of each image for the current crop, avoiding the storage of large ray buffers [S1].
6.  **Training:** Train each NeRF independently on its assigned tile data. Because the NeRFs are non-overlapping and persistent, information learned in one tile is not lost when training proceeds to adjacent tiles [S1].

### Formulas

**Scalability Conditions:**
The algorithm must satisfy three conditions [S1]:
1.  Time complexity: $Time = O(N)$, where $N$ is the total number of pixels.
2.  Memory complexity: $Memory = O(1)$, meaning runtime memory never exceeds a fixed budget.
3.  Quality preservation: $A_s(x) \approx A(x)$, where $A_s$ is the tiled algorithm and $A$ is the reference untiled algorithm.

**Input Size Definition:**
$$ N = N_{im} \times A_{ROI} \times SR^2 $$
Where:
*   $N_{im}$: Number of images.
*   $A_{ROI}$: Surface area of the region of interest.
*   $SR$: Spatial Resolution in pixels per meter [S1].

**Rendering Equation:**
The rendering pipeline remains the standard differentiable volume rendering equation [S1]:
$$ \hat{C}(r) = \sum_{k=1}^{N} w_k \mathbf{c}_k, \quad w_k = T_k \alpha_k, \quad T_k = \prod_{j=1}^{k-1} (1 - \alpha_j) $$
where $\alpha_k = 1 - e^{-\sigma_k \delta_k}$ [S1].

### Parameters
*   **Typical Scene Size:** 10km by 10km area [S1].
*   **Spatial Resolution:** 30cm Ground Sample Distance (GSD) [S1].
*   **Image Count:** 20 satellite images [S1].
*   **Hardware Constraint:** Single GPU device [S1].

### Edge Cases and Pitfalls
*   **Catastrophic Forgetting:** Training a single NeRF sequentially on subsets of rays causes the loss of information on previously seen domains. This is mitigated by dividing the scene into non-overlapping NeRFs, ensuring that once a NeRF is trained, its learned information is persistent [S1].
*   **Memory Overflow:** A typical scene (10km x 10km, 30cm GSD, 20 images) involves 200GB of images, ~600GB for the NeRF at image resolution, and 1.4TB for ray vectors (float32). Loading all these simultaneously is impossible on a single GPU [S1]. The framework avoids this by processing tiles and computing rays on-the-fly.
*   **Boundary Artifacts:** Rays intersecting multiple bounding boxes can cause reconstruction errors. The segmented sampler and 2x2 sliding window are specifically designed to prevent these 3D reconstruction errors along tile edges [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def tile_and_slide_training(images, rpcs, roi_area, spatial_resolution, tile_size):
    """
    Main training loop for the Tile and Slide Framework.
    
    Args:
        images: List of image data.
        rpcs: Rational Polynomial Coefficients for each image.
        roi_area: Total area of the scene (A_ROI).
        spatial_resolution: Resolution in pix/m (SR).
        tile_size: Size of each 3D tile in meters.
        
    Returns:
        trained_nerfs: Dictionary mapping tile coordinates to trained NeRF models.
    """
    # 1. Define the tiling grid aligned with UTM coordinates
    # The grid covers the ROI_area. Tile boundaries are calculated based on tile_size.
    tiles = create_utm_grid(roi_area, tile_size)
    
    # 2. Initialize storage for trained NeRFs
    trained_nerfs = {}
    
    # 3. Prepare image crops with overlap
    # Each tile needs specific pixels from the images.
    # Cropping ensures each NeRF gets all necessary data for its domain.
    image_crops = crop_images_with_overlap(images, tiles)
    
    # 4. Sliding Window Progression (2x2 strategy)
    # Process tiles in a 2x2 window to ensure ray continuity across boundaries.
    # This ensures each ray is used at least once with all necessary NeRFs in memory.
    window_step = 2 # 2x2 window size
    for tile_batch in sliding_window(tiles, window_step):
        
        # Load only the NeRFs for the current 2x2 window into GPU memory
        current_nerfs = {}
        for tile in tile_batch:
            if tile not in trained_nerfs:
                # Initialize a new NeRF for this tile
                # Uses multi-resolution hash tables for geometry and global color network
                current_nerfs[tile] = initialize_nerf()
            else:
                current_nerfs[tile] = trained_nerfs[tile]
        
        # 5. Training Loop for the current window
        for tile in tile_batch:
            nerf = current_nerfs[tile]
            crops_for_tile = image_crops[tile]
            
            # Sample rays for this tile
            # Use segmented sampler: sample per-bounding box, not entire ray
            rays = sample_segmented_rays(crops_for_tile, tile.bounding_box)
            
            # 6. On-the-fly Ray Computation
            # Compute ray origins and directions from RPCs
            # Avoids storing large ray buffers
            for ray in rays:
                ray.origin, ray.direction = compute_ray_from_rpcs(ray.camera_id, rpcs)
            
            # Render and Compute Loss
            # Standard volume rendering equation
            rendered_color = render_volume(nerf, ray)
            loss = compute_loss(rendered_color, ray.target_color)
            
            # Backpropagate and Update
            update_nerf(nerf, loss)
            
        # 7. Persist trained NeRFs
        for tile in tile_batch:
            trained_nerfs[tile] = current_nerfs[tile]
            
    return trained_nerfs
```

## Subtopics

### Voxel-based Radiance Fields
Uses structured voxel grids for feature storage. Choose when memory is less constrained and regular grid access is preferred. [Voxel-based Radiance Fields](11470-voxel-based-radiance-fields.md)

### Neural Multi-View Image Reconstruction with Hash Encoding
Uses hash-based feature grids for efficient lookup. Choose for high-resolution details with lower memory overhead than dense voxels. [Neural Multi-View Image Reconstruction with Hash Encoding](13013-neural-multi-view-image-reconstruction-with-hash-encoding.md)

### G-NeLF
Focuses on global consistency in large scenes. Choose when global coherence is prioritized over out-of-core scalability. [G-NeLF](13480-g-nelf.md)

### SCALAR-NeRF
Uses scalar fields for compact representation. Choose for memory-efficient storage of scalar attributes. [SCALAR-NeRF](14267-scalar-nerf.md)

### TriDF
Uses tri-plane representations. Choose for fast rendering and compact storage in specific domains. [TriDF](14761-tridf.md)

## Operating characteristics
*   **Time Complexity:** Linear with respect to input size $N$ ($O(N)$) [S1].
*   **Memory Complexity:** Constant with respect to input size ($O(1)$), bounded by the fixed memory budget of a single GPU [S1].
*   **Performance:** Tested on a benchmark of Very High Spatial Resolution (VHRS) multi-view satellite images [S1].
*   **Hardware:** Operates on a single GPU device [S1].
*   **Strengths:** Eliminates the need to load all images and networks simultaneously; prevents catastrophic forgetting; ensures seamless rendering across tile edges; scales to 10km x 10km scenes [S1].
*   **Limitations:** Requires careful tiling and cropping; dependent on RPCs for ray computation; assumes relatively flat terrain (Z variation ~200m vs 10km XY) [S1].
*   **When to use:** Large-scale Earth observation scenes (satellite, airborne, UAV) where memory constraints prevent training a single global NeRF.
*   **When not to use:** Small scenes where a single NeRF fits in memory; scenes with complex vertical structures where the flat-terrain assumption fails; multi-GPU setups where other methods might be more efficient.

## Background
The Tile and Slide Framework, also known as Snake-NeRF, was introduced to address the memory footprint limitations of state-of-the-art NeRF methods when applied to large-scale 3D Earth Observation [S1]. Previous work on large-scale NeRFs often palliated memory issues by dividing the scene into NeRFs, but typically required multiple GPUs or pre-training [S1]. This framework specifically targets the constraints of a single GPU device, making it suitable for real-world settings where entire sets of multi-view satellite images (e.g., 200GB) and their associated ray data (e.g., 1.4TB) cannot be loaded into memory [S1]. It builds on the principles of [Voxel and Sparse Grid Representations](13738-voxel-and-sparse-grid-representations.md) and incorporates multi-resolution hash tables for geometry encoding, similar to NeRF-XL [S1].

## Sources
[S1] Tile and Slide : A New Framework for Scaling NeRF from Local to Global 3D Earth Observation, https://arxiv.org/abs/2507.01631v2, fetched 2026-10-26
