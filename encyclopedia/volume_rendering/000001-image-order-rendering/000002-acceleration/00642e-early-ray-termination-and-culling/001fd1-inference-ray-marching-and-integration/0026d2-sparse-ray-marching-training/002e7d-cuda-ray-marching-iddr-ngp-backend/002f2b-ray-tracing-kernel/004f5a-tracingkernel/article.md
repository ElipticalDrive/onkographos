# TracingKernel


## Overview
`TracingKernel` is a CUDA compute shader that performs volumetric light transport by stepping through a 3D grid, dynamically adjusting step size and cascade resolution based on distance from the center, and accumulating transmittance (alpha) [S1, unofficial]. It is designed for the IDDR-NGP backend, where it serves as the core inference step for sparse ray marching, optimizing performance by coarser sampling in distant regions and finer sampling near the center [S1, unofficial]. It differs from standard uniform ray marching by using a logarithmic expansion factor to vary step sizes and cascade levels, and from dense voxel cone tracing by operating on a sparse grid structure with specific boundary handling [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** `id` (`uint3`), representing the thread's position in the output grid, which also defines the starting X, Y, and initial Z coordinates for a Z-axis ray march [S1, unofficial].
- **Global Constants:** `GridResolution` (`uint`, side length of the cubic grid), `GridHalfResolution` (`float`, half of `GridResolution`), `GridCascadeBase` (`uint2`, `{base_level, max_level}`), `CVCT_tex3d` (3D texture/array of pre-processed voxel values), `LightTex3d` (output texture) [S1, unofficial].
- **Output:** `LightTex3d[id]` (`float`), the accumulated transmittance (alpha) along the ray [S1, unofficial].

**Data Structures:**
- `CVCT_tex3d`: A 3D texture storing pre-processed voxel values. The source preprocessing (`RepackAndUpscaleKernel`) computes `CVCT_tex3d[id] = 1 - CVCT_gv[src_index]`, where `src_index = (z * GridResolution + y) * GridResolution + x` [S1, unofficial].
- `LightTex3d`: A 3D texture storing the final alpha values [S1, unofficial].

**Helper Function: SampleGV**
`SampleGV(p)` performs trilinear interpolation on `CVCT_tex3d` at position `p` [S1, unofficial]. The exact mapping of `CVCT_tex3d` to physical density is handled by the preprocessing step, but `TracingKernel` treats the sampled value `a` as the per-unit-step transmittance factor [S1, unofficial].

**Algorithm Steps:**
1. **Coordinate Transformation:** Convert the thread ID `id` to a local coordinate system centered at the grid origin [S1, unofficial].
   $$ \text{pos} = \text{id} - (\text{GridHalfResolution} - 0.5) $$
2. **Distance Calculation:** Compute the Chebyshev distance (max norm) from the center [S1, unofficial].
   $$ \text{central\_pos} = |\text{pos}| $$
   $$ \text{central\_distance} = \max(\text{central\_pos}.x, \max(\text{central\_pos}.y, \text{central\_pos}.z)) $$
3. **Normalization and Expansion Factor:** Normalize the distance to the range $[0, 1]$ and compute an initial expansion factor [S1, unofficial].
   $$ \text{max\_distance} = \text{GridHalfResolution} - 0.5 $$
   $$ \text{central\_distance} \leftarrow \text{central\_distance} / \text{max\_distance} $$
   $$ \text{expand} = \begin{cases} 1.0 & \text{if } \text{central\_distance} = 0 \\ \max(\log_2(\text{central\_distance}) + 2, 1) & \text{otherwise} \end{cases} $$
   This maps the center (`central_distance` $\approx 0$) to `expand` = 1 (finest resolution) and the boundary (`central_distance` $\approx 1$) to `expand` = 2 (coarsest resolution) [S1, unofficial].
4. **Initialization:**
   $$ \alpha = 1.0 $$
   $$ \text{grid\_cascade\_base} = \text{GridCascadeBase}.x $$
5. **Ray Marching Loop:**
   While true:
   a. **Cascade Check:** If `expand >= 2.0`, check if the current cascade level is at the maximum [S1, unofficial]. If `grid_cascade_base >= GridCascadeBase.y`, break the loop [S1, unofficial]. Otherwise, advance to the next cascade level:
      $$ \text{grid\_cascade\_base} \leftarrow \text{grid\_cascade\_base} + \text{GridResolution} $$
      $$ \text{expand} \leftarrow \text{expand} \times 0.5 $$
      $$ \text{pos} \leftarrow \text{pos} \times 0.5 $$
   b. **Sample Low Position:** Calculate the sample position in the current cascade [S1, unofficial].
      $$ \text{p} = \text{pos} + (\text{GridHalfResolution} - 0.5) $$
      $$ \text{p}.y \leftarrow \text{p}.y + \text{grid\_cascade\_base} $$
      $$ a_{\text{low}} = \text{SampleGV}(\text{p}) $$
   c. **Sample High Position:** Calculate the sample position for the next cascade (or boundary) [S1, unofficial].
      If `grid_cascade_base >= GridCascadeBase.y`:
      $$ a_{\text{high}} = 1.0 $$
      Else:
      $$ \text{q} = \text{pos} \times 0.5 + (\text{GridHalfResolution} - 0.5) $$
      $$ \text{q}.y \leftarrow \text{q}.y + \text{grid\_cascade\_base} + \text{GridResolution} $$
      $$ a_{\text{high}} = \text{SampleGV}(\text{q}) $$
   d. **Interpolate Alpha:** Blend between the low and high cascade samples based on the expansion factor [S1, unofficial].
      $$ a = \text{lerp}(a_{\text{low}}, a_{\text{high}}, \text{expand} - 1) $$
   e. **Update Step Size and Accumulate:**
      $$ \text{sample\_step} = \text{SAMPLE\_STEP\_BASE} \times \text{expand} $$
      $$ \alpha \leftarrow \alpha \times a^{\text{sample\_step}} $$
      $$ \text{pos}.z \leftarrow \text{pos}.z - \text{sample\_step} $$
      $$ \text{expand} \leftarrow \text{expand} \times \text{EXPAND\_STEP} $$
   f. **Termination:** The loop continues until `expand >= 2.0` and `grid_cascade_base` reaches the limit [S1, unofficial].
6. **Write Result:**
   $$ \text{id}.y \leftarrow \text{id}.y + \text{GridCascadeBase}.x $$
   $$ \text{LightTex3d}[\text{id}] = \alpha $$

**Parameters:**
- `SAMPLE_STEP_BASE`: Base step size for tracing. Typical value: `0.45` [S1, unofficial].
- `EXPAND_STEP`: Multiplier for the expansion factor per step. Typical value: `1.21` [S1, unofficial].

**Pitfalls and Edge Cases:**
- **Infinite Loop:** The loop is guaranteed to terminate because `expand` increases by `EXPAND_STEP` (1.21) each iteration, eventually exceeding 2.0, at which point the cascade base is incremented [S1, unofficial]. If the cascade base is already at `GridCascadeBase.y`, the loop breaks immediately upon the first check [S1, unofficial].
- **Boundary Conditions:** When `grid_cascade_base` reaches `GridCascadeBase.y`, `a_high` is clamped to 1. This assumes that beyond the highest cascade level, the volume is empty or fully transmissive [S1, unofficial].
- **Precision:** The use of `log2` and floating-point arithmetic for `expand` may introduce small precision errors, but the integer-based cascade base ensures robust level switching [S1, unofficial].
- **Coordinate System:** The `pos` variable is modified in place (scaled and shifted), which requires careful tracking of the coordinate system relative to the grid origin [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def TracingKernel(id: tuple, GridResolution: int, GridHalfResolution: float, 
                  GridCascadeBase: tuple, CVCT_tex3d: list, LightTex3d: list):
    # Helper for trilinear interpolation on a 3D array
    def SampleGV(p: tuple) -> float:
        # Clamp coordinates to valid range [0, GridResolution-1]
        x, y, z = p
        x = max(0, min(GridResolution - 1, x))
        y = max(0, min(GridResolution - 1, y))
        z = max(0, min(GridResolution - 1, z))
        
        # Get integer and fractional parts
        ix, iy, iz = int(x), int(y), int(z)
        fx, fy, fz = x - ix, y - iy, z - iz
        
        # Fetch 8 neighboring voxels
        v000 = CVCT_tex3d[ix, iy, iz]
        v100 = CVCT_tex3d[ix+1, iy, iz] if ix+1 < GridResolution else v000
        v010 = CVCT_tex3d[ix, iy+1, iz] if iy+1 < GridResolution else v000
        v110 = CVCT_tex3d[ix+1, iy+1, iz] if (ix+1 < GridResolution and iy+1 < GridResolution) else v010
        v001 = CVCT_tex3d[ix, iy, iz+1] if iz+1 < GridResolution else v000
        v101 = CVCT_tex3d[ix+1, iy, iz+1] if (ix+1 < GridResolution and iz+1 < GridResolution) else v001
        v011 = CVCT_tex3d[ix, iy+1, iz+1] if (iy+1 < GridResolution and iz+1 < GridResolution) else v001
        v111 = CVCT_tex3d[ix+1, iy+1, iz+1] if (ix+1 < GridResolution and iy+1 < GridResolution and iz+1 < GridResolution) else v011
        
        # Trilinear interpolation
        c0 = v000 + fx * (v100 - v000)
        c1 = v010 + fx * (v110 - v010)
        c01 = c0 + fy * (c1 - c0)
        
        c2 = v001 + fx * (v101 - v001)
        c3 = v011 + fx * (v111 - v011)
        c011 = c2 + fy * (c3 - c2)
        
        return c01 + fz * (c011 - c01)

    pos = (id[0] - (GridHalfResolution - 0.5), 
           id[1] - (GridHalfResolution - 0.5), 
           id[2] - (GridHalfResolution - 0.5))
    
    central_pos = (abs(pos[0]), abs(pos[1]), abs(pos[2]))
    central_distance = max(central_pos)
    
    max_distance = GridHalfResolution - 0.5
    central_distance /= max_distance
    
    if central_distance == 0:
        expand = 1.0
    else:
        expand = max(math.log2(central_distance) + 2, 1.0)
    
    alpha = 1.0
    grid_cascade_base = GridCascadeBase[0]
    
    SAMPLE_STEP_BASE = 0.45
    EXPAND_STEP = 1.21
    
    while True:
        if expand >= 2.0:
            if grid_cascade_base >= GridCascadeBase[1]:
                break
            grid_cascade_base += GridResolution
            expand *= 0.5
            pos = (pos[0] * 0.5, pos[1] * 0.5, pos[2] * 0.5)
            
        p = (pos[0] + (GridHalfResolution - 0.5), 
             pos[1] + (GridHalfResolution - 0.5), 
             pos[2] + (GridHalfResolution - 0.5))
        p = (p[0], p[1] + grid_cascade_base, p[2])
        a_low = SampleGV(p)
        
        if grid_cascade_base >= GridCascadeBase[1]:
            a_high = 1.0
        else:
            q = (pos[0] * 0.5 + (GridHalfResolution - 0.5), 
                 pos[1] * 0.5 + (GridHalfResolution - 0.5), 
                 pos[2] * 0.5 + (GridHalfResolution - 0.5))
            q = (q[0], q[1] + grid_cascade_base + GridResolution, q[2])
            a_high = SampleGV(q)
            
        a = a_low + (a_high - a_low) * (expand - 1.0)
        
        sample_step = SAMPLE_STEP_BASE * expand
        alpha *= a ** sample_step
        pos = (pos[0], pos[1], pos[2] - sample_step)
        expand *= EXPAND_STEP
        
    id = (id[0], id[1] + GridCascadeBase[0], id[2])
    LightTex3d[id] = alpha
```

## Operating characteristics
- **Time Complexity:** The number of steps per ray depends on the distance from the center and the `EXPAND_STEP` [S1, unofficial]. Rays near the center take more steps with smaller step sizes, while rays near the boundary take fewer steps with larger step sizes [S1, unofficial].
- **Memory Complexity:** The kernel uses constant memory for global parameters and local registers for state [S1, unofficial]. It reads from `CVCT_tex3d` and writes to `LightTex3d` [S1, unofficial].
- **Performance:** The dynamic step size and cascade switching allow for efficient rendering of large volumes by reducing the number of samples in empty or distant regions [S1, unofficial]. The use of CUDA compute shaders enables parallel execution across the entire grid [S1, unofficial].
- **Strengths:** Adaptive resolution based on distance, efficient use of sparse grid structures, simple implementation [S1, unofficial].
- **Limitations:** The fixed expansion factor and step size parameters may not be optimal for all volume types [S1, unofficial]. The assumption of `a_high = 1` at the boundary may introduce artifacts if the volume extends beyond the grid [S1, unofficial].

## Background
`TracingKernel` is part of the Cascaded Voxel Cone Tracing (CVCT) technique, specifically implemented in the `arigo/CascatedVoxelConeTracing` repository [S1, unofficial]. It is designed for real-time volumetric rendering, particularly in the context of neural graphics primitives (NGP) and sparse grid representations [S1, unofficial]. The method was developed to address the computational cost of dense ray marching by leveraging the sparsity of the grid and the hierarchical nature of the data [S1, unofficial].

## Sources
[S1] arigo/CascatedVoxelConeTracing, https://github.com/arigo/CascatedVoxelConeTracing, fetched 2026-10-26 (unofficial)
