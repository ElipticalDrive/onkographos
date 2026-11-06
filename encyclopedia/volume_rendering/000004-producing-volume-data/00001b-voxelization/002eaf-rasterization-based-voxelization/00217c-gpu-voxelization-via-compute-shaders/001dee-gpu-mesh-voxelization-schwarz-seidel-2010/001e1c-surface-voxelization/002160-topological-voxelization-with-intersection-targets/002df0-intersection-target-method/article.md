# Intersection Target Method

Also called: 2D 8-Connected, 4-Separating Voxelization, Intersection Target Voxelization

## Overview
The Intersection Target Method is a topological voxelization strategy for effectively 1D input (lines or curves) that guarantees 8-connectivity (adjacent voxels share an edge or corner) and 4-separability (no 4-connected path of background voxels separates foreground components) [S1, unofficial]. It differs from geometric proximity methods by including a voxel if and only if the input intersects a specific geometric "target" shape within that voxel, ensuring topological correctness through the connectivity of these targets [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A parameterized continuous curve $c(t) = (f_x(t), f_y(t))$ for $t \in [0, 1]$.
- Output: A set of voxels (pixels in 2D) $S_d$ such that the voxelization is 8-connected and 4-separable.

**Data Structures:**
The intersection target is defined within each pixel grid cell [S1, unofficial]. For the standard 8-connected variant, the target consists of two line segments: a horizontal segment centered at the pixel center extending to the midpoints of the left and right edges, and a vertical segment centered at the pixel center extending to the midpoints of the top and bottom edges [S1, unofficial]. This forms a "crosshairs" shape. An alternative is a "diamond" target connecting the midpoints of the four edges, corresponding to the "diamond exit" rule in OpenGL/DirectX [S1, unofficial].

**Steps:**
1. Iterate over each pixel $V$ in the grid covering the input bounds.
2. Define the intersection target $T_V$ for $V$ (crosshairs or diamond) [S1, unofficial].
3. Test if the input curve $c(t)$ intersects any part of $T_V$ [S1, unofficial].
4. If an intersection exists, add $V$ to the output set $S_d$ [S1, unofficial].

**Formulas and Thiness Condition:**
To ensure "thiness" (at most one voxel per horizontal layer for curves oriented closer to the y-axis), the curve must satisfy $|df_y/df_x| \ge 1$ for all $t$ [S1, unofficial]. This ensures the curve is steeper than 45 degrees [S1, unofficial]. Under this condition, if the curve intersects the horizontal segment of the crosshairs target at $(x_0, y_0)$, the x-coordinate is bounded by $[x_0 - 1/2, x_0 + 1/2]$, preventing multiple voxels in the same horizontal layer from the same continuous segment [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def intersection_target_voxelization(curve_fn, grid_bounds):
    """
    Voxelizes a 1D curve using the 8-connected, 4-separating intersection target method.
    
    Args:
        curve_fn: Callable c(t) -> (x, y) for t in [0, 1].
        grid_bounds: Tuple (x_min, y_min, x_max, y_max) defining the grid area.
        
    Returns:
        A set of voxel coordinates (x, y) included in the voxelization.
    """
    voxels = set()
    x_min, y_min, x_max, y_max = grid_bounds
    start_x, end_x = int(math.floor(x_min)), int(math.ceil(x_max))
    start_y, end_y = int(math.floor(y_min)), int(math.ceil(y_max))
    
    for x in range(start_x, end_x):
        for y in range(start_y, end_y):
            cx, cy = x + 0.5, y + 0.5
            
            # Check intersection with horizontal segment: y = cy, x in [x, x+1]
            t_h = solve_fy_eq_target(curve_fn, cy, 0.0, 1.0)
            intersects = False
            for t in t_h:
                px, _ = curve_fn(t)
                if x <= px <= x + 1:
                    intersects = True
                    break
            if intersects:
                voxels.add((x, y))
                continue
                
            # Check intersection with vertical segment: x = cx, y in [y, y+1]
            t_v = solve_fx_eq_target(curve_fn, x, 0.0, 1.0)
            for t in t_v:
                _, py = curve_fn(t)
                if y <= py <= y + 1:
                    intersects = True
                    break
            if intersects:
                voxels.add((x, y))
                
    return voxels

def solve_fy_eq_target(curve_fn, target_y, t_min, t_max):
    # [G] The sources do not specify the exact root-finding algorithm.
    # In practice, this solves f_y(t) = target_y for t in [t_min, t_max]
    # using analytical inversion for polynomials or bisection/Newton's method.
    return []

def solve_fx_eq_target(curve_fn, target_x, t_min, t_max):
    # [G] The sources do not specify the exact root-finding algorithm.
    # Solves f_x(t) = target_x for t in [t_min, t_max].
    return []
```

**Edge Cases:**
- Point-like input (effective dimension 0) is ignored unless it falls exactly on the target, which is rare for continuous curves [S1, unofficial].
- Curves not satisfying the thinness condition may produce thicker results [S1, unofficial].

## Subtopics

### 2D 4-Connected, 8-Separating Voxelization
A variant that guarantees 4-connectivity (adjacent voxels share an edge) and 8-separability [S1, unofficial]. It uses a cross-diagonal intersection target consisting of two curves connecting opposite corners of the pixel (e.g., top-left to bottom-right, and top-right to bottom-left), meeting at the pixel center [S1, unofficial]. This geometry ensures that any continuous path exiting the pixel must pass through the targets of the 4-neighbors, enforcing 4-connectivity [S1, unofficial]. An alternative is a square target pushed to the pixel boundary, but the cross-diagonal is preferred for thinness [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Guarantees topological connectivity (8-connected/4-separable) for 1D input [S1, unofficial]. Simple intersection targets (crosshairs or diamond) are easy to implement in shaders [S1, unofficial].
- **Limitations:** Only suitable for effectively 1D input [S1, unofficial]. The diamond variant may include unnecessary voxels for aesthetic reasons but reduces connectivity guarantees slightly compared to the crosshairs variant in specific corner cases [S1, unofficial].
- **When to use:** Not given in the sources.

## Background
The Intersection Target Method was introduced by Laine et al. in "A Topological Approach to Voxelization" (2013) [S1, unofficial]. It builds on the concept of topological voxelization where connectivity is enforced by the geometry of the intersection targets rather than geometric proximity [S1, unofficial]. The method leverages the properties of 8-connectivity and 4-separability, which are equivalent in 2D grid topology [S1, unofficial].

## Sources
[S1] A Topological Approach to Voxelization - NVIDIA, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
