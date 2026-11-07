# Restartable DDA for Voxel Traversal

Also called: DdaState, Voxel Marcher

## Overview
Restartable DDA for Voxel Traversal is a Digital Differential Analyzer (DDA) algorithm designed to traverse a 3D voxel grid while allowing the ray direction to be modified mid-walk, such as during refraction events [S1, unofficial]. It solves the problem of maintaining traversal state consistency when a ray changes direction, enabling efficient volume transmission calculations without restarting the entire traversal from the original origin [S1, unofficial]. It is distinct from standard Amanatides and Woo traversal by its explicit support for re-initializing the state structure with a new origin and direction while preserving the grid geometry context, and it differs from [3D Bresenham Voxel Tracing](3949-3d-bresenham-voxel-tracing.md) by using floating-point distance calculations rather than integer arithmetic [S1, unofficial].

## How it works
The method operates by maintaining a state structure that tracks the current position, distance to the next voxel boundary on each axis, and the step direction [S1, unofficial]. The algorithm consists of an initialization phase that computes these values based on the ray origin, direction, and grid bounds, and a stepping phase that advances the ray to the nearest boundary [S1, unofficial].

**Inputs and Outputs:**
*   **Inputs:** Ray origin $O$, ray direction $D$, grid bounding box $[B_{min}, B_{max}]$, voxel size $C$, grid dimensions $N$ [S1, unofficial].
*   **Outputs:** A boolean indicating success (hit) or failure (miss/out of bounds), and the updated `DdaState` structure [S1, unofficial].

**Data Structures:**
The core state is held in `DdaState` and `GridDims` [S1, unofficial]:
*   `DdaState`:
    *   `d`: `float3` Normalized direction components. Clamped to magnitude $10^{-8}$ if near zero [S1, unofficial].
    *   `idx`: `int3` Current voxel indices $(i, j, k)$ [S1, unofficial].
    *   `stp`: `int3` Step direction per axis ($1$ or $-1$) [S1, unofficial].
    *   `t_max`: `float3` Distance to the next voxel boundary along each axis [S1, unofficial].
    *   `t_delta`: `float3` Distance required to cross one voxel along each axis [S1, unofficial].
    *   `t_cur`: `float` Current distance along the ray from the origin [S1, unofficial].
    *   `axis`: `int` The axis of the current traversal step (0, 1, or 2) [S1, unofficial].
    *   `enter_axis`: `int` The axis of entry into the grid ($-1$ if not set) [S1, unofficial].
*   `GridDims`:
    *   `bmin`: `float3` Bounding box minimum [S1, unofficial].
    *   `bmax`: `float3` Bounding box maximum [S1, unofficial].
    *   `cell`: `float3` Voxel dimensions per axis [S1, unofficial].
    *   `dims`: `int3` Voxel count per axis [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization (`dda_init`):**
    1.  Set $t_{min} = 0.0$, $t_{exit} = 10^{30}$, $enter\_axis = -1$ [S1, unofficial].
    2.  For each axis $a \in \{0, 1, 2\}$:
        *   If $|D[a]| < 10^{-8}$, set $D[a] = \text{sign}(D[a]) \times 10^{-8}$ (or $10^{-8}$ if $D[a]=0$) [S1, unofficial].
        *   Calculate $t_0 = (B_{min}[a] - O[a]) / D[a]$ and $t_1 = (B_{max}[a] - O[a]) / D[a]$ [S1, unofficial].
        *   If $t_0 > t_1$, swap $t_0$ and $t_1$ [S1, unofficial].
        *   If $t_0 > t_{min}$, set $enter\_axis = a$ [S1, unofficial].
        *   Update $t_{min} = \max(t_{min}, t_0)$ [S1, unofficial].
        *   Update $t_{exit} = \min(t_{exit}, t_1)$ [S1, unofficial].
    3.  If $t_{min} > t_{exit}$, return `false` (miss) [S1, unofficial].
    4.  Calculate starting position: $P = O + D \times (t_{min} + 10^{-4})$ [S1, unofficial].
    5.  For each axis $a \in \{0, 1, 2\}$:
        *   $idx[a] = \text{clamp}(\lfloor (P[a] - B_{min}[a]) / C[a] \rfloor, 0, N[a] - 1)$ [S1, unofficial].
        *   $stp[a] = D[a] > 0 ? 1 : -1$ [S1, unofficial].
        *   Calculate next boundary: $Next = B_{min}[a] + (idx[a] + (stp[a] > 0 ? 1 : 0)) \times C[a]$ [S1, unofficial].
        *   $t_{max}[a] = (Next - O[a]) / D[a]$ [S1, unofficial].
        *   $t_{delta}[a] = C[a] / |D[a]|$ [S1, unofficial].
    6.  Set $axis = enter\_axis < 0 ? 1 : enter\_axis$ [S1, unofficial].
    7.  Set $t_{cur} = t_{min}$ [S1, unofficial].
    8.  Return `true` [S1, unofficial].

2.  **Stepping (`dda_step`):**
    1.  Determine the axis with the smallest $t_{max}$:
        *   If $t_{max}[0] < t_{max}[1]$:
            *   If $t_{max}[0] < t_{max}[2]$, $axis = 0$ [S1, unofficial].
            *   Else, $axis = 2$ [S1, unofficial].
        *   Else:
            *   If $t_{max}[1] < t_{max}[2]$, $axis = 1$ [S1, unofficial].
            *   Else, $axis = 2$ [S1, unofficial].
    2.  Update $t_{cur} = t_{max}[axis]$ [S1, unofficial].
    3.  Increment $idx[axis]$ by $stp[axis]$ [S1, unofficial].
    4.  If $idx[axis] < 0$ or $idx[axis] \ge N[axis]$, return `false` (out of bounds) [S1, unofficial].
    5.  Update $t_{max}[axis] += t_{delta}[axis]$ [S1, unofficial].
    6.  Return `true` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dda_init(origin, dir, bmin, bmax, cell, dims):
    t_min = 0.0
    t_exit = 1e30
    enter_axis = -1
    
    d = [0.0, 0.0, 0.0]
    for a in range(3):
        if abs(dir[a]) < 1e-8:
            d[a] = 1e-8 if dir[a] >= 0 else -1e-8
        else:
            d[a] = dir[a]
            
    for a in range(3):
        t0 = (bmin[a] - origin[a]) / d[a]
        t1 = (bmax[a] - origin[a]) / d[a]
        
        if t0 > t1:
            t0, t1 = t1, t0
            
        if t0 > t_min:
            enter_axis = a
        t_min = max(t_min, t0)
        t_exit = min(t_exit, t1)
        
    if t_min > t_exit:
        return False, None
        
    pos = [origin[a] + d[a] * (t_min + 1e-4) for a in range(3)]
    
    state = {
        'd': d,
        'idx': [0, 0, 0],
        'stp': [0, 0, 0],
        't_max': [0.0, 0.0, 0.0],
        't_delta': [0.0, 0.0, 0.0],
        't_cur': t_min,
        'axis': 1,
        'enter_axis': enter_axis
    }
    
    for a in range(3):
        idx_val = int((pos[a] - bmin[a]) / cell[a])
        idx_val = max(0, min(idx_val, dims[a] - 1))
        state['idx'][a] = idx_val
        
        state['stp'][a] = 1 if d[a] > 0.0 else -1
        
        next_val = bmin[a] + (idx_val + (1 if state['stp'][a] > 0 else 0)) * cell[a]
        state['t_max'][a] = (next_val - origin[a]) / d[a]
        state['t_delta'][a] = cell[a] / abs(d[a])
        
    if state['enter_axis'] < 0:
        state['axis'] = 1
    else:
        state['axis'] = state['enter_axis']
        
    return True, state

def dda_step(state, dims):
    t_max = state['t_max']
    if t_max[0] < t_max[1]:
        if t_max[0] < t_max[2]:
            axis = 0
        else:
            axis = 2
    else:
        if t_max[1] < t_max[2]:
            axis = 1
        else:
            axis = 2
            
    state['axis'] = axis
    state['t_cur'] = t_max[axis]
    
    state['idx'][axis] += state['stp'][axis]
    
    if state['idx'][axis] < 0 or state['idx'][axis] >= dims[axis]:
        return False
        
    state['t_max'][axis] += state['t_delta'][axis]
    
    return True
```

**Parameters and Edge Cases:**
*   **Epsilon:** A small value $10^{-4}$ is added to $t_{min}$ when calculating the initial position to ensure the ray starts strictly inside the first voxel, avoiding boundary precision issues [S1, unofficial].
*   **Direction Clamping:** Direction components near zero are clamped to $\pm 10^{-8}$ to prevent division by zero in $t_{max}$ calculations [S1, unofficial].
*   **Degenerate Direction:** If the new ray direction becomes degenerate (e.g., after refraction, if $\dot{rdir, rdir} < 10^{-6}$), the algorithm falls back to the original direction [S1, unofficial].
*   **Restartability:** The method allows re-initialization with a new origin and direction (e.g., at a refraction surface) by calling `dda_init` again, while reusing the grid parameters [S1, unofficial].

## Subtopics

### [Voxel Water Refraction and Transmission](5216-voxel-water-refraction-and-transmission.md)
A three-phase rendering technique for simulating light interaction with a voxel-based water surface and volume. It is distinct from full volumetric path tracing by using a deterministic ray-marching approach optimized for voxel grids, relying on Restartable DDA for Voxel Traversal to handle the change in ray direction at the surface.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per step for `dda_step` [G]. Initialization is $O(1)$ (constant number of arithmetic operations) [G]. Total time is proportional to the number of voxels traversed [G].
*   **Memory Complexity:** $O(1)$ auxiliary memory for the `DdaState` structure [G].
*   **Strengths:** Efficient for sparse and dense voxel grids; supports mid-traversal direction changes (refraction); robust against degenerate rays and boundary cases [S1, unofficial].
*   **Limitations:** Floating-point precision issues may arise for very large grids or very small voxels if not handled carefully (mitigated by epsilon and clamping) [S1, unofficial].
*   **When to use:** Ideal for volumetric rendering involving refraction, transmission, or any scenario where ray direction changes within the volume [S1, unofficial].
*   **When not to use:** Not suitable for integer-only environments or when exact integer arithmetic is required (use [3D Bresenham Voxel Tracing](3949-3d-bresenham-voxel-tracing.md) instead) [S1, unofficial].

## Background
The Restartable DDA for Voxel Traversal is derived from the Amanatides and Woo algorithm but extended to support state restartability for refraction [S1, unofficial]. It is used in the "Voxel Ocean" project for rendering transparent, refractive water over complex terrain, where it handles the transmission phase of light through the water volume [S1, unofficial]. The implementation is found in the `vox-ocean` repository [S1, unofficial].

## Sources
[S1] wlim33/vox-ocean, https://github.com/wlim33/vox-ocean, fetched 2026-10-27 (unofficial)
