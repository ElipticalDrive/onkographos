# Partitioned Distance Map (PDM) Acceleration

Also called: Accelerating Transfer Function Update for Distance Map based Volume Rendering, Intensity-Partitioned Occupancy/Distance Maps, PDM-based Empty Space Skipping, Partitioned Distance Map Update, Direct Volume Rendering with Partitioned Distance Maps (PDM) based Empty Space Skipping

## Overview
Direct Volume Rendering (DVR) with Partitioned Distance Maps (PDM) based Empty Space Skipping (ESS) is an acceleration technique for distance-field ray marching that subdivides the volume's intensity range into $n$ partitions to precompute individual distance maps [S1]. When a transfer function (TF) changes, only the relevant PDMs are merged to form the final distance map $D'$, avoiding a full recomputation of the distance transform [S1]. Unlike the base method [Partitioned Distance Map Combination](1191-partitioned-distance-map-combination.md) which recomputes the global distance map on every TF change, PDM-based ESS trades higher memory usage and one-time initialization cost for significantly faster TF update times, maintaining high frame rates during frequent TF adjustments [S1][S2, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- Volume dataset $V$ with intensity values [S1].
- Number of intensity partitions $n$ [S1].
- Block size $b$ for block-reduction (typically $b=4$) [S1].
- Transfer Function (TF) mapping intensities to RGBA [S2, unofficial].
- Camera parameters (position, view matrix) [S2, unofficial].

**Outputs:**
- Final distance map $D'$ used for ESS during ray casting [S1].
- Rendered image via ray casting [S2, unofficial].

### Data Structures

**Partitioned Occupancy Maps (POMs) and Partitioned Distance Maps (PDMs):**
- **Type:** `uint8` (unsigned 8-bit integer) [S1].
- **Layout:** Block-reduced dimensions. If the volume has dimensions $D_x \times D_y \times D_z$, the map dimensions are $\lceil D_x/b \rceil \times \lceil D_y/b \rceil \times \lceil D_z/b \rceil$ [S1].
- **Content:**
  - POM: Stores occupancy status. Value `0` indicates occupied (contains at least one voxel in the partition's intensity range), `1` indicates empty [S2, unofficial].
  - PDM: Stores Chebyshev distance to the nearest non-empty block within that specific intensity partition. Distances $> 255$ are clamped to 255 [S1].

**Uniform Buffer Object (UBO) Layout (std140, binding 1):**
| Offset | Size | Field | Type |
| :--- | :--- | :--- | :--- |
| 0 | 64 | `model` | `mat4` |
| 64 | 64 | `view` | `mat4` |
| 128 | 64 | `inverseModelView` | `mat4` |
| 192 | 4 | `sampling_factor` | `float` |
| 196 | 4 | `renderContentType` | `int` |
| 200 | 4 | `ditheringMode` | `int` |
| 204 | 4 | `earlyRayTermination` | `bool` |
| 208 | 4 | `essType` | `int` |
| 212 | 4 | `gradientType` | `int` |
| 216 | 4 | `itf_gamma_correction` | `bool` |
| 220 | 4 | `gamma` | `float` |
| 224 | 4 | `enable_bicubic_filtering` | `bool` |
| 228 | 4 | `reuse_normals` | `bool` |
| 232 | 4 | `lightshading_factor` | `float` |
| 240 | 12 | `lightDirection` | `vec3` |
| 252 | 4 | `light_ambient_factor` | `float` |
| 256 | 4 | `light_diffuse_factor` | `float` |
| 260 | 4 | `light_specular_factor` | `float` |
| 264 | 4 | `light_specular_shininess` | `float` |

**Texture Bindings:**
- `texVolume` (binding 0): `sampler3D`
- `texGradient` (binding 1): `sampler3D`
- `texNoise` (binding 2): `sampler2D`
- `texITF` (binding 3): `mediump sampler2D`
- `texOccupancyMap` (binding 4): `mediump usampler3D`
- `texDistanceMap` (binding 5): `mediump usampler3D`

### Preprocessing Steps
1.  **Partition Definition:** Subdivide the total intensity range of the volume into $n$ partitions. Each partition $p$ is defined by lower and upper integer intensity borders $\rho_{min}^p$ and $\rho_{max}^p$ [S1].
2.  **POM Computation:** Compute $n$ Partitioned Occupancy Maps (POMs). The original volume is block-reduced into non-overlapping blocks of size $b^3$. A block is considered occupied in partition $p$ if it contains at least one voxel with an intensity value within $[\rho_{min}^p, \rho_{max}^p]$ [S1].
3.  **PDM Computation:** Compute $n$ Partitioned Distance Maps (PDMs) from the POMs. Each PDM encodes the Chebyshev distance to the nearest non-empty voxel block within that specific intensity partition [S1]. This is typically done using a multi-pass distance transform algorithm on the GPU [S2, unofficial].

### Update Step (TF Change)
When the TF is updated, the final distance map $D'$ is assembled without recomputing the underlying PDMs:
1.  **Determine Active Partitions:** Identify the set $S$ of partition indices $p$ such that the TF maps at least one intensity value in partition $p$ to a non-zero alpha value [S1].
2.  **Merge PDMs:** Compute the element-wise minimum of the corresponding PDMs to form $D'$ [S1]:
    $$ D'(x, y, z) = \min_{p \in S} (PDM_p(x, y, z)) $$
    This merging is performed in a GLSL Compute Shader [S1]. Up to a maximum of 6 PDMs are bound as input textures per pass, aggregating distances to produce $D'$ [S1]. If more PDMs are needed, multiple passes are used with flip-flopping input/output textures [S1].

### Rendering Steps (Ray Casting)
1.  **Ray Origin and Direction:**
    -   Get camera position in object coordinates: `ray_origin = inverseModelView * vec4(view[0][3], view[1][3], view[2][3], 1.0f)` [S2, unofficial].
    -   Intersect ray with bounding box (AABB: minBB=(-0.5,-0.5,-0.5), maxBB=(0.5,0.5,0.5)) using the slab method to find entry ($t_0$) and exit ($t_1$) distances [S2, unofficial].
2.  **Normalize to Texture Coordinates:**
    -   `ray_start = (ray_origin + ray_direction * t_0 - minBB) / (maxBB - minBB)` [S2, unofficial].
    -   `ray_stop = (ray_origin + ray_direction * t_1 - minBB) / (maxBB - minBB)` [S2, unofficial].
3.  **Calculate Step Parameters:**
    -   `ray = ray_stop - ray_start` [S2, unofficial].
    -   `ray_length = length(ray)` [S2, unofficial].
    -   Check for degenerate rays: If `ray_length` is infinite, NaN, or $> \sqrt{3}$ (~1.732051f), return immediately [S2, unofficial].
    -   `n_steps = int(ceil(float(dim_max) * ray_length * sampling_factor))` where `dim_max` is the maximum dimension of the volume texture [S2, unofficial].
    -   `step_vector = normalize(ray) * ray_length / (float(n_steps) - 1.0f)` [S2, unofficial].
4.  **Ray Marching Loop:**
    -   For each step $i$ from $0$ to $n\_steps - 1$:
        -   Calculate sampling position: `samplingPos = ray_start + step_vector * float(i)` [S2, unofficial].
        -   **Empty Space Skipping:** Use $D'$ to determine the step size. The distance map value at the current block position indicates how many blocks can be skipped. Convert this block distance to world/texture space distance and advance the ray [S1][S2, unofficial].
        -   **Volume Sampling:**
            -   Sample intensity from `texVolume` at `samplingPos` [S2, unofficial].
            -   **Gradient Calculation:**
                -   If `gradientType == 0` (on-the-fly): Approximate gradient using tetrahedron technique. Sample 4 voxels at offsets `k.xyy`, `k.yyx`, `k.yxy`, `k.xxx` where `k=ivec2(-1,1)`. Compute `gradientVec = 0.25 * sum(k_i * value_i)`. Normalize and store magnitude in alpha channel [S2, unofficial].
                -   If `gradientType == 1` (precomputed): Sample from `texGradient` at `samplingPos + 0.5/vec3(volumeSize)`. Normalize xyz components and store original magnitude in alpha [S2, unofficial].
            -   **Interpolation:** If `enable_bicubic_filtering` is true, calculate tricubic interpolation weights and mix samples [S2, unofficial].
        -   **Transfer Function:** Map intensity to RGBA using `texITF`. Apply gamma correction if `itf_gamma_correction` is true [S2, unofficial].
        -   **Compositing:** Accumulate color and opacity. If `earlyRayTermination` is true, break loop if accumulated opacity exceeds a threshold [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_pom(volume, block_size, partition_min, partition_max):
    """
    Computes a Partitioned Occupancy Map for a specific intensity partition.
    volume: 3D list of voxel intensities.
    block_size: int, side length of blocks (e.g., 4).
    partition_min, partition_max: int, intensity range.
    Returns: 3D list of uint8 (0=occupied, 1=empty).
    """
    dim_z, dim_y, dim_x = len(volume), len(volume[0]), len(volume[0][0])
    pom_dim_z = (dim_z + block_size - 1) // block_size
    pom_dim_y = (dim_y + block_size - 1) // block_size
    pom_dim_x = (dim_x + block_size - 1) // block_size
    pom = [[[1 for _ in range(pom_dim_x)] for _ in range(pom_dim_y)] for _ in range(pom_dim_z)]
    
    for z in range(pom_dim_z):
        for y in range(pom_dim_y):
            for x in range(pom_dim_x):
                x_start, x_end = x * block_size, min((x + 1) * block_size, dim_x)
                y_start, y_end = y * block_size, min((y + 1) * block_size, dim_y)
                z_start, z_end = z * block_size, min((z + 1) * block_size, dim_z)
                
                occupied = False
                for vz in range(z_start, z_end):
                    for vy in range(y_start, y_end):
                        for vx in range(x_start, x_end):
                            if partition_min <= volume[vz][vy][vx] <= partition_max:
                                occupied = True
                                break
                        if occupied: break
                    if occupied: break
                pom[z][y][x] = 0 if occupied else 1
    return pom

def compute_pdm(pom):
    """
    Computes Chebyshev Distance Map from POM.
    pom: 3D list of uint8 (0=occupied, 1=empty).
    Returns: 3D list of uint8 (distance to nearest 0, clamped to 255).
    """
    dim_z, dim_y, dim_x = len(pom), len(pom[0]), len(pom[0][0])
    dist = [[[255 for _ in range(dim_x)] for _ in range(dim_y)] for _ in range(dim_z)]
    
    # Multi-pass distance transform (simplified logic for demonstration)
    # Pass 1: Forward
    for z in range(dim_z):
        for y in range(dim_y):
            for x in range(dim_x):
                if pom[z][y][x] == 0:
                    dist[z][y][x] = 0
                else:
                    d = 255
                    if x > 0: d = min(d, dist[z][y][x-1] + 1)
                    if y > 0: d = min(d, dist[z][y-1][x] + 1)
                    if z > 0: d = min(d, dist[z-1][y][x] + 1)
                    dist[z][y][x] = d
    # Pass 2: Backward
    for z in range(dim_z-1, -1, -1):
        for y in range(dim_y-1, -1, -1):
            for x in range(dim_x-1, -1, -1):
                d = dist[z][y][x]
                if x < dim_x - 1: d = min(d, dist[z][y][x+1] + 1)
                if y < dim_y - 1: d = min(d, dist[z][y+1][x] + 1)
                if z < dim_z - 1: d = min(d, dist[z+1][y][x] + 1)
                dist[z][y][x] = d
    return dist

def merge_pdms(pdms, active_indices):
    """
    Merges selected PDMs into a final distance map D'.
    pdms: list of 3D lists (PDMs).
    active_indices: list of indices into pdms.
    Returns: 3D list of uint8 (D').
    """
    if not active_indices:
        dim_z, dim_y, dim_x = len(pdms[0]), len(pdms[0][0]), len(pdms[0][0][0])
        return [[[255 for _ in range(dim_x)] for _ in range(dim_y)] for _ in range(dim_z)]
    
    d_prime = [row[:] for row in pdms[active_indices[0]]]
    for idx in active_indices[1:]:
        for z in range(len(d_prime)):
            for y in range(len(d_prime[0])):
                for x in range(len(d_prime[0][0])):
                    d_prime[z][y][x] = min(d_prime[z][y][x], pdms[idx][z][y][x])
    return d_prime

def ray_cast(volume, d_prime, camera_matrix, tf, params):
    """
    Performs direct volume rendering using PDM-based ESS.
    volume: 3D list of intensities.
    d_prime: 3D list of uint8 (final distance map).
    camera_matrix: 4x4 list of lists (inverseModelView).
    tf: Transfer function function/intensity map.
    params: dict with sampling_factor, earlyRayTermination, etc.
    Returns: 2D list of RGBA tuples.
    """
    dim_z, dim_y, dim_x = len(volume), len(volume[0]), len(volume[0][0])
    height, width = params.get('height', 100), params.get('width', 100)
    
    # Ray origin from camera matrix
    ray_origin = [camera_matrix[0][3], camera_matrix[1][3], camera_matrix[2][3]]
    
    minBB = [-0.5, -0.5, -0.5]
    maxBB = [0.5, 0.5, 0.5]
    
    image = [[[0.0, 0.0, 0.0, 0.0] for _ in range(width)] for _ in range(height)]
    
    for y in range(height):
        for x in range(width):
            # Simplified ray direction calculation for front face
            ray_dir = [x / width - 0.5, y / height - 0.5, -1.0]
            ray_len = math.sqrt(sum(c*c for c in ray_dir))
            ray_dir = [c/ray_len for c in ray_dir]
            
            # Intersect with Bounding Box (slab method)
            t0, t1 = intersect_slab(ray_origin, ray_dir, minBB, maxBB)
            if t0 > t1: continue
            
            ray_start = [(ray_origin[i] + ray_dir[i] * t0 - minBB[i]) / (maxBB[i] - minBB[i]) for i in range(3)]
            ray_stop = [(ray_origin[i] + ray_dir[i] * t1 - minBB[i]) / (maxBB[i] - minBB[i]) for i in range(3)]
            
            ray_vec = [ray_stop[i] - ray_start[i] for i in range(3)]
            ray_length = math.sqrt(sum(c*c for c in ray_vec))
            
            if ray_length <= 0 or ray_length > math.sqrt(3.0) or math.isnan(ray_length):
                continue
                
            dim_max = max(dim_z, dim_y, dim_x)
            n_steps = int(math.ceil(dim_max * ray_length * params['sampling_factor']))
            step_vector = [ray_vec[i] / ray_length * ray_length / (n_steps - 1.0) for i in range(3)]
            
            accumulated_color = [0.0, 0.0, 0.0]
            accumulated_alpha = 0.0
            
            for i in range(n_steps):
                sampling_pos = [ray_start[i] + step_vector[i] * i for i in range(3)]
                
                # ESS Step
                bx = int(sampling_pos[0] * dim_x / params['block_size'])
                by = int(sampling_pos[1] * dim_y / params['block_size'])
                bz = int(sampling_pos[2] * dim_z / params['block_size'])
                bx = max(0, min(bx, dim_x // params['block_size'] - 1))
                by = max(0, min(by, dim_y // params['block_size'] - 1))
                bz = max(0, min(bz, dim_z // params['block_size'] - 1))
                
                dist = d_prime[bz][by][bx]
                skip_distance = dist * params['block_size'] / dim_max
                sampling_pos = [sampling_pos[j] + step_vector[j] * (n_steps - 1.0) * skip_distance / ray_length for j in range(3)]
                
                intensity = sample_volume(volume, sampling_pos, params)
                rgba = apply_tf(tf, intensity, params)
                
                if params['earlyRayTermination'] and accumulated_alpha > 0.99:
                    break
                    
                c = [rgba[k] * rgba[3] * (1.0 - accumulated_alpha) for k in range(3)]
                accumulated_color = [accumulated_color[k] + c[k] for k in range(3)]
                accumulated_alpha += rgba[3] * (1.0 - accumulated_alpha)
                
            image[y][x] = accumulated_color
    return image

def intersect_slab(origin, direction, minBB, maxBB):
    t0, t1 = -math.inf, math.inf
    for i in range(3):
        if direction[i] != 0:
            t1 = min(t1, (maxBB[i] - origin[i]) / direction[i])
            t0 = max(t0, (minBB[i] - origin[i]) / direction[i])
        else:
            if origin[i] < minBB[i] or origin[i] > maxBB[i]:
                return (math.inf, -math.inf)
    return (t0, t1)

def sample_volume(volume, pos, params):
    # Bilinear/Trilinear interpolation placeholder
    return 0.5

def apply_tf(tf, intensity, params):
    return [0.0, 0.0, 0.0, 0.5]
```

### Parameters
-   **Block size $b$:** Side length of non-overlapping voxel blocks. Typical value: 4 [S1].
-   **Number of partitions $n$:** Number of intensity subranges. Evaluated values: 16, 32, 64, 128, 256 [S1].
-   **sampling_factor:** Float parameter scaling the number of sampling steps. Typical values depend on resolution/quality [S2, unofficial].
-   **essType:** Integer selecting ESS algorithm (e.g., distance ESS) [S2, unofficial].
-   **earlyRayTermination:** Boolean enabling/disabling early termination [S2, unofficial].
-   **enable_bicubic_filtering:** Boolean enabling/disabling tricubic interpolation [S2, unofficial].
-   **gradientType:** Integer (0: on-the-fly, 1: precomputed) [S2, unofficial].
-   **MAX_SEGMENTS:** Compile-time constant, value 64 [S2, unofficial].

### Edge Cases and Pitfalls
-   **Approximation Error:** $D'$ is an element-wise lower limit of the standard non-partitioned distance map $D$ ($D' \le D$) [S1]. This means ESS may skip fewer voxels (worse performance) if different intensity values in the same partition are mapped to both zero and non-zero alpha by the TF. However, it never skips more voxels than allowed, ensuring correctness [S1].
-   **Degenerate Rays:** If `ray_length` is infinite, NaN, or greater than $\sqrt{3}$ (~1.732051f), the ray is degenerate (e.g., start and endpoint coinciding or outside bounds). The shader returns immediately to prevent performance regressions [S2, unofficial].
-   **Data Type Clamping:** PDM/POM data types are `uint8`. Distances higher than 255 are clamped, which may affect accuracy for very large volumes or large block sizes [S1].

## Operating characteristics
-   **Time Complexity:**
    -   **Preprocessing:** $O(n \cdot V)$ where $V$ is the number of voxels and $n$ is the number of partitions. One-time initialization cost increases with $n$ (e.g., 905.2ms for 256 partitions on Manix dataset) [S1].
    -   **TF Update:** $O(k \cdot V_{blocks})$ where $k$ is the number of active partitions and $V_{blocks}$ is the number of blocks. Significantly faster than full distance map recomputation (e.g., 0.20ms to 1.30ms for 16-256 partitions vs 58.1ms for original method on Manix) [S1].
    -   **Rendering:** $O(R \cdot S)$ where $R$ is the number of rays and $S$ is the average number of steps per ray. ESS reduces $S$.
-   **Memory Complexity:** $O(n \cdot V_{blocks})$ for storing $n$ PDMs. Higher memory usage compared to a single distance map [S1].
-   **Measured Performance:**
    -   **Manix (512x512x460):** Original distance map: 58.1ms. PDM (16 partitions) update: 0.20ms. PDM (256 partitions) update: 1.30ms [S1].
    -   **Stag Beetle (832x832x494):** Original: 146.4ms. PDM (16 partitions) update: 0.41ms. PDM (256 partitions) update: 2.58ms [S1].
    -   **Kingsnake (1024x1024x795):** Original: 228.2ms. PDM (16 partitions) update: 0.80ms. PDM (256 partitions) update: 6.71ms [S1].
-   **Scaling:** Performance gain in TF update time is consistent across datasets, but initialization cost scales with $n$ and volume size [S1].
-   **Strengths:** Fast TF updates, high frame rates during interactive TF adjustment, correctness guaranteed (no over-skipping) [S1].
-   **Limitations:** Higher memory usage, slower one-time initialization, potential performance degradation in rendering if ESS skips fewer voxels due to approximation [S1].
-   **When to use:** Interactive volume rendering with frequent TF changes [S1].
-   **When not to use:** Static TFs, memory-constrained environments, or when one-time initialization time is critical [S1].

## Background
-   **Origin:** Proposed by Michael Rauter in "Accelerating Transfer Function Update for Distance Map based Volume Rendering" (IEEE VIS 2024) [S1].
-   **History:** Extends the work of Deakin and Knackstedt on distance map based ESS [S1][S2, unofficial]. The original approach required full distance map recomputation on TF change, which was a limiting performance factor [S1]. Faludi et al. [5] used an octree of bitfields, reporting ray marching times 35% slower than Deakin’s approach [S1]. PDM-based ESS improves upon this by using partitioned distance maps for faster updates [S1].
-   **Usage:** Used in GPU-based direct volume rendering, particularly in virtual reality and interactive visualization applications [S1][S2, unofficial].

## Sources
[S1] Accelerating Transfer Function Update for Distance Map based Volume Rendering, https://doi.org/10.1109/vis55277.2024.00042, fetched 2026-10-26
[S2] CalamityMichL/DVR_ESS_PDM, https://github.com/CalamityMichL/DVR_ESS_PDM, fetched 2026-10-26 (unofficial)
