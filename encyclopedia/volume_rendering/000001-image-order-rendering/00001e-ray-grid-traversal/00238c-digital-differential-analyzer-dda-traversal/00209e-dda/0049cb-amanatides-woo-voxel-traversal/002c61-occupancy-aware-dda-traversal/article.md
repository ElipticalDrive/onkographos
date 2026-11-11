# Occupancy-aware DDA Traversal

Also called: Active Voxel DDA, Branching DDA, DDA Branch, DDA Branch Kernel, Occupancy-based DDA

## Overview
Occupancy-aware DDA Traversal is a ray-sampling kernel variant that traverses a voxel grid using standard Digital Differential Analyzer (DDA) logic but branches execution based on the occupancy status of each visited voxel [S1, unofficial]. Unlike sibling methods that skip empty space entirely (e.g., DDA Skip), this method synchronizes traversal across all workers and processes every voxel, conditionally recording samples only for active voxels while advancing time for inactive ones [S1, unofficial]. It is primarily adopted in the NerfAcc library for accelerating occupancy grid-based NeRF rendering [S1, unofficial].

## How it works
The method builds on [Amanatides-Woo Voxel Traversal](3132-amanatides-woo-voxal-traversal.md) by replacing the continuous stepping logic with a discrete, occupancy-gated loop. The core distinction is that the traversal interval $[t_0, t_1]$ for the current voxel is determined first, and then the algorithm iterates through this interval, checking occupancy at each step [S1, unofficial].

**Inputs and Outputs:**
- **Inputs:** Ray origin $o$, ray direction $d$, near plane $t_{min}$, far plane $t_{max}$, and a voxel grid data structure supporting an occupancy query `grid.at(ijk)` which returns a boolean [S1, unofficial].
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
def dda_branch(ray_o, ray_d, t_min, t_max, grid, delta_t):
    """
    Occupancy-aware DDA Traversal.
    
    Args:
        ray_o: Ray origin (3,)
        ray_d: Ray direction (3,)
        t_min: Near plane distance
        t_max: Far plane distance
        grid: Object with method at(ijk) returning boolean occupancy
        delta_t: Step size for sampling
        
    Returns:
        buf: List of sample distances t
    """
    buf = []
    t_last = t_min
    
    while t_last <= t_max:
        # Analyzer returns voxel index ijk and interval [t0, t1]
        # This relies on the parent Amanatides-Woo logic or HDDA
        ijk, t0, t1 = analyzer(ray_o, ray_d, t_last)
        
        # Advance t_last until it enters the current voxel
        while t_last <= t0:
            t_last += delta_t
            
        # Traverse inside the current voxel
        while t_last <= t1:
            # Check occupancy of the current voxel
            if grid.at(ijk):
                buf.append(t_last)
            t_last += delta_t
            
    return buf
```

**Edge Cases and Pitfalls:**
- **Precision:** The source does not specify floating-point precision handling. Implementers should ensure $t_{last}$ comparisons are robust against floating-point drift, especially when $\Delta t$ is small [G].
- **Empty Voxels:** Unlike DDA Skip, this method does not skip empty voxels. It iterates through them, performing the occupancy check and advancing $t_{last}$ without recording samples. This can lead to lower performance in sparse scenes compared to skipping variants [S1, unofficial].
- **Multi-resolution:** When using multi-resolution VDB grids (e.g., for Mip-NeRF 360), the `Analyzer` must prioritize finer voxels if multiple voxels are intersected at the same time [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of voxels intersected by the ray, assuming constant-time occupancy checks. Unlike DDA Skip, it does not reduce the number of iterations for empty space [S1, unofficial].
- **Memory Complexity:** $O(S)$ where $S$ is the number of samples recorded. The buffer stores only samples from active voxels [S1, unofficial].
- **Performance:** The method synchronizes traversal across all workers, which can be beneficial for GPU parallelism but may cause divergence if occupancy is highly irregular [S1, unofficial]. It is slower than DDA Skip in sparse scenes but simpler to implement with standard DDA logic [S1, unofficial].
- **Strengths:** Simple integration with existing DDA/HDDA analyzers; consistent synchronization across threads [S1, unofficial].
- **Limitations:** Inefficient for sparse grids due to lack of empty space skipping [S1, unofficial].

## Background
The method was introduced in the supplementary material of "Plug-and-Play Acceleration of Occupancy Grid-based NeRF Rendering using VDB Grid and Hierarchical Ray Traversal" [S1, unofficial]. It is named "DDA branch" to distinguish it from "DDA skip" (used by InstantNGP) and is adopted by the NerfAcc library [S1, unofficial]. The approach leverages standard DDA or Hierarchical DDA (HDDA) as the underlying analyzer [S1, unofficial].

## Sources
[S1] Plug-and-Play Acceleration of Occupancy Grid-based NeRF Rendering using VDB Grid and Hierarchical Ray Traversal, https://doi.org/10.48550/arxiv.2404.10272, fetched 2026-10-24 (unofficial)
[S2] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
