# Occupancy-aware DDA Traversal

Also called: Active Voxel DDA, Branching DDA, DDA Branch, DDA Branch Kernel, Occupancy-based DDA

## Overview
Occupancy-aware DDA Traversal is a ray-sampling kernel variant that traverses a voxel grid using standard Digital Differential Analyzer (DDA) logic but branches execution based on the occupancy status of each visited voxel [S1, unofficial]. Unlike sibling methods that skip empty space entirely (e.g., DDA Skip), this method synchronizes traversal across all workers and processes every voxel, conditionally recording samples only for active voxels while advancing time for inactive ones [S1, unofficial]. It is primarily adopted in the NerfAcc library for accelerating occupancy grid-based NeRF rendering [S1, unofficial].

## How it works
The method builds on [Amanatides-Woo Voxel Traversal](3132-amanatides-woo-voxal-traversal.md) by using a standard DDA or Hierarchical DDA (HDDA) analyzer to determine voxel intervals, then branching execution based on occupancy [S1, unofficial]. The core distinction is that the traversal interval $[t_0, t_1]$ for the current voxel is determined first, and then the algorithm iterates through this interval, checking occupancy at each step [S1, unofficial].

**Inputs and Outputs:**
- **Inputs:** Ray origin $o$, ray direction $d$, near plane $t_{min}$, far plane $t_{max}$, voxel size, grid origin, and a voxel grid data structure supporting an occupancy query `grid.at(ijk)` which returns a boolean [S1, unofficial].
- **Outputs:** A buffer `buf` containing a list of sample positions $t$ along the ray [S1, unofficial].

**Data Structures:**
- `buf`: A dynamic list or array of floats storing sample distances $t$ [S1, unofficial].
- `grid`: A dense or VDB grid structure. The source specifies `grid.at(ijk)` returns whether the voxel at index `ijk` is occupied [S1, unofficial].
- `Analyzer`: A helper function (DDA or HDDA) that returns the voxel index `ijk` and the entry/exit $t$-values $[t_0, t_1]$ for the next voxel intersected by the ray starting from a given $t$ [S1, unofficial].

**Parameters:**
- $\Delta t$: The step size for sampling within a voxel. The source notes a "linearly increasing stepsize" for specific datasets like Mip-NeRF 360, but the algorithm structure uses a constant $\Delta t$ per iteration [S1, unofficial].
- $t_{last}$: The current position along the ray, initialized to $t_{min}$ [S1, unofficial].

**Algorithm Steps:**
1. Initialize `buf` as an empty list.
2. Set $t_{last} \leftarrow t_{min}$.
3. While $t_{last} \le t_{max}$:
    a. Call `Analyzer(ray, t_last)` to obtain the current voxel index `ijk` and the interval $[t_0, t_1]$ where the ray is inside this voxel [S1, unofficial].
    b. **Pre-voxel gap:** While $t_{last} \le t_0$:
        i. $t_{last} \leftarrow t_{last} + \Delta t$ [S1, unofficial].
    c. **Voxel traversal:** While $t_{last} \le t_1$:
        i. Check occupancy: `is_active = grid.at(ijk)` [S1, unofficial].
        ii. If `is_active` is true:
            1. Append $t_{last}$ to `buf` [S1, unofficial].
        iii. $t_{last} \leftarrow t_{last} + \Delta t$ [S1, unofficial].
4. Return `buf` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dda_branch(ray_o, ray_d, t_min, t_max, voxel_size, grid_origin, grid_at, delta_t):
    """
    Occupancy-aware DDA Traversal.
    
    Args:
        ray_o: Ray origin (3,)
        ray_d: Ray direction (3,)
        t_min: Near plane distance
        t_max: Far plane distance
        voxel_size: Uniform voxel edge length
        grid_origin: World-space origin of the voxel grid
        grid_at: Function returning boolean occupancy for voxel index ijk
        delta_t: Step size for sampling
        
    Returns:
        buf: List of sample distances t
    """
    buf = []
    t_last = t_min
    
    while t_last <= t_max:
        # Standard Amanatides-Woo DDA advance to compute current voxel and interval
        ijk = floor((ray_o + ray_d * t_last - grid_origin) / voxel_size)
        t0 = t_last
        t1 = t_max
        for axis in range(3):
            if ray_d[axis] > 0:
                t0 = max(t0, (grid_origin[axis] + ijk[axis] * voxel_size - ray_o[axis]) / ray_d[axis])
                t1 = min(t1, (grid_origin[axis] + (ijk[axis] + 1) * voxel_size - ray_o[axis]) / ray_d[axis])
            elif ray_d[axis] < 0:
                t0 = max(t0, (grid_origin[axis] + (ijk[axis] + 1) * voxel_size - ray_o[axis]) / ray_d[axis])
                t1 = min(t1, (grid_origin[axis] + ijk[axis] * voxel_size - ray_o[axis]) / ray_d[axis])
            else:
                t0 = max(t0, t_last)
                t1 = min(t1, t_last)
                
        # Advance t_last to enter the current voxel
        while t_last <= t0:
            t_last += delta_t
            
        # Traverse inside the current voxel
        while t_last <= t1:
            if grid_at(ijk):
                buf.append(t_last)
            t_last += delta_t
            
    return buf
```

**Edge Cases and Pitfalls:**
- **Precision:** The source does not specify floating-point precision handling. Implementers should ensure $t_{last}$ comparisons are robust against floating-point drift, especially when $\Delta t$ is small [G].
- **Empty Voxels:** Unlike DDA Skip, this method does not skip empty voxels. It iterates through them, performing the occupancy check and advancing $t_{last}$ without recording samples [S1, unofficial].
- **Multi-resolution:** When using multi-resolution VDB grids (e.g., for Mip-NeRF 360), the `Analyzer` must prioritize finer voxels if multiple voxels are intersected at the same time [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** The method synchronizes traversal across all workers, which can be beneficial for GPU parallelism but may cause divergence if occupancy is highly irregular [S1, unofficial].
- **Strengths:** Simple integration with existing DDA/HDDA analyzers; consistent synchronization across threads [S1, unofficial].
- **Limitations:** Processes every voxel regardless of occupancy, branching execution to record samples only for active voxels [S1, unofficial].

## Background
The method was introduced in the supplementary material of "Plug-and-Play Acceleration of Occupancy Grid-based NeRF Rendering using VDB Grid and Hierarchical Ray Traversal" [S1, unofficial]. It is named "DDA branch" to distinguish it from "DDA skip" (used by InstantNGP) and is adopted by the NerfAcc library [S1, unofficial]. The approach leverages standard DDA or Hierarchical DDA (HDDA) as the underlying analyzer [S1, unofficial].

## Sources
[S1] Plug-and-Play Acceleration of Occupancy Grid-based NeRF Rendering using VDB Grid and Hierarchical Ray Traversal, https://doi.org/10.48550/arxiv.2404.10272, fetched 2026-10-24 (unofficial)
[S2] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
