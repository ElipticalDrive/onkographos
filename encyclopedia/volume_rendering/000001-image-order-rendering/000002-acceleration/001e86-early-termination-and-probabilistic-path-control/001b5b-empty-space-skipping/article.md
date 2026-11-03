# Empty-space skipping

Also called: Block Empty Space Skipping, Chebyshev Empty Space Skipping, Empty Region Skipping, Occupancy Grid Pruning

## Overview
Empty-space skipping (ESS) is a ray marching optimization that accelerates volume rendering by skipping regions of the scene that are empty or occluded, thereby reducing the number of samples required to evaluate the radiance field [S1, S2, S3]. It sets itself apart from sibling methods like [GL-NeRF](803-gl-nerf.md) or [VoISDF](821-voisdf-volume-rendering-and-adaptive-sampling.md) by explicitly maintaining auxiliary data structures (occupancy grids or distance maps) to prune traversal, whereas other siblings may rely on hierarchical tree structures or adaptive sampling rates without explicit empty-region culling [S2, S3].

## How it works
**Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, volume dimensions, sampling factor, and precomputed occupancy/distance maps [S1].
**Outputs:** Accumulated color and opacity [S1].

**Data Structures:**
*   **Occupancy Grid / Distance Map:** A 3D texture (`texOccupancyMap` or `texDistanceMap`) storing precomputed values, accessed via `texelFetch` [S1].
    *   `texOccupancyMap`: Stores binary occupancy (1 if occupied, 0 if empty) [S1].
    *   `texDistanceMap`: Stores distance to the nearest occupied voxel [S1].

**Parameters:**
*   `essType`: Integer flag. `1` = Block Empty Space Skipping; `2` = Chebyshev Empty Space Skipping [S1].
*   `alpha_thre`: Opacity threshold for empty regions. Typical value: $10^{-2}$ [S2].
*   `early_stop_eps`: Transmittance threshold for early termination. Typical value: $10^{-4}$ [S2].
*   `sampling_factor`: Controls step count relative to volume dimensions [S1].
*   `ditheringMode`: Integer flag for ray dithering. `1` = procedural noise; `2` = sampled from `texNoise` texture [S1].

**Algorithm Steps:**

1.  **Ray Initialization:** Calculate the number of steps and the step vector [S1].
    $$ n_{\text{steps}} = \lceil \text{dim}_{\max} \times \text{ray\_length} \times \text{sampling\_factor} \rceil $$
    $$ \text{step\_vector} = \frac{\mathbf{d}}{\|\mathbf{d}\|} \times \frac{\text{ray\_length}}{n_{\text{steps}} - 1} $$
    Handle degenerate rays (infinite, NaN, or length $> \sqrt{3}$) by terminating immediately [S1].

2.  **Distance Map Coordinate Setup:** Compute mapping parameters between volume space and distance map texture space [S1].
    $$ \text{dim\_distance\_map} = \text{textureSize}(\text{texDistanceMap}, 0).\text{xyz} $$
    $$ \text{block\_size} = \lceil \text{volumeSize} / \text{dim\_distance\_map} \rceil $$
    $$ \text{volume\_to\_distance\_map} = \text{volumeSize} / (\text{block\_size} \times \text{dim\_distance\_map}) $$
    $$ \text{step\_dist\_texel} = \text{step\_vector} \times \text{volumeSize} / \text{block\_size} $$
    $$ \text{step\_dist\_texel\_inv} = 1.0 / \text{step\_dist\_texel} $$

3.  **Dithering (Optional):** Compute a random offset to reduce banding artifacts [S1].
    If `ditheringMode == 1`:
    $$ \text{random} = \text{fract}(\sin(\text{gl\_FragCoord}.x \times 12.9898 + \text{gl\_FragCoord}.y \times 78.233) \times 43758.5453) $$
    If `ditheringMode == 2`:
    $$ \text{random} = \text{texture}(\text{texNoise}, \text{gl\_FragCoord.xy} / 256).r $$

4.  **Ray Marching Loop:** Iterate with index $i$. For each step:
    a. **Sampling Position:**
    $$ \text{samplingPos} = \mathbf{o} + (i + \text{random}) \times \text{step\_vector} $$
    b. **Map to Distance Map Coordinates:**
    $$ \text{pos\_distance\_map} = \text{volume\_to\_distance\_map} \times \text{samplingPos} $$
    $$ u = \text{pos\_distance\_map} \times \text{dim\_distance\_map} $$
    $$ u_i = \text{clamp}(\lfloor u \rfloor, 0, \text{dim\_distance\_map} - 1) $$
    c. **Fetch Distance/Occupancy:**
    Determine index based on ray direction signs for directional distance maps (if applicable):
    $$ \text{distance\_map\_idx} = (\mathbf{d}.z < 0 ? 1 : 0) + (\mathbf{d}.y < 0 ? 2 : 0) + (\mathbf{d}.x < 0 ? 4 : 0) $$
    Fetch value:
    If `essType == 1` (Block):
    $$ \text{dist} = \text{texelFetch}(\text{texOccupancyMap}, u_i, 0).x $$
    If `essType == 2` (Chebyshev):
    $$ \text{dist} = \text{texelFetch}(\text{texDistanceMap}, u_i, 0).x $$
    d. **Calculate Skip Delta:**
    Compute fractional offset for block alignment:
    $$ r = \text{clamp}(u_i - u, -1.0, 0.0) $$
    If `essType == 1` (Block ESS):
    $$ i_{\text{delta\_xyz}} = (\text{step}(0.0, \text{step\_dist\_texel\_inv}) + r) \times \text{step\_dist\_texel\_inv} $$
    If `essType == 2` (Chebyshev ESS):
    $$ i_{\text{delta\_xyz}} = (\text{step}(0.0, -\text{step\_dist\_texel\_inv}) + \text{sign}(\text{step\_dist\_texel\_inv}) \times \text{dist} + r) \times \text{step\_dist\_texel\_inv} $$
    Update loop index:
    $$ i_{\text{delta}} = \max(1, \lceil \min(i_{\text{delta\_xyz}}.x, i_{\text{delta\_xyz}}.y, i_{\text{delta\_xyz}}.z) \rceil) $$
    $$ i \leftarrow i + i_{\text{delta}} $$
    e. **Backtracking / Empty Space Handling:**
    If `dist == 0` (empty space reached):
    $$ i_{\text{delta\_backwards}} = -\lceil \text{sampling\_factor} \rceil $$
    $$ \text{skipping} = \text{false} $$
    $$ i = \max(i + i_{\text{delta\_backwards}}, i_{\min}) $$
    [S1]
    f. **Volume Evaluation (if not skipped):**
    If ESS is disabled or the region is occupied:
    1.  Get voxel value: `voxelValue = getVolumeIntensityValue(samplingPos.xyz + 0.5/volumeSize, volumeSize, 1.0/volumeSize, false)` [S1].
    2.  Transfer Function: `color = texture(texITF, vec2(voxelValue + 0.5/dimITF[0], 0.5/dimITF[1])).rgba` [S1].
    3.  Gamma Correction: `color.a = pow(color.a, gamma)` if enabled [S1].
    4.  Gradient/Lighting: Compute normal from gradient, apply diffuse/specular lighting [S1].
    5.  Composite:
        $$ \text{color}.a = \text{clamp}(1.0 \times (1.0 - (1.0 - \text{color}.a)^{\text{sampling\_factor\_inv}}), 0.0, 1.0) $$
        $$ \text{color}.xyz \times= \text{color}.a $$
        $$ \text{out\_color} \leftarrow \text{out\_color} + (1.0 - \text{out\_color}.a) \times \text{color} $$

5.  **Early Ray Termination (ERT):**
    If `out_color.a > 0.98` and ERT is enabled, set `out_color.a = 1.0` and break loop [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def step(edge, x):
    """GLSL step: 0.0 if x < edge, else 1.0"""
    return 1.0 if x >= edge else 0.0

def sign(x):
    """GLSL sign: 1.0 if x > 0, -1.0 if x < 0, else 0.0"""
    if x > 0: return 1.0
    if x < 0: return -1.0
    return 0.0

def clamp(val, lo, hi):
    return max(lo, min(val, hi))

def fetch_3d_texture(tex, coords):
    """Simulates GLSL texelFetch(tex, ivec3, 0).x for a 3D list of lists of lists."""
    x, y, z = coords
    x = clamp(x, 0, len(tex) - 1)
    y = clamp(y, 0, len(tex[0]) - 1)
    z = clamp(z, 0, len(tex[0][0]) - 1)
    return tex[x][y][z]

def vec3_add(a, b): return [a[k] + b[k] for k in range(3)]
def vec3_sub(a, b): return [a[k] - b[k] for k in range(3)]
def vec3_mul(a, b): return [a[k] * b[k] for k in range(3)]
def vec3_div(a, b): return [a[k] / b[k] for k in range(3)]
def vec3_scale(a, s): return [a[k] * s for k in range(3)]
def vec3_len(a): return math.sqrt(sum(x*x for x in a))
def vec3_norm(a): l = vec3_len(a); return [x/l for x in a]
def vec3_dot(a, b): return sum(a[k]*b[k] for k in range(3))
def vec3_min(a, b): return [min(a[k], b[k]) for k in range(3)]
def vec3_max(a, b): return [max(a[k], b[k]) for k in range(3)]
def vec3_floor(a): return [math.floor(x) for x in a]
def vec3_ceil(a): return [math.ceil(x) for x in a]
def vec3_step(edge, x): return [step(edge, x[k]) for k in range(3)]
def vec3_sign(x): return [sign(x[k]) for k in range(3)]

def empty_space_skipping_ray_march(ray_origin, ray_dir, volume_size, dim_distance_map_xyz,
                                   tex_occupancy, tex_distance, ess_type, sampling_factor,
                                   alpha_thre, early_stop_eps, dithering_mode, gl_frag_coord):
    ray_length = vec3_len(ray_dir)
    # Handle degenerate rays
    if math.isnan(ray_length) or ray_length > math.sqrt(3):
        return [0.0, 0.0, 0.0, 0.0]

    n_steps = math.ceil(volume_size * ray_length * sampling_factor)
    if n_steps < 2: n_steps = 2
    step_vector = vec3_scale(vec3_norm(ray_dir), ray_length / (n_steps - 1))

    block_size = vec3_ceil([volume_size / dim_distance_map_xyz[k] for k in range(3)])
    volume_to_dm = vec3_div([volume_size, volume_size, volume_size], vec3_mul(block_size, dim_distance_map_xyz))
    step_dist_texel = vec3_div(vec3_mul(step_vector, [volume_size, volume_size, volume_size]), block_size)
    step_dist_texel_inv = vec3_div([1.0, 1.0, 1.0], step_dist_texel)

    if dithering_mode == 1:
        random_val = math.fmod(abs(math.sin(gl_frag_coord[0] * 12.9898 + gl_frag_coord[1] * 78.233) * 43758.5453), 1.0)
    elif dithering_mode == 2:
        random_val = fetch_3d_texture(tex_distance, [int(gl_frag_coord[0] / 256), int(gl_frag_coord[1] / 256), 0])
    else:
        random_val = 0.0

    out_color = [0.0, 0.0, 0.0, 0.0]
    i = 0
    i_min = 0

    while i < n_steps:
        sampling_pos = vec3_add(ray_origin, vec3_scale(step_vector, i + random_val))
        pos_dm = vec3_mul(volume_to_dm, sampling_pos)
        u = vec3_mul(pos_dm, dim_distance_map_xyz)
        u_i = vec3_max(vec3_min(vec3_floor(u), [0, 0, 0]), [dim_distance_map_xyz[k] - 1 for k in range(3)])

        if ess_type == 1:
            dist = fetch_3d_texture(tex_occupancy, u_i)
        else:
            dist = fetch_3d_texture(tex_distance, u_i)

        r = vec3_max(vec3_min(vec3_sub([float(x) for x in u_i], u), [-1.0, -1.0, -1.0]), [0.0, 0.0, 0.0])

        if ess_type == 1:
            i_delta_xyz = vec3_mul(vec3_add(vec3_step(0.0, step_dist_texel_inv), r), step_dist_texel_inv)
        else:
            i_delta_xyz = vec3_mul(vec3_add(vec3_add(vec3_step(0.0, vec3_scale(step_dist_texel_inv, -1.0)), vec3_scale(vec3_sign(step_dist_texel_inv), dist)), r), step_dist_texel_inv)

        min_delta = min(i_delta_xyz[0], i_delta_xyz[1], i_delta_xyz[2])
        i_delta = max(1, math.ceil(min_delta))
        i += i_delta

        if dist == 0:
            i_delta_backwards = -math.ceil(sampling_factor)
            i = max(i + i_delta_backwards, i_min)
            continue

        # Volume evaluation
        voxel_val = fetch_3d_texture(tex_occupancy, u_i) # Simplified from notes
        color = [voxel_val, voxel_val, voxel_val, 1.0]

        # Gradient/Lighting (from notes)
        normal = vec3_scale([-1.0, -1.0, -1.0], 1.0/math.sqrt(3)) # Placeholder gradient [G]
        light = vec3_norm([0.0, 0.0, 1.0]) # Placeholder light [G]
        diffuse = max(0.0, vec3_dot(normal, light))
        color[0] = color[0] * (1.0 + diffuse)
        color[1] = color[1] * (1.0 + diffuse)
        color[2] = color[2] * (1.0 + diffuse)

        sampling_factor_inv = 1.0 / sampling_factor
        color[3] = clamp(1.0 * (1.0 - (1.0 - color[3])**sampling_factor_inv), 0.0, 1.0)
        color[0] *= color[3]
        color[1] *= color[3]
        color[2] *= color[3]
        out_color[0] = out_color[0] + (1.0 - out_color[3]) * color[0]
        out_color[1] = out_color[1] + (1.0 - out_color[3]) * color[1]
        out_color[2] = out_color[2] + (1.0 - out_color[3]) * color[2]
        out_color[3] = out_color[3] + (1.0 - out_color[3]) * color[3]

        if out_color[3] > 0.98:
            out_color[3] = 1.0
            break

    return out_color
```

## Subtopics

### [Ray Marcher with Macrocell Empty Space Skipping](1009-ray-marcher-with-macrocell-empty-space-skipping.md)
Uses precomputed macrocell densities to skip empty space. Chosen when coarse elements have zero density.

### [Empty Space Skipping with Orthogonal BSP Tree](1054-empty-space-skipping-with-orthogonal-bsp-tree.md)
Partitions volume into sub-volumes in an orthogonal BSP tree for visibility sorting. Chosen for texture-based rendering with complex transfer functions.

## Operating characteristics
**Time Complexity:** Proportional to the number of *occupied* samples rather than total volume depth [S2].
**Memory Complexity:** Requires auxiliary storage for the occupancy grid or distance map [S1, S2].
**Performance:** Can prune up to 98% of samples in dense surface scenes (e.g., Lego scene in NeRF-Synthetic) [S2].
**Strengths:** Reduces GPU memory usage by parallelizing across rays rather than over-generating samples [S2]. Prevents performance regressions from degenerate rays [S1].
**Limitations:** Requires precomputation or online caching of the occupancy/distance map [S1, S2]. Skipping occluded regions requires precise density evaluation to compute transmittance accurately [S2]. Parallel efficiency is slightly lower than sample-based parallelization because operations are grouped by ray [S2].
**When to use:** Sparse volumes, NeRF training/inference where memory is constrained, or when empty regions are predictable [S1, S2].
**When not to use:** Not given in the sources.

## Background
Empty-space skipping was established as an acceleration technique for direct volume rendering to avoid computing transparent regions [S3]. Early methods used hierarchical representations or distance fields to adapt step sizes [Zuiderveld et al. 92]. Isosurfacing was also used to find first hit locations [Avila et al. 92]. In the context of Neural Radiance Fields (NeRF), the technique was adapted using Occupancy Grids (from Instant-NGP) to cache empty regions during training, enabling efficient pruning [S2].

## Sources
[S1] CalamityMichL/DVR_ESS_PDM, https://github.com/CalamityMichL/DVR_ESS_PDM, fetched 2026-10-26 (unofficial)
[S2] NerfAcc: A General NeRF Acceleration Toolbox, https://doi.org/10.48550/arxiv.2210.04847, fetched 2026-10-26
[S3] PDF Direct Volume Rendering, https://www.sci.utah.edu/~stevec/classes/cs5630/dvr.pdf, fetched 2026-10-27
