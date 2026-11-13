# TracingKernel

Also called: CVCT Tracing Kernel, Cascaded Voxel Cone Tracing Tracing Kernel

## Overview
`TracingKernel` is a CUDA compute shader that performs volumetric light transport by stepping through a 3D grid, dynamically adjusting step size and cascade resolution based on distance from the center, and accumulating transmittance (alpha) [S1, unofficial]. It is designed for the IDDR-NGP backend, where it serves as the core inference step for sparse ray marching, optimizing performance by coarser sampling in distant regions and finer sampling near the center [S1, unofficial]. It differs from standard uniform ray marching by using a logarithmic expansion factor to vary step sizes and cascade levels, and from dense voxel cone tracing by operating on a sparse grid structure with specific boundary handling [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** `id` (SV_DispatchThreadID, `uint3`), representing the thread's position in the output grid.
- **Global Constants:** `GridResolution` (uint, side length of the cubic grid), `GridHalfResolution` (float, half of `GridResolution`), `GridCascadeBase` (uint2, `{base_level, max_level}`), `CVCT_gv` (texture/array of voxel values), `LightTex3d` (output texture).
- **Output:** `LightTex3d[id]` (float), the accumulated transmittance (alpha) along the ray.

**Data Structures:**
- `CVCT_gv`: A 3D texture or array storing the raw voxel density/transmittance values. Indexed as `(z * GridResolution + y) * GridResolution + x`.
- `LightTex3d`: A 3D texture storing the final alpha values.

**Helper Function: SampleGV**
The source provides a `SampleGV` implementation that performs trilinear interpolation on `CVCT_tex3d` (which appears to be a pre-processed version of `CVCT_gv`, likely $1 - \text{density}$ or similar, based on the `RepackAndUpscaleKernel` snippet showing `1 - ff` and `1 - CVCT_gv`). For the purpose of `TracingKernel`, `SampleGV(p)` returns a float value at position `p` using trilinear interpolation. The exact mapping of `CVCT_tex3d` to physical density is handled by the preprocessing step, but `TracingKernel` treats the sampled value `a` as the probability of absorption/transmittance factor per unit step.

**Algorithm Steps:**
1.  **Coordinate Transformation:** Convert the thread ID `id` to a local coordinate system centered at the grid origin.
    $$ \text{pos} = \text{id} - (\text{GridHalfResolution} - 0.5) $$
2.  **Distance Calculation:** Compute the Chebyshev distance (max norm) from the center.
    $$ \text{central\_pos} = |\text{pos}| $$
    $$ \text{central\_distance} = \max(\text{central\_pos}.x, \max(\text{central\_pos}.y, \text{central\_pos}.z)) $$
3.  **Normalization and Expansion Factor:** Normalize the distance to the range $[0, 1]$ and compute an initial expansion factor.
    $$ \text{max\_distance} = \text{GridHalfResolution} - 0.5 $$
    $$ \text{central\_distance} \leftarrow \text{central\_distance} / \text{max\_distance} $$
    $$ \text{expand} = \max(\log_2(\text{central\_distance}) + 2, 1) $$
    *Note:* This maps the center (`central_distance` $\approx 0$) to `expand` = 1 (finest resolution) and the boundary (`central_distance` $\approx 1$) to `expand` = 2 (coarsest resolution).
4.  **Initialization:**
    $$ \alpha = 1 $$
    $$ \text{grid\_cascade\_base} = \text{GridCascadeBase}.x $$
5.  **Ray Marching Loop:**
    While true:
    a.  **Cascade Check:** If `expand >= 2.0`, check if the current cascade level is at the maximum.
        If `grid_cascade_base >= GridCascadeBase.y`, break the loop.
        Otherwise, advance to the next cascade level:
        $$ \text{grid\_cascade\_base} \leftarrow \text{grid\_cascade\_base} + \text{GridResolution} $$
        $$ \text{expand} \leftarrow \text{expand} \times 0.5 $$
        $$ \text{pos} \leftarrow \text{pos} \times 0.5 $$
    b.  **Sample Low Position:** Calculate the sample position in the current cascade.
        $$ \text{p} = \text{pos} + (\text{GridHalfResolution} - 0.5) $$
        $$ \text{p}.y \leftarrow \text{p}.y + \text{grid\_cascade\_base} $$
        $$ a_{\text{low}} = \text{SampleGV}(\text{p}) $$
    c.  **Sample High Position:** Calculate the sample position for the next cascade (or boundary).
        If `grid_cascade_base >= GridCascadeBase.y`:
        $$ a_{\text{high}} = 1 $$
        Else:
        $$ \text{q} = \text{pos} \times 0.5 + (\text{GridHalfResolution} - 0.5) $$
        $$ \text{q}.y \leftarrow \text{q}.y + \text{grid\_cascade\_base} + \text{GridResolution} $$
        $$ a_{\text{high}} = \text{SampleGV}(\text{q}) $$
    d.  **Interpolate Alpha:** Blend between the low and high cascade samples based on the expansion factor.
        $$ a = \text{lerp}(a_{\text{low}}, a_{\text{high}}, \text{expand} - 1) $$
    e.  **Update Step Size and Accumulate:**
        $$ \text{sample\_step} = \text{SAMPLE\_STEP\_BASE} \times \text{expand} $$
        $$ \alpha \leftarrow \alpha \times a^{\text{sample\_step}} $$
        $$ \text{pos}.z \leftarrow \text{pos}.z - \text{sample\_step} $$
        $$ \text{expand} \leftarrow \text{expand} \times \text{EXPAND\_STEP} $$
    f.  **Termination:** The loop continues until `expand >= 2.0` and `grid_cascade_base` reaches the limit, or effectively when the ray exits the volume (though explicit exit checks are not present, the cascade limit acts as the boundary).
6.  **Write Result:**
    $$ \text{id}.y \leftarrow \text{id}.y + \text{GridCascadeBase}.x $$
    $$ \text{LightTex3d}[\text{id}] = \alpha $$

**Parameters:**
- `SAMPLE_STEP_BASE`: Base step size for tracing. Typical value: `0.45` [S1, unofficial].
- `EXPAND_STEP`: Multiplier for the expansion factor per step. Typical value: `1.21` [S1, unofficial].

**Pitfalls and Edge Cases:**
- **Infinite Loop:** The loop is guaranteed to terminate because `expand` increases by `EXPAND_STEP` (1.21) each iteration, eventually exceeding 2.0, at which point the cascade base is incremented. If the cascade base is already at `GridCascadeBase.y`, the loop breaks immediately upon the first check.
- **Boundary Conditions:** When `grid_cascade_base` reaches `GridCascadeBase.y`, `a_high` is clamped to 1. This assumes that beyond the highest cascade level, the volume is empty or fully transmissive.
- **Precision:** The use of `log2` and floating-point arithmetic for `expand` may introduce small precision errors, but the integer-based cascade base ensures robust level switching.
- **Coordinate System:** The `pos` variable is modified in place (scaled and shifted), which requires careful tracking of the coordinate system relative to the grid origin.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def TracingKernel(id: uint3, GridResolution: int, GridHalfResolution: float, 
                  GridCascadeBase: uint2, CVCT_gv: Texture3D, LightTex3d: Texture3D):
    # Helper for trilinear interpolation
    def SampleGV(p: float3) -> float:
        # Assuming CVCT_tex3d is accessible and pre-processed
        # This is a placeholder for the actual interpolation logic provided in the source
        # which uses CVCT_tex3d. For implementation, one must use the specific 
        # hardware interpolation or manual trilinear code.
        return CVCT_tex3d.SampleLevel(p, 0).r 

    pos = id - (GridHalfResolution - 0.5)
    
    central_pos = abs(pos)
    central_distance = max(central_pos.x, max(central_pos.y, central_pos.z))
    
    max_distance = GridHalfResolution - 0.5
    central_distance /= max_distance
    
    expand = max(log2(central_distance) + 2, 1.0)
    
    alpha = 1.0
    grid_cascade_base = GridCascadeBase.x
    
    SAMPLE_STEP_BASE = 0.45
    EXPAND_STEP = 1.21
    
    while True:
        if expand >= 2.0:
            if grid_cascade_base >= GridCascadeBase.y:
                break
            grid_cascade_base += GridResolution
            expand *= 0.5
            pos *= 0.5
            
        p = pos + (GridHalfResolution - 0.5)
        p.y += grid_cascade_base
        a_low = SampleGV(p)
        
        if grid_cascade_base >= GridCascadeBase.y:
            a_high = 1.0
        else:
            q = pos * 0.5 + (GridHalfResolution - 0.5)
            q.y += grid_cascade_base + GridResolution
            a_high = SampleGV(q)
            
        a = lerp(a_low, a_high, expand - 1.0)
        
        sample_step = SAMPLE_STEP_BASE * expand
        alpha *= pow(a, sample_step)
        pos.z -= sample_step
        expand *= EXPAND_STEP
        
    id.y += GridCascadeBase.x
    LightTex3d[id] = alpha
```

## Operating characteristics
- **Time Complexity:** The number of steps per ray depends on the distance from the center and the `EXPAND_STEP`. Rays near the center take more steps with smaller step sizes, while rays near the boundary take fewer steps with larger step sizes. The total number of steps is roughly proportional to the distance divided by the average step size, which scales logarithmically with distance due to the expansion factor.
- **Memory Complexity:** The kernel uses constant memory for global parameters and local registers for state. It reads from `CVCT_tex3d` and writes to `LightTex3d`. The memory access pattern is coherent within thread blocks due to the structured grid traversal.
- **Performance:** The dynamic step size and cascade switching allow for efficient rendering of large volumes by reducing the number of samples in empty or distant regions. The use of CUDA compute shaders enables parallel execution across the entire grid.
- **Strengths:** Adaptive resolution based on distance, efficient use of sparse grid structures, simple implementation.
- **Limitations:** The fixed expansion factor and step size parameters may not be optimal for all volume types. The assumption of `a_high = 1` at the boundary may introduce artifacts if the volume extends beyond the grid.

## Background
`TracingKernel` is part of the Cascaded Voxel Cone Tracing (CVCT) technique, specifically implemented in the `arigo/CascatedVoxelConeTracing` repository [S1, unofficial]. It is designed for real-time volumetric rendering, particularly in the context of neural graphics primitives (NGP) and sparse grid representations. The method was developed to address the computational cost of dense ray marching by leveraging the sparsity of the grid and the hierarchical nature of the data.

## Sources
[S1] arigo/CascatedVoxelConeTracing, https://github.com/arigo/CascatedVoxelConeTracing, fetched 2026-10-26 (unofficial)
