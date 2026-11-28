# DDA Skip

Also called: DDA Skip Kernel, Skipping DDA

## Overview
DDA Skip is a ray-sampling kernel that bypasses all leading empty voxels in a sparse grid before initiating the standard marching process through occupied regions [S1, unofficial]. It is distinguished from its sibling [DDA Branch](3132-amanatides-woo-voxel-traversal.md) by deferring occupancy checks until an occupied voxel is guaranteed to be found, rather than branching per voxel [S1, unofficial]. This method is adopted by Instant-NGP [S1, unofficial].

## How it works
**Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, near plane $t_{min}$, far plane $t_{max}$, sparse grid $G$ (occupancy map), and step size $\Delta t$ [S1, unofficial].
**Outputs:** List of sample points $buf$ along the ray within $[t_{min}, t_{max}]$ where $G$ is occupied [S1, unofficial].

**Data Structures:**
* $buf$: List of floats (sample distances $t$).
* $t_{last}$: Float, current sampling position, initialized to $t_{min}$.
* $ijk$: Integer triplet, current voxel coordinates.
* $t_0, t_1$: Floats, entry and exit distances of the current voxel along the ray.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dda_skip(ray, t_min, t_max, grid, delta_t):
    buf = []
    t_last = t_min
    while t_last <= t_max:
        t1 = t_min
        while True:
            ijk, t0, t1 = dda_step(ray, t1)  # Advances DDA state to next voxel boundary [G]
            if grid.at(ijk):
                break
        while t_last <= t0:
            t_last += delta_t
        while t_last <= t1:
            buf.append(t_last)
            t_last += delta_t
    return buf
```

**Parameters:**
* $\Delta t$: Step size. Not given in the sources.

**Edge Cases:**
* Floating-point errors may cause discrepancies in rendering results compared to other samplers [S1, unofficial].
* If the ray never hits an occupied voxel, the inner repeat loop may run indefinitely or until $t_{max}$ is exceeded depending on implementation bounds [G].

## Subtopics

## Operating characteristics
* **Performance:** DDA Skip performs poorly with standard DDA, offering no speedup or even slowing down rendering [S1, unofficial]. However, when combined with VDB grids and Hierarchical DDA (HDDA), it renders faster than the baseline [S1, unofficial].
* **Strengths:** Reduces computational overhead in threads when using hierarchical structures like VDB [S1, unofficial].
* **Limitations:** Ineffective with dense or standard DDA traversals [S1, unofficial].

## Background
DDA Skip was introduced in the context of accelerating occupancy grid-based NeRF rendering [S1, unofficial]. It is specifically adopted by Instant-NGP [S1, unofficial]. The method contrasts with "DDA Branch" (used in NerfAcc), which synchronizes traversal and branches based on occupancy per voxel [S1, unofficial].

## Sources
[S1] Plug-and-Play Acceleration of Occupancy Grid-based NeRF Rendering using VDB Grid and Hierarchical Ray Traversal, https://doi.org/10.48550/arxiv.2404.10272, fetched 2026-10-24 (unofficial)
[S2] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
