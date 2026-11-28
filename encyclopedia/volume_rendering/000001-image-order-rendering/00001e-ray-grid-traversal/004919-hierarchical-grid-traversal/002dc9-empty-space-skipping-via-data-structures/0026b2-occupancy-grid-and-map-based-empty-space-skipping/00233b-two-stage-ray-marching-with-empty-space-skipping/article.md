# Two-stage Ray Marching with Empty-Space Skipping

Also called: Empty Space Compactness, Occupancy Map Empty Space Skipping, Occupancy-based early exit, Simple occupancy map traversal, Two-phase volumetric rendering

## Overview
Two-stage Ray Marching with Empty-Space Skipping is a rendering optimization that accelerates ray traversal in sparse volumetric data by bypassing empty regions using a precomputed occupancy map [S2, unofficial]. It splits ray traversal into two distinct phases: Stage 1 performs cheap lookups in a lower-resolution occupancy grid to locate the entry point into active volume blocks, and Stage 2 performs expensive density sampling only within those active segments [S2, unofficial]. This method sets itself apart from its siblings by decoupling the traversal logic into a coarse "find entry" phase and a fine "integrate" phase, relying on a flat or sparse occupancy grid rather than hierarchical structures like octrees [S2, unofficial]. It builds on the shared principles of [Occupancy Grid and Map-Based Empty Space Skipping](3852-occupancy-grid-and-map-based-empty-space-skipping.md) by utilizing a secondary acceleration structure to define active regions and skip empty space [S1][S2, unofficial].

## How it works
The method operates by first constructing an occupancy map that indicates which blocks of the primary volume contain data [S1]. During rendering, rays traverse this map to skip empty space, then switch to high-resolution sampling upon entering an occupied block [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Primary volume data (density/opacity field) at resolution $N_{vol} \times N_{vol} \times N_{vol}$ [S2, unofficial].
    *   Occupancy map at resolution $N_{occ} \times N_{occ} \times N_{occ}$, where $N_{occ} = N_{vol} / B$ and $B$ is the block size factor [S1].
    *   Camera parameters: origin, direction, projection matrix, and world-to-camera transform [S2, unofficial].
    *   Volume bounds: $\mathbf{B}_{extent}$ (half-extents) and voxel resolution $V_{res}$ [S2, unofficial].
    *   Parameters: `_RayMaxLength`, `_DensityMult`, `_Smoothing`, `_SmokeOrigin`, `_SmokeRadius` [S2, unofficial].
*   **Outputs:**
    *   Final color buffer (image) with accumulated opacity and color [S2, unofficial].

### Data Structures
1.  **Primary Volume Buffer:** A 3D texture or structured buffer storing density/opacity values [S2, unofficial].
2.  **Occupancy Map:** A 3D texture or structured buffer storing binary or scalar occupancy status [S2, unofficial].
    *   **Layout:** Flattened 1D buffer index for voxel $(x, y, z)$ in row-major order:
        $$ \text{index} = z \cdot (N_{vol,x} \cdot N_{vol,y}) + y \cdot N_{vol,x} + x $$
        [S2, unofficial]
    *   **Values:** Typically `0` (empty) or `1` (filled/active). Out-of-bounds queries return `-1` [S2, unofficial].
3.  **Ray State:**
    *   `origin`: Camera position in world space.
    *   `rayVector`: Normalized direction vector.
    *   `length`: Current distance along the ray.
    *   `smokeValue`: Current sampled value (used to track stage state: `-1` for Stage 1, `1` for Stage 2) [S2, unofficial].

### Numbered Steps

#### 1. Occupancy Map Construction (Pre-render)
A compute shader generates the occupancy map [S1]. Each shader invocation evaluates a unique volume block for occupancy and writes the result to the associated occupancy map voxel [S1].
*   For each block in the occupancy grid, sample the corresponding region in the primary volume.
*   Determine occupancy based on a threshold (e.g., max density > 0 or opacity > $\epsilon$) [G].
*   Write the result to the occupancy map voxel [S1].

#### 2. Ray Initialization
For each pixel $(u, v)$:
1.  Compute ray direction in world space:
    $$ \mathbf{d}_{clip} = (u \cdot 2 - 1, v \cdot 2 - 1, 0) $$
    $$ \mathbf{d}_{cam} = \text{mul}(\mathbf{M}_{invProj}, \mathbf{d}_{clip}) $$
    $$ \mathbf{rayVector} = \text{normalize}(\text{mul}(\mathbf{M}_{camToWorld}, \mathbf{d}_{cam})) $$
    [S2, unofficial]
2.  Set `origin` to camera position.
3.  Initialize `length = 0`, `stepSize = \Delta t_{vol}` (volume voxel step), `smokeValue = -1` (indicating Stage 1 active) [S2, unofficial].

#### 3. Stage 1: Empty Space Skipping (Occupancy Lookup)
Loop until an occupied block is found or the ray exits the volume:
1.  Calculate current world position:
    $$ \mathbf{p} = \mathbf{origin} + \text{length} \cdot \mathbf{rayVector} $$
    [S2, unofficial]
2.  Check volume bounds. If $\mathbf{p}$ is outside $\mathbf{B}_{extent}$, return black color (miss) [S2, unofficial].
3.  Query occupancy map:
    *   Convert $\mathbf{p}$ to occupancy grid coordinates:
        $$ \mathbf{u}_{occ} = \lfloor \mathbf{p} / (B \cdot V_{res}) \rfloor $$
        [G] (Derived from parent principles and S1 Eq. 10)
    *   Sample `smokeValue = getSmokeVoxel(p)` [S2, unofficial].
4.  **Skip Logic:**
    *   If `smokeValue == 0` (empty block):
        *   Calculate steps to next occupancy voxel boundary. The step count $\Delta i$ in volume voxels is derived from the distance to the next occupancy grid cell boundary [S1].
        *   Using the vectorized step calculation [S1]:
            $$ \Delta \mathbf{i} = \lceil (\text{step}(\Delta \mathbf{u}) + \lfloor \mathbf{u} \rfloor - \mathbf{u}) / \Delta \mathbf{u} \rceil $$
            where $\Delta \mathbf{u}$ is the ray step in occupancy space:
            $$ \Delta \mathbf{u}_{\text{occ}} = \frac{d_{\text{vol}}}{B} \Delta \mathbf{t}_{\text{vol}} $$
            [S1]
        *   The scalar step to advance is:
            $$ \Delta i = \max(\min(\Delta \mathbf{i}), 1) $$
            [S1]
        *   Update `length += \Delta i \cdot \Delta t_{vol}`.
    *   If `smokeValue == 1` (occupied block):
        *   **Backward Adjustment:** To account for interpolation halos or boundary isosurfaces, decrement the step count to ensure the ray goes back at least one voxel length:
            $$ \Delta i_{\text{backward}} = -\lfloor f \rfloor $$
            where $f$ is the sampling factor [S1].
            $$ \text{length} += \Delta i_{\text{backward}} \cdot \Delta t_{vol} $$
            [S1]
        *   Switch to Stage 2.

#### 4. Stage 2: Density Sampling
Perform high-resolution integration within the occupied block.
1.  Initialize `stage2StepNum = 0`, `accumulatedDensity = 0`, `stage2StepMax = 40` (or until ray exit) [S2, unofficial].
2.  Loop while `stage2StepNum < stage2StepMax`:
    *   Calculate `samplePoint = origin + length * rayVector`.
    *   Compute density:
        $$ \text{density} = 1 - \text{smoothstep}(\text{_Smoothing}, 1.0, \frac{\|\mathbf{p} - \mathbf{S}_{origin}\|}{\mathbf{S}_{radius}}) $$
        [S2, unofficial]
        *(Note: Source [S2, unofficial] uses screen UVs for noise, which is a known pitfall; correct implementation uses world position)* [S2, unofficial].
    *   Accumulate: `accumulatedDensity += density * _DensityMult`.
    *   Update `length += stepSize`.
    *   Increment `stage2StepNum`.
3.  **Resume Skipping:** After Stage 2, check if the ray is still within an occupied block. If the occupancy map coordinate $u_{occ}$ changes and the previous voxel had zero opacity, resume Stage 1 skipping [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_stage_ray_march(origin, ray_vector, volume_texture, occupancy_map, params):
    """
    params:
        B: Block size factor (ratio of vol res to occ res)
        voxel_res: Resolution of primary volume voxels (world units per voxel)
        bounds_extent: Half-extents of volume bounds (x, y, z)
        ray_max_length: Maximum ray length
        density_mult: Scaling factor for density
        smoothing: Smoothstep parameter
        smoke_origin, smoke_radius: Sphere parameters for density
    """
    length = 0.0
    step_size = params.voxel_res
    smoke_value = -1  # -1: Stage 1 (skip), 1: Stage 2 (sample)
    stage2_steps = 0
    stage2_max = 40
    accumulated_density = 0.0
    
    # Precompute occupancy step delta
    # Delta u_occ is the ray step in occupancy grid units
    # Delta t_vol is the ray step in volume units (voxel_res)
    # From S1 Eq 10: Delta u_occ = (d_vol / B) * Delta t_vol. Assuming d_vol=1 for normalized coords or voxel_res scaling.
    # Geometrically, step in occupancy space = step in volume space / B.
    delta_u_occ = ray_vector / (params.B * params.voxel_res)
    
    while length < params.ray_max_length:
        # 1. Calculate current position
        pos = origin + length * ray_vector
        
        # 2. Bounds Check (Fixed from S2 asymmetric bug)
        if (abs(pos.x) > params.bounds_extent.x or 
            abs(pos.y) > params.bounds_extent.y or 
            abs(pos.z) > params.bounds_extent.z):
            return 0.0, 0.0, 0.0 # Miss
        
        # 3. Determine Stage
        if smoke_value <= 0:
            # Stage 1: Skip Empty Space
            # Query occupancy map
            occ_coord = [int(floor(pos.x / (params.B * params.voxel_res))),
                         int(floor(pos.y / (params.B * params.voxel_res))),
                         int(floor(pos.z / (params.B * params.voxel_res)))]
            smoke_value = occupancy_map.sample(occ_coord)
            
            if smoke_value == 0:
                # Empty block: Skip to next occupancy voxel
                # Current u in occupancy space
                u_occ = [pos.x / (params.B * params.voxel_res),
                         pos.y / (params.B * params.voxel_res),
                         pos.z / (params.B * params.voxel_res)]
                
                # Calculate steps to next integer boundary for each dimension using S1 Eq 8 & 9
                delta_i_vec = []
                for j in range(3):
                    if delta_u_occ[j] == 0:
                        delta_i_vec.append(float('inf'))
                    else:
                        # step function: 1 if delta_u >= 0 else 0
                        step_val = 1.0 if delta_u_occ[j] >= 0 else 0.0
                        numerator = step_val + floor(u_occ[j]) - u_occ[j]
                        delta_i_vec.append(ceil(numerator / delta_u_occ[j]))
                
                # Minimum step, clamped to 1
                delta_i = max(min(delta_i_vec), 1.0)
                
                # Advance length (delta_i is in volume voxel steps)
                length += delta_i * params.voxel_res
                
            elif smoke_value == 1:
                # Occupied block found
                # Backward adjustment to handle interpolation halos (S1 Eq 11)
                # f is sampling factor, typically 1.0 for standard marching
                delta_i_backward = -1 
                length += delta_i_backward * params.voxel_res
                
                # Switch to Stage 2
                smoke_value = 1
                stage2_steps = 0
                accumulated_density = 0.0
            else:
                # smoke_value == -1 (out of bounds), should not happen due to bounds check, but treat as miss
                return 0.0, 0.0, 0.0
                
        else:
            # Stage 2: Sample Density
            # Check if we should resume skipping (if we exited the block)
            occ_coord = [int(floor(pos.x / (params.B * params.voxel_res))),
                         int(floor(pos.y / (params.B * params.voxel_res))),
                         int(floor(pos.z / (params.B * params.voxel_res)))]
            current_occupancy = occupancy_map.sample(occ_coord)
            
            if current_occupancy == 0:
                # Exited occupied block, resume Stage 1
                smoke_value = -1
                continue
                
            # Perform sampling
            sample_point = pos
            dist_from_origin = sqrt(sum((sample_point[k] - params.smoke_origin[k])**2 for k in range(3)))
            normalized_dist = dist_from_origin / params.smoke_radius
            
            # Density calculation (Simplified from S2, ignoring screen-space noise pitfall)
            density = 1.0 - smoothstep(params.smoothing, 1.0, normalized_dist)
            
            accumulated_density += density * params.density_mult
            length += params.voxel_res
            stage2_steps += 1
            
            if stage2_steps >= params.stage2_max:
                # End of block or max steps
                smoke_value = -1 # Resume skipping
                continue

    return accumulated_density, 0.0, 0.0 # Simplified return

def smoothstep(edge0, edge1, x):
    t = max(0.0, min(1.0, (x - edge0) / (edge1 - edge0)))
    return t * t * (3.0 - 2.0 * t)
```

### Parameters and Typical Values
*   **B (Block Size):** Ratio of volume resolution to occupancy map resolution. Larger $B$ skips more empty voxels per sample but reduces precision near occupied regions [S1]. Typical values depend on sparsity; e.g., $B=4$ or $B=8$ [S1].
*   **Step Size:** Usually equal to the primary volume voxel resolution [S2, unofficial].
*   **Stage 2 Max Steps:** Fixed limit (e.g., 40) or until ray exit [S2, unofficial].
*   **Smoothing:** Smoothstep edge parameter for density falloff [S2, unofficial].

### Edge Cases and Pitfalls
1.  **Stage 2 Bounds Ignorance:** The accumulation loop may run for a fixed number of steps regardless of whether the ray has exited the voxel grid, causing smoke to bleed past the volume boundary [S2, unofficial].
2.  **Asymmetric Bounds Check:** Incorrect bounds checking logic (e.g., `position.y < -_BoundsExtent.y + _BoundsExtent.y`) can anchor the volume incorrectly or cause out-of-bounds access [S2, unofficial].
3.  **Variable Shadowing:** Using `length` as a variable name can shadow intrinsic functions like `length()` in HLSL/GLSL [S2, unofficial].
4.  **Screen-Space Noise:** Sampling noise using screen UVs (`uv`) pins the noise pattern to the screen, causing it to "swim" as the camera moves. Correct implementation requires sampling noise using world position (`samplePoint`) [S2, unofficial].
5.  **Interpolation Halos:** Trilinear interpolation can cause a sampling point near the edge of an empty block to map to a non-zero opacity if a neighboring block is occupied. The backward adjustment step is necessary to prevent missing these boundary details [S1].

## Subtopics

### [PDF Light Weight Space Leaping](5848-pdf-light-weight-space-leaping.md)
An image-space method that uses detector rays and space leap rays to skip empty space without pre-processing or 3D data structures. Choose when memory is constrained or transfer functions change frequently.

### [Volume Occupancy Map Skip](5991-volume-occupancy-map-skip.md)
Builds on this method by using a preprocessed occupancy map to jump directly from one non-empty voxel to the next. Choose for sparse datasets where a flat occupancy grid offers a better trade-off between memory and traversal simplicity than hierarchical structures.

## Operating characteristics
*   **Time Complexity:** Stage 1 is $O(1)$ per occupancy voxel visited. Stage 2 is $O(N)$ where $N$ is the number of voxels in the active segment. Total time is proportional to the number of active voxels plus the number of empty blocks skipped [S3, unofficial].
*   **Memory Complexity:** $O(N_{occ}^3)$ for the occupancy map, where $N_{occ} = N_{vol} / B$. This is significantly smaller than the primary volume if $B > 1$ [S1].
*   **Performance:** Performance scales with the sparsity of the volume. Higher $B$ values improve performance on sparse volumes but may reduce accuracy near boundaries [S1].
*   **Strengths:** Significantly reduces the number of samples that need to be processed [S3, unofficial].
*   **Limitations:** Requires pre-computation of the occupancy map; less effective for dense volumes; fixed step size in Stage 2 may lead to over-sampling or bleeding [S2, unofficial].

## Background
The method originates from efficient ray casting techniques for volumetric images, specifically utilizing distance maps and occupancy grids for empty space skipping [S1]. It is a foundational technique in GPU-based volume rendering, often implemented using compute shaders to generate the occupancy map and ray marching kernels to perform the traversal [S1][S2, unofficial].

## Sources
[S1] Efficient ray casting of volumetric images using distance maps for empty space skipping, https://doi.org/10.1007/s41095-019-0155-y, fetched 2026-10-24
[S2] Schokolasbotter/volumetric-smoke-raymarching, https://github.com/Schokolasbotter/volumetric-smoke-raymarching, fetched 2026-10-28 (unofficial)
[S3] ChatGPT version of Ct Volume Rendering, https://www.flyriver.com/s/ct-volume-rendering, fetched 2026-10-29 (unofficial)
