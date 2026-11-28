# Multi-level DDA with Adaptive Mip Selection

Also called: DDA-based volume sampling routine, Multi-level traversal scheme

## Overview
This method is a ray traversal and sampling algorithm for volume rendering that combines Digital Differential Analyzer (DDA) brick traversal with delta tracking, using multiple mip levels of a majorant texture to adaptively balance step size and sampling accuracy [S1, book]. It solves the problem of inefficient sampling in sparse or heterogeneous volumes by allowing large steps in empty regions (coarse mip levels) while maintaining precision near density changes (fine mip levels) [S1, book]. It sets itself apart from standard DDA by dynamically adjusting the spatial resolution of the majorant lookup per step without branching, and from pure delta tracking by using DDA to skip empty space more efficiently [S1, book].

## How it works
The algorithm traverses a ray through a volume hierarchy of bricks, using a majorant texture to estimate the probability of interaction [S1, book]. It maintains a target optical thickness (`tauToGo`) and steps through the volume, dynamically selecting the mip level for the majorant lookup based on a heuristic that increases resolution as the ray progresses and decreases it upon collision detection [S1, book].

### Inputs and Outputs
*   **Inputs**: Ray origin (`rayOrigin`), ray direction (`rayDir`), volume data (density texture, majorant texture hierarchy), initial distance `t` (usually 0 or start of volume) [S1, book].
*   **Outputs**: Collision point, type of collision (real or null), updated ray state [S1, book].

### Data Structures
*   **Majorant Texture Hierarchy**: A set of mip levels (0 to 3) of a texture storing the majorant density [S1, book]. The source specifies 3 mip levels are computed [S1, book].
*   **Brick Geometry**: Bricks are dilated by 0.5 voxels in the direction of travel to improve numerical stability and reduce step counts [S1, book]. The brick dimension at mip level `m` is `dim = 8 << m` [S1, book].

### Parameters
*   **Initial Mip Level**: 3.0 (coarsest) [S1, book].
*   **Mip Increment**: 0.33 per iteration [S1, book].
*   **Mip Decrement on Collision**: 2.0 [S1, book].
*   **Mip Bounds**: Clamped between 0.0 and 3.0 [S1, book].
*   **Brick Dilation**: 0.5 voxels [S1, book].

### Numbered Steps
1.  Initialize the target optical thickness `tauToGo` by sampling a uniform random number `r` and computing `tauToGo = -log(1.0 - r)` [S1, book].
2.  Initialize the mip level `mip = 3.0` [S1, book].
3.  Enter the volume traversal loop (while inside the volume):
    a.  Calculate the current ray position: `pos = rayOrigin + t * rayDir` [S1, book].
    b.  Lookup the majorant density at `pos` using the current mip level, rounded to an integer: `majorant = lookupMajorant(pos, round(mip))` [S1, book].
    c.  Compute the next DDA step distance `nextt` using the `stepDDA` function with the current mip level [S1, book].
    d.  Increment the mip level: `mip = min(3.0, mip + 0.33)` [S1, book].
    e.  Calculate the step distance `dt = nextt - t` and the optical thickness traversed `dtau = majorant * dt` [S1, book].
    f.  Update the current distance `t = nextt` [S1, book].
    g.  Subtract the traversed optical thickness from the target: `tauToGo -= dtau` [S1, book].
    h.  If `tauToGo > 0`, continue to the next iteration (no collision yet) [S1, book].
    i.  If `tauToGo <= 0` (collision detected):
        i.  Decrement the mip level: `mip = max(0.0, mip - 2.0)` [S1, book].
        ii. Step back to the exact collision point using linear interpolation: `t += dt * (tauToGo / dtau)` [S1, book].
        iii. Lookup the actual density at the collision point: `density = lookup(rayOrigin + t * rayDir)` [S1, book].
        iv. Perform Russian roulette for real vs. null collision: if `random() * majorant < density`, handle a real collision; otherwise, treat as a null collision [S1, book].
        v.  For a null collision, restart the traversal by sampling a new target optical thickness: `tauToGo = -log(1.0 - random())` [S1, book].

### Formulas
*   **Target Optical Thickness**: `tauToGo = -log(1.0 - random())` [S1, book].
*   **Brick Dimension**: `dim = 8 << mip` [S1, book].
*   **DDA Offset**: `ofs = ri * (((ri >= 0.0) ? dim + 0.5 : -0.5) - ro)` [S1, book].
    *   Where `ri` is the ray inverse direction (`1.0 / rayDir`) and `ro` is the ray origin [S1, book].
*   **DDA Step Distance**: `tmax = floor(pos * (1.0 / dim)) * dim * ri + ofs` [S1, book].
    *   The final step distance is `min(tmax.x, min(tmax.y, tmax.z))` [S1, book].
*   **Collision Point Refinement**: `t_collision = t_prev + dt * (tauToGo_remaining / dtau_traversed)` [S1, book].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def step_dda(ro, ri, pos, mip):
    """
    Compute the next DDA step distance using dilated brick boundaries.
    ro: Ray origin (float3)
    ri: Ray inverse direction (float3)
    pos: Current ray position (float3)
    mip: Current mip level (float)
    """
    dim = 8 << int(mip)
    # Calculate offset component-wise based on direction and dilation
    ofs_x = ri.x * ((dim + 0.5 - ro.x) if ri.x >= 0 else (-0.5 - ro.x))
    ofs_y = ri.y * ((dim + 0.5 - ro.y) if ri.y >= 0 else (-0.5 - ro.y))
    ofs_z = ri.z * ((dim + 0.5 - ro.z) if ri.z >= 0 else (-0.5 - ro.z))
    
    # Calculate tmax for each dimension
    tmax_x = math.floor(pos.x * (1.0 / dim)) * dim * ri.x + ofs_x
    tmax_y = math.floor(pos.y * (1.0 / dim)) * dim * ri.y + ofs_y
    tmax_z = math.floor(pos.z * (1.0 / dim)) * dim * ri.z + ofs_z
    
    return min(tmax_x, tmax_y, tmax_z)

def multi_level_dda_adaptive_mip(ray_origin, ray_dir, volume_data):
    """
    Main traversal loop.
    ray_origin: float3
    ray_dir: float3
    volume_data: Object containing lookup_majorant and lookup functions
    """
    # Initialize target optical thickness
    r = random.random()
    tau_to_go = -math.log(1.0 - r)
    
    # Initialize mip level
    mip = 3.0
    
    # Assume t starts at 0 or entry point
    t = 0.0
    
    # Precompute inverse ray direction
    inv_ray_dir = 1.0 / ray_dir
    
    # Loop until collision handled or ray exits volume (volume exit handled externally)
    while True:
        # Calculate current position
        pos = ray_origin + t * ray_dir
        
        # Lookup majorant at current mip level
        majorant = volume_data.lookup_majorant(pos, int(round(mip)))
        
        # Compute next DDA step
        next_t = step_dda(ray_origin, inv_ray_dir, pos, int(round(mip)))
        
        # Increment mip level
        mip = min(3.0, mip + 0.33)
        
        # Calculate step distance and optical thickness
        dt = next_t - t
        dtau = majorant * dt
        
        # Update current distance
        t = next_t
        
        # Update remaining optical thickness
        tau_to_go -= dtau
        
        # Check for collision
        if tau_to_go > 0:
            continue # No collision, continue to next step
        
        # Collision detected
        # Decrement mip level
        mip = max(0.0, mip - 2.0)
        
        # Step back to exact collision point
        t += dt * (tau_to_go / dtau)
        
        # Lookup actual density at collision point
        collision_pos = ray_origin + t * ray_dir
        density = volume_data.lookup(collision_pos)
        
        # Russian roulette for real vs null collision
        r = random.random()
        if r * majorant < density:
            # Handle real collision
            return {"type": "real", "position": collision_pos, "density": density}
        else:
            # Null collision
            # Restart traversal with new target
            r = random.random()
            tau_to_go = -math.log(1.0 - r)
            # Continue loop to step forward again
```

### Edge Cases and Pitfalls
*   **Numerical Instability on Brick Boundaries**: Ray intersections falling exactly on brick boundaries can cause numerical issues [S1, book]. This is mitigated by dilating brick boundaries by 0.5 voxels in the direction of travel, ensuring the DDA oversteps into each brick [S1, book].
*   **Execution Divergence on GPUs**: Implementing separate loops for each mip level causes execution divergence on SIMT devices (GPUs) [S1, book]. The adaptive mip selection is computed dynamically per step without branching to avoid this [S1, book].
*   **Precision**: The mip level is rounded to an integer for texture lookup, but maintained as a float for the heuristic increment/decrement [S1, book].

## Subtopics

### Hierarchical Digital Differential Analyzer (HDDA)
A sibling method that uses a hierarchical DDA structure [S1, book]. It differs in its specific implementation of the hierarchy and traversal logic [S1, book]. [HDDA](1122-hierarchical-digital-differential-analyzer-hdda.md)

### VolumeHDDA
A sibling method, likely an extension or variant of HDDA for volume rendering. [VolumeHDDA](2353-volumehdda.md)

## Operating characteristics
*   **Time Complexity**: The number of steps is reduced compared to regular DDA due to larger steps in empty space and dilated boundaries [S1, book]. For example, the adaptive DDA achieved 3.9ms on an NVIDIA RTX 3090 for one sample per pixel at 1920x1080 with 3 bounces on the Moana cloud scene [S1, book].
*   **Memory Complexity**: Memory usage is 86.7MB for the Moana cloud scene with BC4 compression [S1, book].
*   **Strengths**: Reduces the number of volume and majorant lookups, especially in empty space regions [S1, book]. Improves numerical stability [S1, book]. Avoids execution divergence on GPUs [S1, book].
*   **Limitations**: Requires computation of multiple mip levels of the majorant texture [S1, book].
*   **When to Use**: Suitable for GPU-based unbiased volume path tracing where performance and numerical stability are critical [S1, book].

## Background
The method was introduced in "Efficient Unbiased Volume Path Tracing on the GPU" [S1, book]. It builds on the principles of Hierarchical DDA Traversal [17026-hierarchical-dda-traversal.md] and adapts them for volume rendering by integrating delta tracking and adaptive mip selection [S1, book]. The approach was inspired by techniques used in screen-space reflections [S1, book].

## Sources
[S1] Efficient Unbiased Volume Path Tracing on the GPU, https://doi.org/10.1007/978-1-4842-7185-8_43, fetched 2026-10-24 (book)
