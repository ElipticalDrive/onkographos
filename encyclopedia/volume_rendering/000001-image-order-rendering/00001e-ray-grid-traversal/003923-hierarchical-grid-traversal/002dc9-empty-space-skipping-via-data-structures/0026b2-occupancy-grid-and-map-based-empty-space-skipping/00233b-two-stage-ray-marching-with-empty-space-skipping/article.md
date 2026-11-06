# Two-stage Ray Marching with Empty-Space Skipping

Also called: Occupancy Map Empty Space Skipping, Occupancy-based early exit, Simple occupancy map traversal, Two-phase volumetric rendering

## Overview
Two-stage Ray Marching with Empty-Space Skipping is a rendering optimization that accelerates volumetric ray casting by decoupling the search for active volume data from the expensive density sampling process [S1][S2, unofficial]. It solves the problem of wasting computational resources sampling empty space by using a lower-resolution occupancy map to quickly identify entry points into active regions [S1][S2, unofficial]. Unlike its siblings, it employs a strict two-phase loop: Stage 1 performs cheap, coarse-grained occupancy lookups to skip inactive blocks, while Stage 2 performs fine-grained, expensive density sampling only within confirmed active regions [S2, unofficial]. This approach is distinct from hierarchical methods like [Hierarchical Sampling with Occupancy Grid](3035-hierarchical-sampling-with-occupancy-grid.md) by using a flat occupancy map and fixed-step skipping rather than recursive tree traversal, and from [Distance Map Acceleration](972-distance-map-acceleration.md) by relying on binary occupancy flags rather than distance-to-nearest-occupied-voxel metrics [S1][S2, unofficial].

## How it works
The method operates in two distinct phases per ray. It requires a precomputed occupancy map and a primary volume dataset.

### Inputs and Outputs
*   **Inputs:**
    *   Primary volume data (density/opacity field).
    *   Occupancy map: A lower-resolution grid where each voxel indicates if its corresponding volume block is active (non-zero density/opacity) or empty [S1][S2, unofficial].
    *   Camera parameters: Origin, direction, projection matrix, and world-to-camera matrices [S2, unofficial].
    *   Parameters:
        *   $B$: Block size factor, the ratio of volume resolution to occupancy map resolution (e.g., $B=4$) [S1].
        *   $\tau$: Threshold for defining "active" data (e.g., density $> \tau$) [S1].
        *   `_RayMaxLength`: Maximum traversal distance [S2, unofficial].
        *   `stepSize`: Step size for Stage 1 traversal [S2, unofficial].
        *   `stage2StepMax`: Maximum number of sampling steps in Stage 2 [S2, unofficial].
*   **Outputs:**
    *   Final accumulated color and opacity for each pixel [S2, unofficial].

### Data Structures
1.  **Occupancy Map:** A 3D grid of integers (or bits). Each element corresponds to a block of $B \times B \times B$ volume voxels.
    *   Value `1`: Block contains active data.
    *   Value `0`: Block is empty.
    *   Generation: Computed via a compute shader where each invocation evaluates a unique volume block for occupancy and writes the result [S1].
2.  **Volume Buffer:** Structured buffer or texture storing the primary density/opacity values [S2, unofficial].
3.  **Ray State:**
    *   `origin`: Camera position in world space.
    *   `rayVector`: Normalized direction vector.
    *   `length`: Current distance traveled along the ray.
    *   `smokeValue`: Current occupancy/density sample result.

### Numbered Steps

1.  **Ray Initialization:**
    *   Compute ray direction from UV coordinates using inverse projection and camera-to-world matrices [S2, unofficial]:
        $$ \mathbf{rayVector} = \text{normalize}(\text{mul}(_{\text{CamToWorldMatrix}}, \text{float4}(\text{mul}(_{\text{invProjectionMatrix}}, \text{float4}(uv.x \cdot 2 - 1, uv.y \cdot 2 - 1, 0, 1)).xyz, 0)).xyz) $$
    *   Initialize `length = 0`, `smokeValue = -1` (or 0) [S2, unofficial].

2.  **Stage 1: Occupancy Lookup (Empty Space Skipping):**
    *   Loop while `smokeValue <= 0` AND `length < _RayMaxLength`:
        *   Increment `length` by `stepSize` [S2, unofficial].
        *   Calculate world position: $\mathbf{position}_{WS} = \mathbf{origin} + \text{length} \cdot \mathbf{rayVector}$ [S2, unofficial].
        *   Query occupancy: `smokeValue = getSmokeVoxel(positionWS)` [S2, unofficial].
            *   `getSmokeVoxel` converts world position to voxel indices, checks bounds, and returns the occupancy map value (0 or 1) [S2, unofficial].
    *   **Exit Condition:** If `smokeValue <= 0` after exceeding `_RayMaxLength`, the ray misses the volume. Output black color and terminate [S2, unofficial].

3.  **Stage 2: Density Sampling (Active Region Traversal):**
    *   If `smokeValue == 1` (entry detected):
        *   Initialize `stage2StepNum = 0`, `accumulatedDensity = 0` [S2, unofficial].
        *   Loop while `stage2StepNum < stage2StepMax`:
            *   Calculate `samplePoint = origin + length * rayVector` [S2, unofficial].
            *   Compute density: `density = sampleDensity(samplePoint, uv)` [S2, unofficial].
                *   Note: `sampleDensity` may use noise textures sampled via world position or screen UVs (see Pitfalls) [S2, unofficial].
            *   Update accumulation: `accumulatedDensity += density * _DensityMult` [S2, unofficial].
            *   Increment `length` by `stepSize` [S2, unofficial].
            *   Increment `stage2StepNum` [S2, unofficial].
    *   **Resuming Skipping:** After Stage 2, if the ray is still within the volume bounds, check if the occupancy map coordinate $u_{occ}$ has changed and if the previously sampled voxel had zero opacity. If so, resume Stage 1 skipping [S1].

4.  **Backward Adjustment (Boundary Handling):**
    *   When an occupied region is found, decrement the ray step number by $\Delta i_{\text{backward}} = -\lfloor f \rfloor$ to ensure the ray goes back at least one voxel length, accounting for interpolation or boundary isosurfaces [S1]. Here, $f$ is the sampling factor [S1].

### Formulas

**Step Calculation in Occupancy Space:**
To determine how many steps to skip in the occupancy map, the algorithm calculates the distance to the next block boundary. For a ray parameter $\mathbf{u}$ and step $\Delta \mathbf{u}$, the number of steps $\Delta \mathbf{i}$ on each dimension $j \in \{x, y, z\}$ is:

$$ \Delta i_j = \begin{cases} \lceil (1 + \lfloor \mathbf{u}_j \rfloor - \mathbf{u}_j) / \Delta \mathbf{u}_j \rceil, & \text{if } \Delta \mathbf{u}_j > 0 \\ \lceil (\lfloor \mathbf{u}_j \rfloor - \mathbf{u}_j) / \Delta \mathbf{u}_j \rceil, & \text{otherwise} \end{cases} $$

This can be vectorized using the Heaviside step function:

$$ \Delta \mathbf{i} = \lceil (\text{step}(\Delta \mathbf{u}) + \lfloor \mathbf{u} \rfloor - \mathbf{u}) / \Delta \mathbf{u} \rceil $$

The final step count is the minimum of the dimensional steps, clamped to a minimum of 1:

$$ \Delta i = \max(\min(\Delta \mathbf{i}), 1) $$

**Occupancy Map Step Size:**
The distance step in occupancy map space $\Delta \mathbf{u}_{\text{occ}}$ is derived from the volume step $\Delta \mathbf{t}_{\text{vol}}$ and block size $B$:

$$ \Delta \mathbf{u}_{\text{occ}} = \frac{d_{\text{vol}}}{B} \Delta \mathbf{t}_{\text{vol}} $$

where $d_{\text{vol}}$ is the volume resolution [S1].

**Density Calculation (Example):**
$$ \text{density} = 1 - \text{smoothstep}(_{\text{Smoothing}}, 1.0, \text{normalizedDistance} + \text{noiseValue}) $$
where $\text{normalizedDistance} = \text{length}(\mathbf{pos} - _{\text{SmokeOrigin}}) / _{\text{SmokeRadius}}$ [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_stage_ray_marching(ray_origin, ray_direction, occupancy_map, volume_data, params):
    """
    ray_origin: float3
    ray_direction: float3 (normalized)
    occupancy_map: 3D array of ints (0 or 1)
    volume_data: 3D array of floats
    params: dict containing B, stepSize, RayMaxLength, stage2StepMax, etc.
    """
    length = 0.0
    smoke_value = 0  # 0 means empty/unfound
    
    # Stage 1: Find active region
    while smoke_value <= 0 and length < params['RayMaxLength']:
        length += params['stepSize']
        position_ws = ray_origin + length * ray_direction
        
        # Convert world position to occupancy map index
        # Assuming simple division by block size B for regular grids
        voxel_pos = position_ws / params['voxel_resolution']
        occ_index = voxel_pos / params['B']
        
        if is_within_bounds(occ_index, occupancy_map.shape):
            smoke_value = occupancy_map[occ_index]
        else:
            smoke_value = 0 # Out of bounds is empty
            
    # If no active region found within max length
    if smoke_value <= 0:
        return (0.0, 0.0, 0.0, 0.0) # Black, transparent

    # Stage 2: Sample density in active region
    stage2_step_num = 0
    accumulated_density = 0.0
    
    while stage2_step_num < params['stage2StepMax']:
        sample_point = ray_origin + length * ray_direction
        
        # Sample density from volume data
        density = sample_volume_density(sample_point, volume_data, params)
        
        accumulated_density += density * params['DensityMult']
        
        length += params['stepSize']
        stage2_step_num += 1
        
        # Optional: Check if we should resume skipping
        # If current position is in a new empty block, break to Stage 1
        # This requires checking occupancy_map again
        
    # Composite color based on accumulated_density
    color = composite_color(accumulated_density)
    return color

def sample_volume_density(pos, volume_data, params):
    # Convert world pos to voxel index
    voxel_pos = (pos + params['BoundsExtent']) / params['VoxelResolution']
    voxel_idx = floor(voxel_pos)
    
    if not is_within_bounds(voxel_idx, volume_data.shape):
        return 0.0
        
    # Trilinear interpolation or direct lookup
    val = volume_data[voxel_idx]
    
    # Apply transfer function or density calculation if needed
    # Example from [S2, unofficial]:
    # dist = length(pos - SmokeOrigin)
    # norm_dist = dist / SmokeRadius
    # val = 1 - smoothstep(Smoothing, 1.0, norm_dist)
    
    return val

def is_within_bounds(idx, shape):
    return all(0 <= i < s for i, s in zip(idx, shape))
```

### Edge Cases and Pitfalls
*   **Boundary Bleeding:** Stage 2 may ignore volume bounds if the loop runs for a fixed number of steps (`stage2StepMax`) regardless of whether the ray has exited the voxel grid, causing smoke to bleed past the volume boundary [S2, unofficial].
*   **Asymmetric Bounds Check:** Incorrect bounds checking logic (e.g., `position.y < -_BoundsExtent.y + _BoundsExtent.y`) can anchor the volume incorrectly at ground level or cause out-of-bounds access [S2, unofficial].
*   **Variable Shadowing:** Using variable names like `length` that shadow intrinsic functions (e.g., HLSL `length()`) can cause compilation errors or logic bugs [S2, unofficial].
*   **Noise Sampling:** Sampling noise using screen UVs (`uv`) pins the noise pattern to the screen, causing it to "swim" as the camera moves. Correct implementation requires sampling noise using world position (`samplePoint`) to attach detail to the volume [S2, unofficial].
*   **Interpolation Halo:** Trilinear interpolation of the volume can cause a sampling point near the edge of an empty block to map to a non-zero opacity if a neighboring block is occupied. A backward step adjustment ($\Delta i_{\text{backward}}$) is required to re-sample the boundary correctly [S1].

## Subtopics

### [PDF Light Weight Space Leaping](5848-pdf-light-weight-space-leaping.md)
PDF Light Weight Space Leaping (LWSL) is a space-leaping acceleration technique for volume rendering that exploits ray coherence to skip empty space without requiring pre-processing or 3D data structures. It divides image-plane rays into two groups: **detector rays**, which perform full ray casting to determine the distance to the first non-empty cell, and **space leap rays**, which are accelerated by skipping empty space based on information propagated from detector rays. Unlike data-structure-based methods (e.g., DFB) or temporal coherence techniques, LWSL is an image-space method with memory and run-time complexity proportional only to the image size, making it independent of dataset complexity and transfer function changes.

### Volume Occupancy Map Skip
Volume Occupancy Map Skip is a technique to skip empty voxels (e.g., sparse clouds) by using a preprocessed Volume Occupancy Map. If the current voxel is detected as empty, traversal jumps directly to the next non-empty voxel [S3, unofficial]. In a medical imaging project, this technique increased rendering frame rate by 3.2x [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Stage 1 is $O(k)$ where $k$ is the number of empty blocks skipped. Stage 2 is $O(m)$ where $m$ is the number of samples in the active region. Overall complexity depends on the sparsity of the volume. For sparse volumes, it is significantly faster than brute-force marching [S3, unofficial].
*   **Memory Complexity:** Requires storage for the primary volume data and the occupancy map. The occupancy map size is $1/B^3$ of the volume data size [S1].
*   **Measured Performance:** In a medical imaging project, using a Volume Occupancy Map for skipping increased rendering frame rate by 3.2x [S3, unofficial].
*   **Strengths:** Simple to implement; effective for sparse volumes; low memory overhead for the acceleration structure [S1][S3, unofficial].
*   **Limitations:** Precision loss near occupied regions if $B$ is too large; fixed step sizes may miss thin structures; requires precomputation of the occupancy map [S1].
*   **When to Use:** Sparse volumetric data (e.g., clouds, smoke, medical CT with large empty regions) [S1][S3, unofficial].
*   **When Not to Use:** Dense volumes where most rays hit active data; dynamic volumes where occupancy map updates are too expensive [S1].

## Background
The concept of using occupancy maps for empty space skipping in volume rendering was introduced in the paper "Efficient ray casting of volumetric images using distance maps for empty space skipping" [S1]. The method leverages the observation that large portions of volumetric datasets are empty, and skipping these regions can significantly reduce rendering time. The two-stage approach (search then sample) is a common optimization pattern in graphics, adapted here for volumetric data using a flat occupancy grid [S1][S2, unofficial]. The technique is widely used in real-time rendering of atmospheric effects and medical visualization [S1][S3, unofficial].

## Sources
[S1] Efficient ray casting of volumetric images using distance maps for empty space skipping, https://doi.org/10.1007/s41095-019-0155-y, fetched 2026-10-24
[S2] Schokolasbotter/volumetric-smoke-raymarching, https://github.com/Schokolasbotter/volumetric-smoke-raymarching, fetched 2026-10-28 (unofficial)
[S3] 体渲染中的体素遍历技术:从理论到实践-百度开发者中心, https://developer.baidu.com/article/detail.html?id=6975905, fetched 2026-10-25 (unofficial)
