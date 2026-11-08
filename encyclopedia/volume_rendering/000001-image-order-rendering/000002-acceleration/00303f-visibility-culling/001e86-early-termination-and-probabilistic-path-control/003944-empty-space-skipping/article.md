# Empty-space skipping

Also called: ESS, Empty Region Skipping, Empty space culling

## Overview
Empty-space skipping (ESS) is a volume rendering acceleration technique that minimizes computation by skipping the sampling of transparent or empty regions in a volumetric dataset [S1, S3, S5, unofficial]. It solves the problem of wasted cycles spent evaluating radiance fields or transfer functions in areas that contribute nothing to the final image [S3, S6, unofficial]. Unlike [Opacity-based Early Termination](7920-opacity-based-early-termination.md), which stops a ray after it has accumulated sufficient opacity, ESS prevents the ray from entering empty regions in the first place by using auxiliary data structures (e.g., occupancy grids, octrees, or distance maps) to "jump" over transparent space [S2, S4, S8, unofficial].

## How it works
**Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, volume bounding box, and an empty-space data structure (e.g., occupancy map, distance map, or octree).
**Outputs:** Updated accumulated color $C_{acc}$ and opacity $\alpha_{acc}$, or a termination flag.

**Data Structures:**
1.  **Occupancy/Distance Map:** A 3D texture (or grid) storing binary occupancy or distance-to-nearest-occupied-voxel values [S4, S5, unofficial].
    *   `texOccupancyMap`: Stores binary flags (occupied/empty) [S4, unofficial].
    *   `texDistanceMap`: Stores distance values to the nearest non-empty region [S4, unofficial].
2.  **Coarse Resolution Texture (Min/Max):** A 3D RGB texture, typically $1/8$ the size of the original volume in each dimension, storing disjoint blocks of size $8^3$ [S7, unofficial].
    *   R-component: Minimum scalar value in the block.
    *   G-component: Maximum scalar value in the block.
    *   B-component: Unused or reserved [S7, unofficial].
3.  **Color Table (CT):** A 2D texture indicating for every (min, max) pair if there is at least one non-zero color component after shading [S7, unofficial].

**Parameters:**
*   `sampling_factor`: Controls step density relative to volume dimensions. Inverse is `sampling_factor_inv = 1.0f / sampling_factor` [S4, unofficial].
*   `alpha_thre`: Opacity threshold below which a region is considered empty. Typical value: $10^{-2}$ [S5, unofficial].
*   `early_stop_eps`: Transmittance threshold for early termination. Typical value: $10^{-4}$ [S5, unofficial].
*   `essType`: Integer flag for ESS variant. $1$ = Block ESS; $2$ = Chebyshev ESS [S4, unofficial].
*   `ditheringMode`: Integer flag for ray dithering. $1$ = procedural noise; $2$ = texture sample [S4, unofficial].

**Algorithm Steps:**

1.  **Ray Initialization:**
    *   Calculate step vector and number of steps:
        $$ n_{steps} = \lceil \text{dim}_{max} \cdot \|\mathbf{d}\| \cdot \text{sampling\_factor} \rceil $$
        $$ \mathbf{step\_vector} = \frac{\mathbf{d}}{\|\mathbf{d}\|} \cdot \frac{\|\mathbf{d}\|}{n_{steps} - 1} $$
    *   Initialize distance map lookup parameters:
        $$ \text{dim}_{dist} = \text{textureSize}(\text{texDistanceMap}, 0).xyz $$
        $$ \text{block\_size} = \lceil \frac{\text{volumeSize}}{\text{dim}_{dist}} \rceil $$
        $$ \text{volume\_to\_dist} = \frac{\text{volumeSize}}{\text{block\_size} \cdot \text{dim}_{dist}} $$
        $$ \text{step\_dist\_texel} = \mathbf{step\_vector} \cdot \frac{\text{volumeSize}}{\text{block\_size}} $$
        $$ \text{step\_dist\_texel\_inv} = \frac{1}{\text{step\_dist\_texel}} $$
    *   Determine distance map index based on ray direction signs (0-7):
        $$ \text{idx} = (\mathbf{d}_z < 0 ? 1 : 0) + (\mathbf{d}_y < 0 ? 2 : 0) + (\mathbf{d}_x < 0 ? 4 : 0) $$

2.  **Ray Marching Loop:**
    For each sample index $i$:
    *   **Dithering:** Compute random offset `random` based on `ditheringMode` [S4, unofficial].
    *   **Sampling Position:**
        $$ \mathbf{pos} = \mathbf{o} + (i + \text{random}) \cdot \mathbf{step\_vector} $$
    *   **Map Coordinate:**
        $$ \mathbf{u} = \mathbf{pos} \cdot \text{volume\_to\_dist} \cdot \text{dim}_{dist} $$
        $$ \mathbf{u}_i = \text{clamp}(\lfloor \mathbf{u} \rfloor, 0, \text{dim}_{dist} - 1) $$
    *   **Fetch Empty Space Info:**
        *   If `essType == 1` (Block): `dist = texelFetch(texOccupancyMap, u_i, 0).x`
        *   If `essType == 2` (Chebyshev): `dist = texelFetch(texDistanceMap, u_i, 0).x`
    *   **Calculate Skip Delta:**
        *   Fractional offset: $r = \text{clamp}(\mathbf{u}_i - \mathbf{u}, -1.0, 0.0)$
        *   If `essType == 1`:
            $$ \Delta_{xyz} = (\text{step}(0.0, \text{step\_dist\_texel\_inv}) + r) \cdot \text{step\_dist\_texel\_inv} $$
        *   If `essType == 2`:
            $$ \Delta_{xyz} = (\text{step}(0.0, -\text{step\_dist\_texel\_inv}) + \text{sign}(\text{step\_dist\_texel\_inv}) \cdot \text{dist} + r) \cdot \text{step\_dist\_texel\_inv} $$
        *   Scalar delta: $\Delta = \max(1, \lceil \min(\Delta_{xyz}) \rceil)$
    *   **Handle Empty Space / Backtracking:**
        *   If `dist == 0` (empty space reached but no skip possible or backtracking needed):
            $$ \Delta_{back} = -\lceil \text{sampling\_factor} \rceil $$
            $$ i = \max(i + \Delta_{back}, i_{min}) $$
            Continue to next iteration.
        *   Else:
            $$ i += \Delta $$
            Continue to next iteration.
    *   **Sample Volume (if not skipped):**
        *   Fetch voxel value: $v = \text{getVolumeIntensityValue}(\mathbf{pos})$
        *   Transfer Function: $\mathbf{c} = \text{texture}(\text{texITF}, (v, 0))$
        *   Gamma Correction: $\mathbf{c}.a = \mathbf{c}.a^{\gamma}$
        *   Gradient/Lighting: Compute normal $\mathbf{n}$ from gradient. Compute diffuse/specular lighting.
        *   Composite:
            $$ \mathbf{c}.a = \text{clamp}(1.0 \cdot (1.0 - (1.0 - \mathbf{c}.a)^{\text{sampling\_factor\_inv}}), 0.0, 1.0) $$
            $$ \mathbf{c}.xyz *= \mathbf{c}.a $$
            $$ C_{acc} = C_{acc} + (1.0 - \alpha_{acc}) \cdot \mathbf{c} $$
            $$ \alpha_{acc} = \alpha_{acc} + (1.0 - \alpha_{acc}) \cdot \mathbf{c}.a $$
    *   **Early Termination:**
        *   If $\alpha_{acc} > 0.98$ and `earlyRayTermination` is enabled:
            $$ \alpha_{acc} = 1.0 $$
            Break loop.

3.  **Edge Cases:**
    *   Degenerate rays (infinite, NaN, or length $> \sqrt{3}$) are terminated immediately [S4, unofficial].
    *   When backtracking, the index is clamped to $i_{min}$ to prevent moving backwards beyond the last occupied step [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def empty_space_skipping_raymarch(ray_origin, ray_dir, volume_size, 
                                  tex_occupancy, tex_distance, tex_itf, 
                                  ess_type, sampling_factor, alpha_thre, 
                                  early_stop_eps, dithering_mode):
    # 1. Initialization
    dim_max = max(volume_size)
    ray_length = length(ray_dir)
    
    # Check for degenerate rays
    if ray_length == 0 or isnan(ray_length) or ray_length > sqrt(3):
        return vec3(0.0), 0.0

    n_steps = int(ceil(dim_max * ray_length * sampling_factor))
    step_vector = normalize(ray_dir) * ray_length / (n_steps - 1)
    sampling_factor_inv = 1.0 / sampling_factor
    
    # Distance map params
    dim_dist = texture_size(tex_distance)
    block_size = ceil(volume_size / dim_dist)
    volume_to_dist = volume_size / (block_size * dim_dist)
    step_dist_texel = step_vector * (volume_size / block_size)
    step_dist_texel_inv = 1.0 / step_dist_texel
    
    # Direction index for distance map
    idx = 0
    if ray_dir.z < 0: idx += 1
    if ray_dir.y < 0: idx += 2
    if ray_dir.x < 0: idx += 4
    
    # Dithering
    if dithering_mode == 1:
        random = fract(sin(gl_frag_coord.x * 12.9898 + gl_frag_coord.y * 78.233) * 43758.5453)
    else:
        random = texture(tex_noise, gl_frag_coord.xy / 256).r
        
    acc_color = vec3(0.0)
    acc_alpha = 0.0
    i = 0
    
    while i < n_steps:
        # Sampling position
        sampling_pos = ray_origin + float(i + random) * step_vector
        
        # Map to distance map coords
        pos_dist = sampling_pos * volume_to_dist
        u = pos_dist * dim_dist
        u_i = clamp(floor(u), ivec3(0), dim_dist - 1)
        
        # Fetch empty space info
        if ess_type == 1:
            dist = texel_fetch(tex_occupancy, u_i, 0).x
        else:
            dist = texel_fetch(tex_distance, u_i, 0).x
            
        # Calculate skip delta
        r = clamp(u_i - u, -1.0, 0.0)
        
        if ess_type == 1:
            delta_xyz = (step(0.0, step_dist_texel_inv) + r) * step_dist_texel_inv
        else:
            delta_xyz = (step(0.0, -step_dist_texel_inv) + sign(step_dist_texel_inv) * float(dist) + r) * step_dist_texel_inv
            
        delta = max(1, int(ceil(min(min(delta_xyz.x, delta_xyz.y), delta_xyz.z))))
        
        # Handle empty space / backtracking
        if dist == 0:
            delta_back = -int(ceil(sampling_factor))
            i = max(i + delta_back, 0)
            continue
        else:
            i += delta
            continue
            
        # Sample volume
        voxel_val = get_volume_intensity(sampling_pos, volume_size)
        color = texture(tex_itf, vec2(voxel_val, 0.5)).rgba
        
        # Gamma correction
        color.a = pow(color.a, 2.2) # Typical gamma
        
        # Lighting (simplified)
        normal = -get_gradient(sampling_pos, volume_size, voxel_val)
        light = normalize(vec3(1.0, 1.0, 1.0))
        diffuse = max(0.0, dot(normal, light))
        color.rgb = color.rgb * (0.5 + 0.5 * diffuse)
        
        # Composite
        color.a = clamp(1.0 * (1.0 - pow(1.0 - color.a, sampling_factor_inv)), 0.0, 1.0)
        color.rgb *= color.a
        
        acc_color = acc_color + (1.0 - acc_alpha) * color.rgb
        acc_alpha = acc_alpha + (1.0 - acc_alpha) * color.a
        
        # Early termination
        if acc_alpha > 0.98:
            acc_alpha = 1.0
            break
            
    return acc_color, acc_alpha
```

## Subtopics

### [Ray Marcher with Macrocell Empty Space Skipping](1009-ray-marcher-with-macrocell-empty-space-skipping.md)
An acceleration technique that divides a volume into coarse macrocells, precomputes a boolean occupancy flag for each, and skips ray marching within macrocells that contain only zero-density data. It sets itself apart from sibling methods like [Empty Space Skipping with Orthogonal BSP Tree](1054-empty-space-skipping-with-orthogonal-bsp-tree.md) and [Empty Space Skipping with Octree](1610-empty-space-skipping-with-octree.md) by using a fixed, uniform grid hierarchy rather than a dynamic tree structure, and from [Spatial Skipping (Occupancy Grid)](1614-spatial-skipping-occupancy-grid.md) by operating on precomputed boolean flags derived from transfer-function-dependent densities rather than simple binary occupancy.

### [Empty Space Skipping with Orthogonal BSP Tree](1054-empty-space-skipping-with-orthogonal-bsp-tree.md)
This method accelerates texture-based volume rendering by partitioning the volume into sub-volumes (boxes) based on voxel properties, reorganizing them into an orthogonal BSP tree for visibility sorting, and skipping sub-volumes that are fully transparent according to the transfer function. It extends previous 2D texture approaches to 3D textures, which produce better image quality but require new sorting strategies due to arbitrary slicing plane orientations. Unlike siblings that use uniform grids or octrees, this method uses a one-stage goal-oriented growth process to separate surface voxels from interior voxels, optimizing lit rendering by minimizing unnecessary lighting computations.

### [Empty Space Skipping with Octree](1610-empty-space-skipping-with-octree.md)
Empty Space Skipping with Octree is an acceleration technique for isosurface rendering that uses an octree structure storing conservative minimum and maximum scalar value estimates to skip transparent or irrelevant regions. It solves the problem of expensive spline evaluations in empty space by allowing rays to bypass nodes that do not contain the target isovalue. Unlike standard empty-space skipping which may rely on coarse bricks, this method uses hierarchical min/max ranges to prune nodes.

### [Spatial Skipping (Occupancy Grid)](1614-spatial-skipping-occupancy-grid.md)
Spatial Skipping (Occupancy Grid) is an acceleration technique that improves uniform sampling by identifying empty regions in the volume and skipping them, effectively creating a transmittance estimator based on the ratio of occupied steps. It sets itself apart from its siblings by using a simple, explicit voxel-based occupancy grid with a conservative density threshold to binarize density, whereas alternatives like [Empty Space Skipping with Octree](1610-empty-space-skipping-with-octree.md) use hierarchical structures or [Transfer-Function Independent Empty-Space Skipping](5038-transfer-function-independent-empty-space-skipping.md) optimize for specific transfer functions.

### [Free Space Skipping](4656-free-space-skipping.md)
Free space skipping is an acceleration technique for volume rendering that avoids computing expensive view-dependent color models for points located in transparent or low-density regions of the scene. It operates by querying a hierarchy of density representations—typically a coarse voxel grid followed by a fine voxel grid—and discarding samples where the accumulated alpha (opacity) falls below specific thresholds. This method is distinct from sibling techniques like [Empty Space Skipping with Octree](1610-empty-space-skipping-with-octree.md) or [Spatial Skipping (Occupancy Grid)](1614-spatial-skipping-occupancy-grid.md) because it relies on learned, continuous density fields rather than binary occupancy or static bounding volumes, allowing it to sculpt the free space dynamically during training.

### [Parallel Tree-Construction for Empty-Space Skipping](5037-parallel-tree-construction-for-empty-space-skipping.md)
Parallel tree-construction algorithms enable the run-time re-construction of acceleration data structures for empty-space skipping, primarily to accommodate dynamic changes in the transfer function. This approach distinguishes itself from static acceleration structures by allowing the hierarchy to be rebuilt on the GPU at interactive frame rates, ensuring that the skipping logic remains consistent with the current visual parameters.

### [Transfer-Function Independent Empty-Space Skipping](5038-transfer-function-independent-empty-space-skipping.md)
This family of techniques accelerates volume rendering by storing visibility information in the acceleration structure independent of the transfer function, thereby avoiding reconstruction after transfer-function changes. It decouples spatial culling from opacity evaluation, enabling real-time transfer function adjustments without structural overhead, unlike [Ray Marcher with Macrocell Empty Space Skipping](1009-ray-marcher-with-macrocell-empty-space-skipping.md) or [Empty Space Skipping with Orthogonal BSP Tree](1054-empty-space-skipping-with-orthogonal-bsp-tree.md).

### [SparseLeap](5152-sparseleap.md)
SparseLeap is a hybrid object-order (rasterization) and image-order (ray-casting) method for efficient empty space skipping in large, sparse volumes with fine structures. It decouples empty space skipping data structures from the underlying volume representation, optimizing both the number of samples evaluated and the number of look-ups into the skipping data structure. Unlike standard octree-based skipping which suffers from space fragmentation and high traversal costs around fine structures, SparseLeap rasterizes occupancy geometry to merge consecutive same-class segments, reducing fragmentation and enabling linear list traversal.

### [Homogeneity Acceleration](5754-homogeneity-acceleration.md)
Homogeneity acceleration is an image-order volume rendering technique that extends empty-space skipping by approximating the contribution of uniform, non-empty regions instead of sampling them voxel-by-voxel. It distinguishes itself from its sibling [Empty-space skipping](1002-empty-space-skipping.md) by handling regions that contain data but lack internal variation, thereby reducing computational load where presence acceleration alone would still require per-sample evaluation.

### Brick Classification and Empty Space Skipping
A technique to classify bricks into three categories (skipped/empty, early termination/saturated, or sampled densely) to bypass large sets of work, maintaining state in a shared array on GPU and host memory [S15]. The brick classification metadata is stored as an array consisting of one 32-bit integer per brick of the dataset [S15]. If a brick is identified as empty, the renderer leaps over that space instead of sampling it [S15]. When the transfer function changes, metadata becomes invalid; the system offloads recomputation to a background thread, and until complete, unprocessed bricks are considered 'missing', causing the renderer to request potentially empty bricks which are then filtered by the empty check [S15]. The test platform can process about 7.5 million bricks per second for metadata updates [S15].

### TF-Adaptive Empty-Space Skip
A raymarch acceleration technique that skips volume cells provably contributing zero opacity by checking if the cell's stored max deviation is below a cutoff derived from live contrast/trim uniforms [S16, unofficial]. It declares binding 5 (the max pyramid) in the fragment shader and consults it coarse-to-fine: checking pyramid level 1, then level 0, before paying for a full-res sample [S16, unofficial]. When a skip occurs, it lands back on the fixed step grid defined by $t_{min} + \text{jitter} + k \cdot \text{stepLength}$ to ensure every surviving sample stays bit-identical to the un-accelerated march [S16, unofficial]. The cutoff is calculated as $\text{cutoff} = \text{deadband} - 0.05$, where deadband is derived from `applyContrastWindow`; if cell max deviation < cutoff, the cell is skipped [S16, unofficial]. In dense cosmic-web structures (volume-inside pose), the coarse+fine pyramid check rarely proves a cell empty, adding two textureLoad+slab-test pairs per step with no skip payoff, causing a net regression (+47% volume, +65% scalar-volume per-layer) [S16, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   ESS reduces the number of samples evaluated per ray, effectively reducing the time complexity of the ray-marching loop from $O(N)$ to $O(N_{occupied})$, where $N_{occupied} \ll N$ in sparse volumes [S5, S6, unofficial].
*   Memory overhead depends on the data structure: Occupancy grids add $O(V)$ bits per voxel [S5, unofficial]; Octrees add overhead for tree nodes [S8, unofficial]; Distance maps add $O(V)$ bytes [S4, unofficial].

**Measured Performance:**
*   In GPU-based raycasting, combining early ray termination with empty-space skipping (RC-β) significantly outperforms standard raycasting (RC) and early termination alone (RC-α). For example, on a Head dataset, RC-β achieved 23.4 fps vs 14.4 fps (RC-α) vs 4.8 fps (RC) [S7, unofficial].
*   ESS provides a 2.3x to 5x performance gain in some implementations [S12, unofficial].
*   In neural radiance fields (NeRF), ESS increases rendering speed from 1345 ms to 133 ms (approx. 10x speedup) with minimal storage overhead (5 MB for occupancy volumes of a 300-frame video) [S13].

**Strengths and Limitations:**
*   **Strengths:** Drastically reduces computation in sparse volumes; compatible with various acceleration structures (octrees, grids, BSPs) [S1, S6, S8, unofficial].
*   **Limitations:** Per-sample checking of opacity is expensive; traversing a hierarchy requires specific traversal logic [S1, unofficial]. Empty-space detection is always performed, even if early ray termination is in use, because the intermediate pass modifies z-values and cannot be discarded [S7, unofficial]. In dense scenes, the overhead of querying the empty-space structure may outweigh the benefits [S16, unofficial].

**When to Use:**
*   Use when the volume has large transparent regions [S3, S7, unofficial].
*   Use when the transfer function is static or changes infrequently (for static structures) [S15].
*   Do not use for fully opaque volumes or very dense volumes where empty space is rare [S16, unofficial].

## Background
Empty-space skipping was introduced as a fundamental optimization for volume raycasting to address the computational cost of sampling transparent regions [S1, unofficial][S10, book]. Early methods used bounding boxes or hulls to limit ray traversal [S3, unofficial][S10, book]. The technique evolved to include hierarchical structures like octrees and kd-trees for more precise culling [S6, S8, unofficial]. In the context of neural rendering, ESS has been adapted to prune empty space during the training and inference of NeRFs, using occupancy grids to skip evaluations of the radiance field in empty regions [S5, unofficial][S13]. It is widely used in GPU-based volume raycasting to improve performance and quality [S9].

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
