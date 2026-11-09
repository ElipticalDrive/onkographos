# Empty-space skipping

Also called: ESS, Empty Region Skipping, Empty space culling

## Overview
Empty-space skipping (ESS) is a volume rendering acceleration technique that reduces computational cost by identifying and bypassing regions of the volumetric dataset that are transparent or contain no relevant data [S1, S2, S6, unofficial]. Unlike standard ray marching which samples at fixed intervals regardless of content, ESS uses auxiliary data structures (such as octrees, occupancy grids, or bounding volumes) to "jump" over empty intervals, thereby minimizing the number of expensive density and color evaluations [S3, S4, S8, unofficial]. It sets itself apart from sibling methods like [Early Stop](1790-early-stop.md) (which terminates based on accumulated opacity) and [Russian Roulette](1371-russian-roulette.md) (which probabilistically terminates) by proactively avoiding the evaluation of samples that are known a priori to contribute zero or negligible radiance [S1, S5, unofficial].

## How it works
The method modifies the standard front-to-back ray marching loop by inserting a check against an acceleration structure before evaluating the volume density or transfer function.

### Inputs and Outputs
*   **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, volume dataset $V$, acceleration structure $A$ (encoding empty space), transfer function $TF$.
*   **Output:** Accumulated color $C_{acc}$ and opacity $\alpha_{acc}$ for the pixel.

### Data Structures
The acceleration structure $A$ varies by variant but generally stores spatial occupancy or scalar bounds. Common layouts include:
1.  **Occupancy Grid:** A 3D texture or array where each voxel stores a binary flag (occupied/empty) [S4, S5, unofficial].
2.  **Min-Max Brick/Block:** A coarse 3D texture storing the minimum and maximum scalar values of a block of fine voxels. For example, a block of size $8^3$ might be represented by a single texel with R=Min, G=Max [S7, unofficial].
3.  **Octree:** A hierarchical tree where each node stores min/max bounds of its children [S8, unofficial].

### Algorithm Steps
1.  **Initialization:** Calculate the ray step vector $\mathbf{s}$ and total number of steps $N$ based on volume dimensions and sampling factor [S4, unofficial]. Initialize accumulated color $C_{acc} = (0,0,0,0)$ and transmittance $T = 1.0$.
2.  **Ray Traversal Loop:** Iterate through the ray from near to far clipping plane.
3.  **Empty Space Check:** At the current sampling position $\mathbf{p}$, query the acceleration structure $A$.
    *   If $A$ indicates the region is **empty** (e.g., opacity threshold not met, min/max bounds outside transfer function range, or binary flag is false), skip the evaluation.
    *   Calculate the skip distance $\Delta_{skip}$ based on the size of the empty region or the distance to the next non-empty boundary [S4, S7, unofficial].
    *   Advance the ray position $\mathbf{p} \leftarrow \mathbf{p} + \Delta_{skip} \cdot \mathbf{d}$ and continue to the next iteration [S8, unofficial].
    *   If $A$ indicates the region is **non-empty**, proceed to step 4.
4.  **Volume Evaluation:** Sample the volume density $\sigma$ and color $\mathbf{c}$ at $\mathbf{p}$ using the transfer function [S4, unofficial].
5.  **Compositing:** Update accumulated opacity and color using front-to-back blending:
    $$ \alpha_{sample} = 1 - e^{-\sigma \Delta t} $$
    $$ C_{acc} = C_{acc} + \mathbf{c} \cdot \alpha_{sample} \cdot T $$
    $$ T = T \cdot (1 - \alpha_{sample}) $$
6.  **Termination:** If $T < \epsilon$ (early termination) or the ray exits the volume, break the loop [S5, unofficial].

### Pseudocode
The following pseudocode implements a generic ESS loop using a distance/occupancy map for skipping, based on [S4, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def empty_space_skipping_raymarch(ray_origin, ray_direction, volume_texture, 
                                  occupancy_map, transfer_function, 
                                  volume_size, sampling_factor, ess_type):
    """
    ray_origin: vec3
    ray_direction: vec3 (normalized)
    volume_texture: 3D texture for density/color
    occupancy_map: 3D texture storing skip distances or occupancy flags
    transfer_function: function mapping density to color/alpha
    volume_size: vec3 (dimensions of the volume)
    sampling_factor: float (steps per unit length)
    ess_type: int (1=Block ESS, 2=Chebyshev ESS)
    """
    
    # 1. Initialize Ray Parameters
    ray_length = length(ray_direction) # Or defined by clipping planes
    n_steps = int(ceil(volume_size.max() * sampling_factor))
    step_vector = normalize(ray_direction) * (ray_length / (n_steps - 1))
    
    # Precompute distance map lookup parameters
    dim_dist_map = occupancy_map.size()
    block_size = ceil(volume_size / dim_dist_map)
    vol_to_dist_map = volume_size / (block_size * dim_dist_map)
    step_dist_texel = step_vector * vol_to_dist_map
    step_dist_texel_inv = 1.0 / step_dist_texel
    
    # Dithering for anti-aliasing
    random_val = compute_dithering(ray_origin.xy) 
    
    # 2. Loop
    out_color = vec4(0.0)
    current_t = 0.0
    
    # Determine distance map index based on ray direction signs for caching optimization
    dist_map_idx = 0
    if ray_direction.z < 0: dist_map_idx += 1
    if ray_direction.y < 0: dist_map_idx += 2
    if ray_direction.x < 0: dist_map_idx += 4
    
    for i in range(n_steps):
        # Calculate sampling position
        sampling_pos = ray_origin + (i + random_val) * step_vector
        
        # Check bounds
        if not is_inside_volume(sampling_pos, volume_size):
            break
            
        # Map to distance map coordinates
        pos_dist_map = sampling_pos * vol_to_dist_map
        u = pos_dist_map * dim_dist_map
        u_i = clamp(floor(u), 0, dim_dist_map - 1)
        
        # Fetch occupancy/distance value
        if ess_type == 1: # Block ESS
            dist_val = occupancy_map[u_i].x
        else: # Chebyshev ESS
            dist_val = occupancy_map[u_i].x
            
        # Calculate skip delta
        r = clamp(u_i - u, -1.0, 0.0)
        
        if ess_type == 1:
            i_delta_xyz = (step(0.0, step_dist_texel_inv) + r) * step_dist_texel_inv
        else:
            i_delta_xyz = (step(0.0, -step_dist_texel_inv) + sign(step_dist_texel_inv) * dist_val + r) * step_dist_texel_inv
            
        i_delta = max(1, int(ceil(min(min(i_delta_xyz.x, i_delta_xyz.y), i_delta_xyz.z))))
        
        # Handle empty space detection (dist_val == 0 implies empty in some implementations, 
        # or specific threshold check)
        # Note: Logic varies by implementation. Here we assume if dist_val indicates empty, we skip.
        # For Block ESS, if the block is empty, we jump.
        
        if is_empty_block(dist_val, ess_type):
            # Skip forward
            i += i_delta
            continue
        else:
            # Handle backtrack if necessary (e.g., if we jumped too far or need to re-sample boundary)
            if dist_val == 0 and not is_empty_block(dist_val, ess_type): # Edge case handling
                i_delta_backwards = -int(ceil(sampling_factor))
                i = max(i + i_delta_backwards, 0)
                continue

        # 3. Evaluate Volume (Only if not skipped)
        voxel_value = volume_texture[sampling_pos].r # Density
        color = transfer_function(voxel_value)
        
        # Apply gamma correction if needed
        if gamma_correction:
            color.a = pow(color.a, gamma)
            
        # 4. Composite
        alpha_sample = clamp(1.0 * (1.0 - pow(1.0 - color.a, 1.0/sampling_factor)), 0.0, 1.0)
        color.rgb *= color.a
        
        out_color = out_color + (1.0 - out_color.a) * color
        
        # 5. Early Termination
        if out_color.a > 0.98:
            out_color.a = 1.0
            break
            
    return out_color
```

### Parameters
*   **Sampling Factor:** Controls the base resolution of samples. Typical values depend on volume size and desired quality [S4, unofficial].
*   **Opacity Threshold ($\alpha_{thre}$):** The threshold below which a region is considered empty. Typical value: $10^{-2}$ [S5, unofficial].
*   **Transmittance Threshold ($\epsilon$):** The threshold below which ray marching terminates. Typical value: $10^{-4}$ [S5, unofficial].
*   **ESS Type:** Integer flag selecting the skipping variant (e.g., Block vs. Chebyshev) [S4, unofficial].

### Edge Cases and Pitfalls
*   **Degenerate Rays:** Rays with infinite length, NaN components, or length $>\sqrt{3}$ (in normalized grid coordinates) should be terminated immediately to prevent performance regressions [S4, unofficial].
*   **Backtracking:** When an empty space detection triggers a skip that overshoots a boundary or requires re-evaluation, the loop index must be clamped to prevent moving backwards beyond the last valid occupied step [S4, unofficial].
*   **Transfer Function Dependency:** Standard ESS is often transfer-function dependent. If the transfer function changes, the acceleration structure (e.g., min-max bricks) may become invalid, requiring recomputation [S1, unofficial][S16].
*   **Intermediate Pass Overhead:** In some GPU implementations, an intermediate pass is required to check for empty space to update depth buffers (z-values). This pass cannot be discarded even if early ray termination is used, because the alpha channel is reserved for termination flags [S7, unofficial].

## Subtopics

### [Ray Marcher with Macrocell Empty Space Skipping](1009-ray-marcher-with-macrocell-empty-space-skipping.md)
Uses fixed uniform macrocells with precomputed boolean occupancy flags. Chosen for simplicity and static scenes.

### [Empty Space Skipping with Orthogonal BSP Tree](1054-empty-space-skipping-with-orthogonal-bsp-tree.md)
Uses an orthogonal BSP tree for visibility sorting and skipping. Chosen for lit rendering to minimize lighting computations.

### [Empty Space Skipping with Octree](1610-empty-space-skipping-with-octree.md)
Uses an octree with min/max scalar values. Chosen for interactive transfer function changes as it embeds structure in the volume container.

### [Spatial Skipping (Occupancy Grid)](1614-spatial-skipping-occupancy-grid.md)
Uses a uniform occupancy grid. Chosen for simple implementation and transmittance estimation.

### [Free Space Skipping](4656-free-space-skipping.md)
Uses learned continuous density fields. Chosen for dynamic training-based acceleration.

### [Parallel Tree-Construction for Empty-Space Skipping](5037-parallel-tree-construction-for-empty-space-skipping.md)
Rebuilds acceleration structures on GPU. Chosen for dynamic transfer functions.

### [Transfer-Function Independent Empty-Space Skipping](5038-transfer-function-independent-empty-space-skipping.md)
Stores visibility independent of TF. Chosen to avoid reconstruction overhead.

### [Homogeneity Acceleration](5754-homogeneity-acceleration.md)
Approximates uniform non-empty regions. Chosen to reduce load where presence acceleration alone is insufficient.

### Brick Classification and Empty Space Skipping
Classifies bricks into skipped, early-termination, or sampled categories using a shared metadata array. Chosen for GPU-based ray-guided rendering where metadata updates can be offloaded to background threads. Pitfall: Metadata invalidation during TF changes causes temporary "missing" brick requests [S16].

### TF-Adaptive Empty-Space Skip
Skips cells if max deviation is below a cutoff derived from live contrast/trim uniforms. Chosen for precise zero-opacity skipping. Pitfall: In dense structures, the overhead of pyramid checks can cause net regression [S17, unofficial].

### Empty Space Skipping via Acceleration Structure
Uses a two-level volumetric hierarchy (e.g., 16x16x16 structure for 32x32x32 blocks). Chosen for GPU-extended optimization providing 1.5-2x speedup [S18, unofficial].

## Operating characteristics
*   **Time Complexity:** Reduces the number of volume samples from $O(N)$ to $O(N_{occupied})$, where $N_{occupied} \ll N$ in sparse volumes. The cost of querying the acceleration structure is typically $O(1)$ for grids/textures or $O(\log N)$ for trees [S6, unofficial].
*   **Memory Complexity:** Requires additional memory for the acceleration structure. For example, an occupancy grid adds minimal overhead, while min-max bricks add 2-4 bytes per block [S7, unofficial][S14]. Storage for a 300-frame video occupancy volume is ~8 MB [S14].
*   **Performance:** Can significantly improve frame rates. For example, RC-β (with ESS) achieved 23.4 fps vs 4.8 fps for RC (no acceleration) on a head dataset [S7, unofficial]. In NeRF inference, ESS increased speed from 1345 ms to 133 ms [S14].
*   **Strengths:** Highly effective for sparse volumes; reduces memory bandwidth pressure by skipping texture fetches.
*   **Limitations:** Overhead of structure traversal/querying; transfer-function dependency in many variants requires structure rebuilds; less effective for dense, fully opaque volumes [S6, S12, unofficial].

## Background
Empty-space skipping originated as a fundamental optimization in direct volume rendering (DVR) to address the high computational cost of evaluating hundreds of samples per ray [S6, unofficial]. Early methods used hierarchical representations like octrees to traverse coarse regions with larger steps [S10, book]. The technique has been widely adopted in GPU-based raycasting, often combined with frame-to-frame coherence to estimate initial sampling points [S9]. It remains a core component in modern neural rendering accelerators (e.g., NerfAcc) to prune empty and occluded regions during training and inference [S5, unofficial].

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
[S12] PDF Volume Rendering - Isenberg, https://tobias.isenberg.cc/graphics/uploads/Class/07-volume-rendering.pdf, fetched 2026-10-28 (unofficial)
[S13] PDF GPU Isosurface Raycasting of Volume Datasets Based On Box-Splines, https://minho-kim-uos.github.io/papers/birs13.pdf, fetched 2026-10-29 (unofficial)
[S14] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
[S15] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S16] An Analysis of Scalable GPU-Based Ray-Guided Volume Rendering, https://pmc.ncbi.nlm.nih.gov/articles/PMC4260263/, fetched 2026-10-26
[S17] feat(volumes): raymarch acceleration — GPU pyramids, TF-adaptive skipping, cone LOD, https://github.com/rulkens/skymap/pull/556, fetched 2026-10-28 (unofficial)
[S18] GENERAL IMPLEMENTATION ASPECTS OF THE GPU- ..., http://sv-journal.org/2011-1/02/, fetched 2026-10-30 (unofficial)
