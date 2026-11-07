# Empty-space skipping

Also called: ESS, Empty Region Skipping, Empty space culling

## Overview
Empty-space skipping (ESS) is a volume rendering acceleration technique that minimizes computation by skipping the sampling of empty space in a volumetric dataset [S3, S4, S5, unofficial]. It solves the problem of wasted cycles spent evaluating transparent or low-density regions, which constitute the main bottleneck in ray marching [S5, unofficial]. It sets itself apart from [Opacity-based Early Termination](7920-opacity-based-early-termination.md) (which stops the ray after it has entered the volume) by preventing the ray from entering or traversing empty regions in the first place, often using auxiliary data structures like occupancy grids, distance maps, or octrees [S2, S4, S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, volume dataset, and an empty-space data structure (e.g., occupancy map, distance map, octree, or coarse min/max texture) [S4, S7, S8, unofficial].
*   **Output:** Accumulated color and opacity for the pixel, with significantly fewer volume samples evaluated [S5, unofficial].

**Data Structures:**
The specific layout depends on the variant, but common structures include:
1.  **Occupancy/Distance Map:** A 3D texture (e.g., `texOccupancyMap`, `texDistanceMap`) storing precomputed values. Accessed via `texelFetch` [S4, unofficial].
2.  **Coarse Min/Max Texture:** A 3D RGB texture, typically $1/8$ the size of the original volume in each dimension, storing disjoint blocks (e.g., $8^3$). The R-component stores the minimum scalar value, and the G-component stores the maximum scalar value in the block [S7, unofficial].
3.  **Occupancy Grid:** A binary grid where a voxel is marked occupied if its density/opacity exceeds a threshold (e.g., $\alpha_i < 10^{-2}$) [S5, unofficial].
4.  **Brick Classification Array:** An array of 32-bit integers, one per brick, storing state (skipped, saturated, sampled) [S15].

**Algorithm Steps (General ESS with Distance/Occupancy Map):**
The following pseudocode implements a generic ESS loop using a distance map or occupancy map, combining logic from [S4, unofficial] and [S5, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_with_ess(ray_origin, ray_dir, volume_texture, 
                       distance_map_texture, dim_volume, dim_dist_map,
                       sampling_factor, alpha_thre=1e-2, early_stop_eps=1e-4,
                       ess_type=1):
    """
    ess_type: 1 = Block ESS, 2 = Chebyshev ESS [S4, unofficial]
    """
    # 1. Initialize ray parameters
    ray_length = length(ray_dir) # Assuming ray_dir is normalized or length is known
    if ray_length == 0 or isnan(ray_length) or ray_length > sqrt(3):
        return 0.0, 0.0 # Handle degenerate rays [S4, unofficial]

    n_steps = int(ceil(dim_volume.max() * ray_length * sampling_factor))
    step_vector = normalize(ray_dir) * (ray_length / (n_steps - 1))
    
    # Precompute distance map lookup parameters [S4, unofficial]
    block_size = ceil(dim_volume / dim_dist_map)
    volume_to_dist_map = dim_volume / (block_size * dim_dist_map)
    step_dist_texel = step_vector * volume_to_dist_map
    step_dist_texel_inv = 1.0 / step_dist_texel
    
    # Determine distance map index based on ray direction signs [S4, unofficial]
    # Note: This assumes a specific coordinate system; adjust if necessary.
    distance_map_idx = 0 
    # (Implementation detail: usually handled by pre-computed maps or simple sign checks)

    # 2. Initialize accumulation
    acc_color = vec3(0.0)
    acc_alpha = 0.0
    i = 0
    
    # 3. Ray Marching Loop
    while i < n_steps:
        # Calculate sampling position with optional dithering [S4, unofficial]
        # random = ... (procedural or texture noise)
        sampling_pos = ray_origin + (i + 0.5) * step_vector
        
        # Map sampling position to distance map coordinates [S4, unofficial]
        pos_dist_map = volume_to_dist_map * sampling_pos
        u = pos_dist_map * dim_dist_map
        u_i = clamp(floor(u), ivec3(0), dim_dist_map - 1)
        
        # Fetch distance/occupancy value [S4, unofficial]
        if ess_type == 1: # Block ESS
            dist = texelFetch(distance_map_texture, u_i, 0).x
        else: # Chebyshev ESS
            dist = texelFetch(distance_map_texture, u_i, 0).x
            
        # Calculate step delta to skip empty space [S4, unofficial]
        r = clamp(u_i - u, -1.0, 0.0)
        
        if ess_type == 1:
            i_delta_xyz = (step(0.0, step_dist_texel_inv) + r) * step_dist_texel_inv
        else:
            i_delta_xyz = (step(0.0, -step_dist_texel_inv) + sign(step_dist_texel_inv) * float(dist) + r) * step_dist_texel_inv
            
        i_delta = max(1, int(ceil(min(min(i_delta_xyz.x, i_delta_xyz.y), i_delta_xyz.z))))
        
        # Handle backtracking if empty space detected but no skip possible [S4, unofficial]
        if dist == 0 and i_delta <= 1:
            i_delta_backwards = -int(ceil(sampling_factor))
            i = max(i + i_delta_backwards, 0)
            continue
            
        # Advance index
        i += i_delta
        
        # 4. Sample Volume (if not skipped)
        # Check if we are still within bounds and not terminated
        if acc_alpha > 1.0 - early_stop_eps:
            break
            
        # Evaluate volume intensity
        voxel_value = get_volume_intensity(sampling_pos, volume_texture)
        
        # Transfer function lookup
        color = transfer_function(voxel_value)
        
        # Apply gamma correction to alpha if needed [S4, unofficial]
        if use_gamma:
            color.a = pow(color.a, gamma)
            
        # Compute lighting (optional, depends on renderer) [S4, unofficial]
        # gradient = get_gradient(sampling_pos, volume_texture)
        # normal = -gradient.xyz
        # ... compute diffuse/specular ...
        
        # Composite using front-to-back blending [S4, unofficial]
        # Note: The formula in the parent article is standard. 
        # Here we adapt the specific implementation from [S4, unofficial] which uses sampling_factor_inv
        alpha_sample = clamp(1.0 * (1.0 - pow(1.0 - color.a, 1.0/sampling_factor)), 0.0, 1.0)
        color.rgb *= alpha_sample
        
        acc_color = acc_color + (1.0 - acc_alpha) * color.rgb
        acc_alpha = acc_alpha + alpha_sample * (1.0 - acc_alpha)
        
        # Early Ray Termination [S4, unofficial]
        if acc_alpha > 0.98:
            acc_alpha = 1.0
            break

    return acc_color, acc_alpha
```

**Parameters:**
*   `sampling_factor`: Controls the base number of steps relative to volume dimensions. Inverse is often stored as `sampling_factor_inv` [S4, unofficial].
*   `alpha_thre`: Opacity threshold below which a region is considered empty. Typical value: $10^{-2}$ [S5, unofficial].
*   `early_stop_eps`: Transmittance threshold for early termination. Typical value: $10^{-4}$ [S5, unofficial].
*   `essType`: Integer flag selecting the ESS variant. 1 = Block Empty Space Skipping; 2 = Chebyshev Empty Space Skipping [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Rays:** Rays with infinite length, NaN values, or length $> \sqrt{3}$ must be terminated immediately to prevent performance regressions [S4, unofficial].
*   **Backtracking:** When backtracking due to empty space detection (e.g., `dist==0`), the index must be clamped to the minimum valid index to prevent moving backwards beyond the last occupied step [S4, unofficial].
*   **Transfer Function Dependency:** Standard ESS depends on the transfer function. If the transfer function changes, the empty-space data structure (e.g., occupancy grid or min/max bounds) may become invalid and require recomputation [S1, unofficial][S15].
*   **Intermediate Pass Overhead:** In some implementations (e.g., using coarse min/max textures), empty-space detection is performed in an intermediate pass. This pass cannot be discarded even if early ray termination is active, because it modifies z-values for depth testing. The alpha channel cannot be used for this flag as it indicates early termination [S7, unofficial].

## Subtopics

### [Ray Marcher with Macrocell Empty Space Skipping](1009-ray-marcher-with-macrocell-empty-space-skipping.md)
Uses a fixed, uniform grid hierarchy (macrocells) with precomputed boolean occupancy flags. Sets itself apart by using a static grid rather than a dynamic tree, and operates on transfer-function-dependent densities rather than simple binary occupancy.

### [Empty Space Skipping with Orthogonal BSP Tree](1054-empty-space-skipping-with-orthogonal-bsp-tree.md)
Partitions the volume into sub-volumes based on voxel properties into an orthogonal BSP tree for visibility sorting. Sets itself apart by using a one-stage goal-oriented growth process to separate surface from interior voxels, optimizing lit rendering.

### [Empty Space Skipping with Octree](1610-empty-space-skipping-with-octree.md)
Uses an octree structure storing conservative minimum and maximum scalar value estimates. Sets itself apart by using hierarchical min/max ranges to prune nodes that do not contain the target isovalue, solving the problem of expensive spline evaluations in empty space.

### [Spatial Skipping (Occupancy Grid)](1614-spatial-skipping-occupancy-grid.md)
Uses a simple, explicit voxel-based occupancy grid with a conservative density threshold to binarize density. Sets itself apart by creating a transmittance estimator based on the ratio of occupied steps, unlike hierarchical structures.

### [Free Space Skipping](4656-free-space-skipping.md)
Relies on learned, continuous density fields (e.g., from neural radiance fields) rather than binary occupancy or static bounding volumes. Sets itself apart by sculpting free space dynamically during training.

### [Parallel Tree-Construction for Empty-Space Skipping](5037-parallel-tree-construction-for-empty-space-skipping.md)
Enables run-time re-construction of acceleration data structures (like octrees) on the GPU to accommodate dynamic changes in the transfer function. Sets itself apart by allowing interactive frame-rate updates of the hierarchy.

### [Transfer-Function Independent Empty-Space Skipping](5038-transfer-function-independent-empty-space-skipping.md)
Stores visibility information in the acceleration structure independent of the transfer function. Sets itself apart by decoupling spatial culling from opacity evaluation, enabling real-time TF adjustments without structural overhead.

### [Homogeneity Acceleration](5754-homogeneity-acceleration.md)
Extends empty-space skipping by approximating the contribution of uniform, non-empty regions instead of sampling them voxel-by-voxel. Sets itself apart by handling regions that contain data but lack internal variation, reducing load where presence acceleration alone would still require per-sample evaluation.

### Brick Classification and Empty Space Skipping
A technique to classify bricks into three categories (skipped/empty, early termination/saturated, or sampled densely) to bypass large sets of work, maintaining state in a shared array on GPU and host memory [S15]. It maintains a table of brick state shared on both GPU and host memories. If a brick is identified as empty, the renderer leaps over that space instead of sampling it [S15]. The metadata is stored as an array of one 32-bit integer per brick [S15]. A pitfall is that when the transfer function changes, metadata becomes invalid; the system offloads recomputation to a background thread, and until complete, unprocessed bricks are considered 'missing', causing the renderer to request potentially empty bricks which are then filtered [S15]. The test platform can process about 7.5 million bricks per second for metadata updates [S15].

### SparseLeap
A method that optimizes both the number of samples to evaluate and the number of look-ups into the empty-space skipping data structure [S6, unofficial]. An advantage of SparseLeap over other empty-space skipping techniques is that it is agnostic of the data structure used for managing volume data (e.g., page-table hierarchies or octrees) and can be used in conjunction with them [S6, unofficial].

### TF-Adaptive Empty-Space Skip
A raymarch acceleration technique that skips volume cells provably contributing zero opacity by checking if the cell's stored max deviation is below a cutoff derived from live contrast/trim uniforms [S16, unofficial]. It declares binding 5 (the max pyramid) in the fragment shader and consults it coarse-to-fine: checking pyramid level 1, then level 0, before paying for a full-res sample [S16, unofficial]. When a skip occurs, it lands back on the fixed step grid defined by $t_{min} + jitter + k \cdot stepLength$ to ensure every surviving sample stays bit-identical to the un-accelerated march [S16, unofficial]. The cutoff is calculated as $cutoff = (deadband - 0.05)$, where $deadband$ is derived from $applyContrastWindow$. If cell max deviation $< cutoff$, the cell is skipped [S16, unofficial]. A pitfall is that in dense cosmic-web structures (volume-inside pose), the coarse+fine pyramid check rarely proves a cell empty, adding two textureLoad+slab-test pairs per step with no skip payoff, causing a net regression (+47% volume, +65% scalar-volume per-layer) [S16, unofficial].

## Operating characteristics
*   **Time Complexity:** Reduces the number of samples per ray from $O(N)$ to $O(N_{occupied})$, where $N_{occupied} \ll N$ in sparse volumes. The cost of querying the acceleration structure (e.g., octree traversal or texture lookup) is added per step or per block [S6, unofficial].
*   **Memory Complexity:** Requires additional memory for the empty-space data structure (e.g., occupancy grid, distance map, or octree). Storage for occupancy volumes of a 300-frame video is about 8 MB [S13].
*   **Measured Performance:**
    *   In GPU-based raycasting, combining early ray termination with empty-space skipping (RC-$\beta$) significantly outperforms standard raycasting (RC) and early termination alone (RC-$\alpha$). For a Head dataset, RC-$\beta$ achieved 23.4 fps vs 14.4 fps (RC-$\alpha$) vs 4.8 fps (RC) [S7, unofficial].
    *   ESS provides a 2.3x to 5x performance gain in some implementations [S12, unofficial].
    *   In neural rendering, ESS increases rendering speed of a single shared MLP model from 1345 ms to 133 ms, improving speed by 6 times while adding only 5 MB to storage cost [S13].
*   **Strengths:** Significantly reduces computation in sparse volumes; avoids evaluating expensive lighting or transfer functions in transparent regions [S5, S7, unofficial].
*   **Limitations:** Performance gain depends on the sparsity of the volume; in dense volumes, the overhead of querying the acceleration structure may outweigh the benefits [S6, unofficial]. Requires precomputation or online maintenance of the empty-space structure, which can be costly if the transfer function changes frequently [S1, unofficial][S15].
*   **When to use:** Use when the volume has large transparent regions and the transfer function is static or changes infrequently. Use when memory bandwidth is a bottleneck and skipping samples reduces texture fetches [S5, unofficial].
*   **When not to use:** Avoid when the volume is dense (e.g., smoke, fog) where most samples contribute to the image, or when the transfer function changes every frame and the overhead of rebuilding the acceleration structure is too high [S1, unofficial][S15].

## Background
Empty-space skipping was introduced as a fundamental optimization for volume raycasting to address the computational waste of sampling "in the empty" [S3, unofficial]. Early work by Levoy [S10, book] and Zuiderveld et al. [S10, book] proposed using hierarchical representations or distance fields to adapt step sizes. The technique has been widely adopted in GPU-based rendering, with implementations by Klein et al. [S9] and Lee and Shin [S9] exploiting spatial coherence and frame-to-frame coherence. Modern implementations include block-based skipping [S4, unofficial], octree-based culling [S7, S8, unofficial], and neural rendering adaptations like NerfAcc [S5, unofficial] and SparseLeap [S6, unofficial].

## Sources
[S1] GPU-Based Volume Ray-Casting with Advanced Illumination, https://doi.org/10.2312/egt.20091069, fetched 2026-10-26 (unofficial)
[S2] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
[S3] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S4] CalamityMichL/DVR_ESS_PDM, https://github.com/CalamityMichL/DVR_ESS_PDM, fetched 2026-10-26 (unofficial)
[S5] NerfAcc: A General NeRF Acceleration Toolbox, https://doi.org/10.48550/arxiv.2210.04847, fetched 2026-10-26 (unofficial)
[S6] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S7] [PDF] Acceleration Techniques for GPU-based Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Acceleration_Techniques_for_GPU-based_Volume_Rendering/vis03-rc.pdf, fetched 2026-10-26 (unofficial)
[S8] Volume Ray Casting Techniques and Applications using ..., https://halogenica.net/bin/graphics/volume-rendering-using-gpgpu/Thesis.pdf, fetched 2026-10-27 (unofficial)
[S9] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S10] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27 (book)
[S11] PDF Direct Volume Rendering - KTH, https://www.kth.se/social/files/565e35dff27654457fb84363/08_VolumeRendering.pdf, fetched 2026-10-27 (unofficial)
[S12] PDF GPU Isosurface Raycasting of Volume Datasets Based On Box-Splines, https://minho-kim-uos.github.io/papers/birs13.pdf, fetched 2026-10-29 (unofficial)
[S13] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
[S14] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S15] An Analysis of Scalable GPU-Based Ray-Guided Volume Rendering, https://pmc.ncbi.nlm.nih.gov/articles/PMC4260263/, fetched 2026-10-26
[S16] feat(volumes): raymarch acceleration — GPU pyramids, TF-adaptive skipping, cone LOD, https://github.com/rulkens/skymap/pull/556, fetched 2026-10-28 (unofficial)
