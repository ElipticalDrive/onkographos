# Distance Map Acceleration

Also called: Accelerated Volume Rendering with Chebyshev Distance Maps, Chebyshev Distance Maps, Chebyshev distance map ray traversal, Distance Field Ray Marching Acceleration, Empty space skipping using distance maps, Ray Traversal with Distance Map Skipping

## Overview
Distance Map Acceleration is a ray traversal technique that uses a precomputed Chebyshev distance map, derived from an occupancy map, to skip sampling points in empty regions of a volumetric image [S1]. It reduces the number of texture samples and improves frame rates for volume and isosurface rendering by allowing rays to leap past inactive blocks without evaluating the transfer function or volume texture [S1]. Unlike sibling methods that may use hierarchical structures or simple occupancy grids for binary skipping, this method encodes the *distance* to the nearest occupied region, enabling larger, more informed skip steps [S1]. It builds on [Occupancy Grid and Map-Based Empty Space Skipping](3852-occupancy-grid-and-map-based-empty-space-skipping.md) by adding a distance field to the occupancy data.

## How it works
The method consists of an offline/preprocessing phase to generate acceleration structures and a runtime phase for ray traversal.

### Inputs and Outputs
**Inputs:**
*   Volume texture $V$ of dimensions $d_{\text{vol}} = (d_x, d_y, d_z)$ [S1].
*   Transfer function (TF) parameters defining opacity and gradient thresholds [S2, unofficial].
*   Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ [S1].

**Outputs:**
*   Final color and opacity for each pixel [S1].
*   Occupancy map $O$ and Distance map $D$ (generated during preprocessing) [S1].

### Data Structures
1.  **Occupancy Map ($O$):** A 3D texture of dimensions $d_{\text{occ}} = d_{\text{vol}} / B$, where $B$ is the block size [S1]. Each voxel $u_{\text{occ}}$ stores a binary value indicating if the corresponding volume block is active [S1].
    *   **Layout:** 1 component (e.g., `float` or `uint`).
    *   **Size:** $1/64^{\text{th}}$ of the volume memory for $B=4$ [S1].
2.  **Distance Map ($D$):** A 3D texture of the same dimensions as $O$ [S1]. Each voxel stores the Chebyshev distance (L-infinity norm) to the nearest occupied block [S1].
    *   **Layout:** 1 component (e.g., `float`).
    *   **Value:** Distance in units of occupancy map voxels [S1].
3.  **Gradient Map ($G$):** (Optional) A 3D texture storing precomputed gradient magnitudes for each voxel in $V$ [S1].
    *   **Layout:** 1 component (`float`).
    *   **Generation:** Computed via the tetrahedron method using 4 neighbor samples [S1].

### Parameters
*   $B \in \mathbb{N}^3$: Block size. Default $B=4$ [S1][S2, unofficial].
*   $f \ge 1$: Sampling factor for quality adjustment. Ensures at least one sample per voxel on average [S1].
*   $\tau$: Isosurface threshold (for isosurface rendering) [S1].
*   `imin`, `imax`: Intensity thresholds for TF. Default `imin`=0.1, `imax`=1.0 [S2, unofficial].
*   `gmin`, `gmax`: Gradient thresholds for TF. Default `gmin`=0.0, `gmax`=0.2 [S2, unofficial].
*   `clip_distance`: Distance from camera for volume clipping. Default 1.0f in benchmark mode [S2, unofficial].

### Preprocessing Steps
1.  **Generate Gradient Map (Optional):** If `use_precomputed_gradient` is true, dispatch a compute shader to calculate gradient magnitudes for all voxels in $V$ using the tetrahedron method [S1][S2, unofficial].
    *   *Note:* The tetrahedron method computes gradient direction $\mathbf{g}$ as:
        $$ \mathbf{g} = 0.25 \cdot \left( k_{xyy} V(\mathbf{p} + k_{xyy}) + k_{yyx} V(\mathbf{p} + k_{yyx}) + k_{yxy} V(\mathbf{p} + k_{yxy}) + k_{xxx} V(\mathbf{p} + k_{xxx}) \right) $$
        where $k = (1, -1)$ and $\mathbf{p}$ is the voxel position [S1]. The magnitude is $\|\mathbf{g}\|$ [S1].
2.  **Generate Occupancy Map:** Dispatch a compute shader. For each block defined by $u_{\text{occ}}$:
    *   Determine volume block extents:
        $$ u_{\text{vol}}^{\min} = B \cdot u_{\text{occ}} $$
        $$ u_{\text{vol}}^{\max} = \min(u_{\text{vol}}^{\min} + B, d_{\text{vol}}) - 1 $$
    *   **Volume Rendering Criterion:** The block is occupied if any voxel in $[u_{\text{vol}}^{\min}, u_{\text{vol}}^{\max}]$ has non-zero alpha according to the TF [S1].
    *   **Isosurface Rendering Criterion:** The block is occupied if the isosurface threshold $\tau$ lies between the min and max voxel values in the block, considering a single-voxel halo around the region [S1].
    *   Write the result to $O[u_{\text{occ}}]$ [S1].
3.  **Generate Distance Map:** Convert $O$ to $D$ using a multi-stage Chebyshev distance transform algorithm on the GPU [S1][S2, unofficial]. This computes the L-infinity distance from each empty block to the nearest occupied block [S1].

### Runtime Ray Traversal
1.  **Ray-Volume Intersection:** Compute entry $t_{\text{entry}}$ and exit $t_{\text{exit}}$ of the ray with the volume bounding box [S1].
2.  **Sampling Count:** Calculate the number of equidistant sampling points $n$:
    $$ n = \lceil \max \vec{x}(\mathbf{d}) \cdot \| \mathbf{t}_{\text{exit}} - \mathbf{t}_{\text{entry}} \| \cdot f \rceil $$
    where $\max \vec{x}(\mathbf{d})$ is the maximum component of the ray direction vector [S1].
3.  **Step Size:** Calculate the change in normalized texture coordinates:
    $$ \Delta t = \frac{t_{\text{exit}} - t_{\text{entry}}}{n - 1} $$
    The $i$-th sampling point is $\mathbf{t}_i = \mathbf{t}_{\text{entry}} + i \cdot \Delta t$ for $i = 0 \dots n-1$ [S1].
4.  **Distance Map Skipping:**
    *   Map the current ray position $\mathbf{t}$ to occupancy map coordinates $u_{\text{occ}}$:
        $$ u_{\text{occ}} = \frac{d_{\text{vol}}}{B} \mathbf{t} $$
        (More precisely, $t_{\text{occ}} = \frac{d_{\text{vol}}}{B \cdot d_{\text{occ}}} t_{\text{vol}}$) [S1].
    *   Query $D[u_{\text{occ}}]$. Let $d_{\text{skip}}$ be this value [S1].
    *   If $D[u_{\text{occ}}] > 0$ (empty region), calculate the number of steps $\Delta i_j$ to skip along dimension $j \in \{x, y, z\}$ to reach the next occupied region:
        $$ \Delta i_j = \begin{cases} \lceil (1 + \lfloor u_j \rfloor - u_j) / \Delta u_j \rceil, & \text{if } \Delta u_j > 0 \\ \lceil (\lfloor u_j \rfloor - u_j) / \Delta u_j \rceil, & \text{otherwise} \end{cases} $$
        where $u_j$ is the current coordinate in the occupancy map and $\Delta u_j$ is the step size in occupancy map units [S1].
    *   Advance the ray index $i$ by $\max_j(\Delta i_j)$ [S1].
    *   If $D[u_{\text{occ}}] == 0$ (occupied region), perform standard volume rendering: sample $V$ and $G$ (if available) at $\mathbf{t}_i$, apply TF, and accumulate color/opacity [S1][S2, unofficial].
5.  **Clipping:** Clip the volume against a plane at `clip_distance` from the camera and against the depth buffer [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def distance_map_accelerated_render(ray_origin, ray_direction, volume_texture, occupancy_map, distance_map, block_size, sampling_factor):
    # 1. Compute ray-volume intersection
    t_entry, t_exit = intersect_box(ray_origin, ray_direction, volume_bounds)
    if t_entry > t_exit:
        return transparent_color

    # 2. Determine sampling parameters
    max_dir_component = max(abs(ray_direction.x), abs(ray_direction.y), abs(ray_direction.z))
    ray_length = length(t_exit - t_entry)
    n_samples = ceil(max_dir_component * ray_length * sampling_factor)
    
    if n_samples < 1:
        n_samples = 1
        
    delta_t = (t_exit - t_entry) / (n_samples - 1)
    
    # 3. Map ray to occupancy map grid
    # t_vol is normalized [0,1] in volume space
    # t_occ is normalized [0,1] in occupancy map space
    # u_occ is integer index in occupancy map
    # d_vol, d_occ are dimensions
    d_vol = volume_texture.dimensions
    d_occ = occupancy_map.dimensions
    
    # Precompute scaling factor
    occ_scale = d_vol / (block_size * d_occ) # Simplified from source: u_occ = (d_vol / B) * t_vol
    
    current_t = t_entry
    accumulated_color = vec4(0.0)
    accumulated_opacity = 0.0
    
    for i in range(n_samples):
        # Map current ray position to occupancy map index
        t_vol_normalized = current_t # Assuming t is normalized [0,1]
        u_occ = floor(t_vol_normalized * d_vol / block_size)
        
        # Query distance map
        dist = distance_map.sample(u_occ)
        
        if dist > 0:
            # Empty region: Skip
            # Calculate steps to next occupied block
            # This requires calculating the distance to the boundary of the current block
            # and dividing by the step size in occupancy map units.
            # Source formula for delta_i_j is complex; simplified logic:
            # Skip 'dist' blocks in occupancy map space.
            # Convert 'dist' in occupancy units to steps in ray index.
            
            # Step size in occupancy map units per ray sample
            delta_u_occ = (t_exit - t_entry) / (n_samples - 1) * (d_vol / block_size)
            
            # Number of samples to skip
            samples_to_skip = ceil(dist / delta_u_occ)
            
            # Advance ray
            current_t += samples_to_skip * delta_t
            i += samples_to_skip - 1 # Adjust loop counter
            continue
            
        else:
            # Occupied region: Sample
            # Sample volume texture
            value = volume_texture.sample(current_t)
            
            # Sample gradient map (if available)
            if gradient_map_available:
                grad = gradient_map.sample(current_t)
            else:
                grad = 0.0
                
            # Apply Transfer Function
            color, opacity = transfer_function(value, grad)
            
            # Front-to-back compositing
            alpha = 1.0 - accumulated_opacity
            accumulated_color += color * opacity * alpha
            accumulated_opacity += opacity * alpha
            
            if accumulated_opacity >= 1.0:
                break
                
        current_t += delta_t
        
    return accumulated_color
```

## Subtopics

### [Partitioned Distance Map Combination](1191-partitioned-distance-map-combination.md)
Runtime assembly of partitioned distance maps into a global map using ping-pong buffers.

### [Occupancy Map Generation](1197-occupancy-map-generation.md)
GPU compute shader routine to generate the binary occupancy map from volume and TF textures.

## Operating characteristics
*   **Time Complexity:** Preprocessing is $O(N)$ where $N$ is the number of voxels/blocks, dominated by the distance transform [S1]. Runtime is $O(S)$ where $S$ is the number of samples in occupied regions, significantly reduced from the total ray length by skipping [S1].
*   **Memory Complexity:** Occupancy map is $1/B^3$ of the volume size. Distance map is same size as occupancy map. Gradient map is same size as volume if precomputed [S1][S2, unofficial].
*   **Performance:** Reduces texture samples in empty space, improving frame rates [S1]. Performance depends on the sparsity of the volume and the block size $B$ [S1].
*   **Strengths:** Efficient empty space skipping, simple data structure, GPU-friendly [S1][S2, unofficial].
*   **Limitations:** Distance map generation cost, memory overhead for distance map, less effective for dense volumes [S1].
*   **When to use:** Sparse volumes with large empty regions [S1].
*   **When not to use:** Dense volumes where skipping provides little benefit, or memory is constrained [S1].

## Background
The method was introduced by Lachlan Deakin and Mark Knackstedt in 2019 in "Accelerated Volume Rendering with Chebyshev Distance Maps" [S1][S2, unofficial]. It is implemented in the open-source Vulkan renderer `VkVolume` [S2, unofficial]. The approach extends earlier work on occupancy grids by incorporating distance fields to allow larger skip steps [S1].

## Sources
[S1] Efficient ray casting of volumetric images using distance maps for empty space skipping, https://doi.org/10.1007/s41095-019-0155-y, fetched 2026-10-24
[S2] GitHub - LDeakin/VkVolume: Vulkan-based volume renderer with efficient ..., https://github.com/LDeakin/VkVolume, fetched 2026-10-26 (unofficial)
