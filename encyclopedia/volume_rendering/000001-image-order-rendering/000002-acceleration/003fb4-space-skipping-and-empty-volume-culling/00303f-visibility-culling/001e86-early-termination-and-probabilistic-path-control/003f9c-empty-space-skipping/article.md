# Empty-space skipping

Also called: ESS, Empty Region Skipping, Empty space culling

## Overview
Empty-space skipping (ESS) is a volume rendering acceleration technique that reduces computational cost by identifying and bypassing regions of the volumetric dataset that are transparent or contain no relevant data [S1, S2, S6, unofficial]. It solves the problem of wasted computation when ray marching through empty space, where standard fixed-step sampling yields no visual contribution [S3, S5, unofficial]. Unlike [Early Stop](1790-early-stop.md), which terminates rays based on accumulated opacity, ESS proactively skips intervals of empty space using auxiliary data structures (e.g., octrees, distance maps, or occupancy grids) [S2, S4, S8, unofficial]. It differs from [Bounding Volume Culling](1239-bounding-volume-culling.md) by allowing skipping *within* the volume bounds rather than just before entry, and from [Russian Roulette](1371-russian-roulette.md) by deterministically skipping empty regions rather than probabilistically terminating rays [S5, S6, unofficial].

## How it works
**Inputs:**
*   Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized).
*   Volume dataset $V$ (3D texture or grid).
*   Transfer function $TF$ (mapping scalar values to color/opacity).
*   Empty-space data structure $D$ (e.g., occupancy map, distance map, octree, or coarse resolution texture) [S2, S4, S7, S8, unofficial].
*   Parameters: `sampling_factor` (steps per unit length), `alpha_thre` (opacity threshold for empty), `early_stop_eps` (transmittance threshold for termination) [S4, S5, unofficial].

**Outputs:**
*   Accumulated color $C_{acc}$ and opacity $\alpha_{acc}$ for the pixel.

**Data Structures:**
1.  **Occupancy/Distance Map ($D$):** A 3D texture or grid coarser than $V$.
    *   *Block ESS (Type 1):* Stores binary occupancy or block size. Accessed via `texelFetch` [S4, unofficial].
    *   *Chebyshev ESS (Type 2):* Stores distance to nearest non-empty voxel. Accessed via `texelFetch` [S4, unofficial].
    *   *Coarse Resolution Texture (Type 3):* 3D RGB texture, 1/8 size of $V$ in each dimension. R-channel: min scalar value in block; G-channel: max scalar value in block [S7, unofficial].
    *   *CT Texture:* 2D texture mapping (min, max) pairs to boolean "non-empty" status, updated when transfer function changes [S7, unofficial].
2.  **Volume Texture ($V$):** High-resolution 3D texture storing scalar density values [S4, unofficial].
3.  **Transfer Function Texture ($TF$):** 1D or 2D texture mapping density to RGBA [S4, unofficial].

**Algorithm Steps:**

1.  **Ray Validation:** Check for degenerate rays (infinite, NaN, or length $> \sqrt{3}$). Terminate immediately if degenerate [S4, unofficial].
2.  **Initialization:**
    *   Calculate total steps: $n_{steps} = \lceil \text{dim}_{max} \cdot \|\mathbf{d}\| \cdot \text{sampling\_factor} \rceil$ [S4, unofficial].
    *   Calculate step vector: $\mathbf{s} = \frac{\mathbf{d} \cdot \|\mathbf{d}\|}{n_{steps} - 1}$ [S4, unofficial].
    *   Initialize distance map lookup parameters:
        *   $\text{dim}_{map} = \text{textureSize}(D, 0).xyz$
        *   $\text{block\_size} = \lceil \text{volumeSize} / \text{dim}_{map} \rceil$
        *   $\mathbf{v}_{to\_map} = \text{volumeSize} / (\text{block\_size} \cdot \text{dim}_{map})$
        *   $\mathbf{s}_{tex} = \mathbf{s} \cdot \text{volumeSize} / \text{block\_size}$
        *   $\mathbf{s}_{tex\_inv} = 1.0 / \mathbf{s}_{tex}$ [S4, unofficial]
    *   Initialize $C_{acc} = (0,0,0,0)$, $\alpha_{acc} = 0$, loop index $i = 0$ [S4, unofficial].
3.  **Ray Marching Loop:** While $i < n_{steps}$ and $\alpha_{acc} < 1.0$:
    *   **Dithering (Optional):** Compute random offset $r$ to reduce banding.
        *   If `ditheringMode` == 1: $r = \text{fract}(\sin(\mathbf{gl\_FragCoord}.x \cdot 12.9898 + \mathbf{gl\_FragCoord}.y \cdot 78.233) \cdot 43758.5453)$ [S4, unofficial].
        *   If `ditheringMode` == 2: $r = \text{texture}(\text{texNoise}, \mathbf{gl\_FragCoord}.xy / 256).r$ [S4, unofficial].
    *   **Sampling Position:** $\mathbf{p}_{sample} = \mathbf{o} + (i + r) \cdot \mathbf{s}$ [S4, unofficial].
    *   **Empty-Space Query:**
        *   Map $\mathbf{p}_{sample}$ to map coordinates: $\mathbf{u} = \mathbf{p}_{sample} \cdot \mathbf{v}_{to\_map} \cdot \text{dim}_{map}$.
        *   Integer index $\mathbf{u}_i = \text{clamp}(\lfloor \mathbf{u} \rfloor, 0, \text{dim}_{map}-1)$ [S4, unofficial].
        *   Fetch value $val$ from $D$ at $\mathbf{u}_i$ [S4, unofficial].
        *   **Calculate Skip Delta ($\Delta i$):**
            *   Fractional offset $r_{frac} = \text{clamp}(\mathbf{u}_i - \mathbf{u}, -1.0, 0.0)$ [S4, unofficial].
            *   If `essType` == 1 (Block ESS):
                *   $\Delta \mathbf{i}_{xyz} = (\text{step}(0.0, \mathbf{s}_{tex\_inv}) + r_{frac}) \cdot \mathbf{s}_{tex\_inv}$ [S4, unofficial].
            *   If `essType` == 2 (Chebyshev ESS):
                *   $\Delta \mathbf{i}_{xyz} = (\text{step}(0.0, -\mathbf{s}_{tex\_inv}) + \text{sign}(\mathbf{s}_{tex\_inv}) \cdot val + r_{frac}) \cdot \mathbf{s}_{tex\_inv}$ [S4, unofficial].
            *   $\Delta i = \max(1, \lceil \min(\Delta \mathbf{i}_{xyz}.x, \Delta \mathbf{i}_{xyz}.y, \Delta \mathbf{i}_{xyz}.z) \rceil)$ [S4, unofficial].
            *   **Backtracking:** If $val == 0$ (empty space reached but no skip possible or backtracking needed):
                *   $\Delta i_{back} = -\lceil \text{sampling\_factor} \rceil$
                *   $i = \max(i + \Delta i_{back}, 0)$
                *   Continue to next iteration (skip sampling) [S4, unofficial].
            *   **Advance:** $i \leftarrow i + \Delta i$ [S4, unofficial].
            *   **Re-evaluate Position:** If $i$ was advanced by $\Delta i > 1$, re-calculate $\mathbf{p}_{sample}$ for the new $i$ to ensure correct sampling position, or skip sampling if the entire interval was empty. *Note: Standard ESS implementations often sample at the boundary of the skipped region or skip entirely if the region is fully empty. The source [S4, unofficial] implies skipping the loop body if empty, but sampling if non-empty.*
    *   **Volume Sampling (if not skipped):**
        *   Fetch density: $v = \text{texture}(V, \mathbf{p}_{sample})$ [S4, unofficial].
        *   Transfer Function: $\mathbf{c} = \text{texture}(TF, v)$ [S4, unofficial].
        *   Gamma Correction: If enabled, $\mathbf{c}.a = \mathbf{c}.a^{\gamma}$ [S4, unofficial].
        *   Gradient/Lighting (Optional): Compute normal $\mathbf{n}$ from gradient of $V$ at $\mathbf{p}_{sample}$. Compute diffuse/specular lighting [S4, unofficial].
        *   Composite:
            *   $\alpha_{sample} = \mathbf{c}.a \cdot \text{sampling\_factor\_inv}$ (approximation) or derived from density $\sigma$ and step size $\delta$: $\alpha_{sample} = 1 - e^{-\sigma \delta}$ [S5, unofficial].
            *   $C_{acc} \leftarrow C_{acc} + \mathbf{c} \cdot \alpha_{sample} \cdot (1 - \alpha_{acc})$
            *   $\alpha_{acc} \leftarrow \alpha_{acc} + \alpha_{sample} \cdot (1 - \alpha_{acc})$ [S4, S5, unofficial].
    *   **Early Termination:** If $\alpha_{acc} > 0.98$ and `earlyRayTermination` is true, set $\alpha_{acc} = 1.0$ and break [S4, unofficial].

**Parameters:**
*   `essType`: 1 (Block), 2 (Chebyshev) [S4, unofficial].
*   `alpha_thre`: Opacity threshold for empty region. Typical: $10^{-2}$ [S5, unofficial].
*   `early_stop_eps`: Transmittance threshold for termination. Typical: $10^{-4}$ [S5, unofficial].
*   `sampling_factor`: Steps per unit length [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Rays:** Must be checked to prevent infinite loops or performance regressions [S4, unofficial].
*   **Backtracking:** When empty space is detected at the current sample, the ray may need to backtrack to the start of the empty block to ensure no boundary samples are missed. Index must be clamped to $\ge 0$ [S4, unofficial].
*   **Transfer Function Changes:** In methods using min/max bounds (e.g., [S7, unofficial]), the CT texture must be updated on the CPU when the transfer function changes, otherwise empty-space detection becomes invalid [S7, unofficial][S15].
*   **Intermediate Pass Overhead:** In some GPU implementations, an intermediate pass is required to check for empty space even if early ray termination is active, because the alpha channel is used for termination flags. This pass cannot be discarded [S7, unofficial].

## Subtopics

### [Ray Marcher with Macrocell Empty Space Skipping](1009-ray-marcher-with-macrocell-empty-space-skipping.md)
Uses a fixed, uniform grid hierarchy (macrocells) with precomputed boolean occupancy flags to skip empty regions. Chosen when a static, uniform acceleration structure is preferred over dynamic trees.

### [Empty Space Skipping with Orthogonal BSP Tree](1054-empty-space-skipping-with-orthogonal-bsp-tree.md)
Partitions volume into sub-volumes using an orthogonal BSP tree, sorting them for visibility and skipping transparent sub-volumes. Chosen for lit rendering where minimizing lighting computations on interior voxels is critical.

### [Empty Space Skipping with Octree](1610-empty-space-skipping-with-octree.md)
Uses an octree storing min/max scalar values to skip regions below an isovalue/opacity threshold. Chosen for interactive transfer function changes, as the hierarchy can be re-evaluated quickly.

### [Spatial Skipping (Occupancy Grid)](1614-spatial-skipping-occupancy-grid.md)
Uses a uniform occupancy grid to skip empty regions, creating a transmittance estimator. Chosen for simple, uniform acceleration without hierarchical overhead.

### [Free Space Skipping](4656-free-space-skipping.md)
Uses learned, continuous density fields (coarse/fine voxel grids) to skip low-density regions. Chosen for neural radiance fields (NeRFs) where dynamic sculpting of free space is possible.

### [Parallel Tree-Construction for Empty-Space Skipping](5037-parallel-tree-construction-for-empty-space-skipping.md)
Reconstructs acceleration structures (e.g., octrees) on the GPU at runtime. Chosen for dynamic scenes or frequent transfer function changes requiring real-time structure updates.

### [Transfer-Function Independent Empty-Space Skipping](5038-transfer-function-independent-empty-space-skipping.md)
Stores visibility information independent of the transfer function. Chosen when decoupling spatial culling from opacity evaluation is required to avoid reconstruction overhead.

### [SparseLeap](5152-sparseleap.md)
Hybrid object-order/rasterization method that rasterizes occupancy geometry to merge segments. Chosen for large, sparse volumes with fine structures to reduce fragmentation and traversal costs.

### [Homogeneity Acceleration](5754-homogeneity-acceleration.md)
Approximates uniform, non-empty regions with fewer samples. Chosen when regions contain data but lack internal variation, extending basic empty-space skipping.

### Brick Classification and Empty Space Skipping
Classifies volume bricks into skipped, saturated, or sampled categories using a shared metadata array. Chosen for brick-based rendering where large sets of work can be bypassed. Pitfall: Metadata invalidation on transfer function change requires background recomputation.

### TF-Adaptive Empty-Space Skip
Skips cells if their max deviation is below a cutoff derived from live contrast/trim uniforms. Chosen for precise zero-opacity skipping. Pitfall: In dense structures, the overhead of pyramid checks can cause performance regression.

## Operating characteristics
*   **Time Complexity:** Depends on the data structure. Octrees/BSPs offer $O(\log N)$ traversal but higher per-step overhead. Uniform grids/occupancy maps offer $O(1)$ lookup but may skip less efficiently in sparse data [S6, unofficial].
*   **Memory Complexity:** Requires auxiliary storage for the empty-space structure (e.g., octree nodes, distance maps, occupancy grids). Storage for occupancy volumes of a 300-frame video is ~8 MB [S13].
*   **Performance:** Significant speedups in sparse volumes. E.g., RC-β (with ESS) achieved 23.4 fps vs 4.8 fps (RC) for a head dataset [S7, unofficial]. ESS increased rendering speed of a single MLP model from 1345 ms to 133 ms [S13].
*   **Strengths:** Reduces sample count dramatically in empty regions; compatible with various rendering pipelines (CPU/GPU) [S1, S5, unofficial].
*   **Limitations:** Overhead of structure traversal/query; invalidation on transfer function change (unless TF-independent); potential performance regression in dense volumes where skips are rare [S6, S16, unofficial].
*   **When to Use:** Sparse volumes, large datasets with significant empty space, interactive applications requiring fast frame rates.
*   **When Not to Use:** Dense volumes with little empty space (overhead may exceed benefits), or when memory bandwidth is the primary bottleneck and the auxiliary structure is large [S6, unofficial].

## Background
Empty-space skipping originated as a fundamental optimization in direct volume rendering to address the high computational cost of sampling transparent regions [S1, unofficial][S10, book]. Early methods used hierarchical representations (octrees, kd-trees) to store min/max bounds [S6, S8, unofficial]. The technique has been widely adopted in GPU-based raycasting, often combined with early ray termination [S2, S7, unofficial]. Recent developments include integration with neural radiance fields (NeRFs) via occupancy grids (e.g., Instant-NGP, NerfAcc) and parallel GPU-based tree construction for dynamic scenes [S5, unofficial][S13][S14, unofficial].

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
