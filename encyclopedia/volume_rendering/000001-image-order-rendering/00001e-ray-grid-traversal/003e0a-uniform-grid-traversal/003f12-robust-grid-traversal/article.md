# Robust Grid Traversal

Also called: Robust Ray Traversal

## Overview
Robust Grid Traversal is a modification to standard Digital Differential Analyzer (DDA) grid traversal designed to prevent numerical robustness errors where valid intersections are missed due to floating-point precision limits. It solves the problem of rejecting valid intersection candidates simply because their computed parameter falls outside the bounds of the currently visited cell due to rounding errors. Unlike traditional methods that discard such intersections, this method retains the closest intersection parameter globally and terminates only when the ray's maximum parameter for the current cell exceeds that retained value.

## How it works
**Inputs and Outputs:**
- **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$ (normalized), grid bounds, cell size, and a list of primitives (e.g., triangles) associated with each cell.
- **Output:** The closest intersection point $\mathbf{p}$ and its parameter $t_{hit}$, or null if no intersection is found.

**Data Structures:**
- **Ray State:**
  - $t_{min}$: Current cell entry parameter.
  - $t_{max}$: Current cell exit parameter.
  - $t_{hit}$: The parameter of the closest valid intersection found so far. Initialized to $\infty$.
- **Grid:** Uniform grid mapping cells to primitive lists.

**Algorithm Steps:**
1. **Initialization:** Compute the initial $t_{min}$ and $t_{max}$ for the first cell entered by the ray. Set $t_{hit} = \infty$.
2. **Traversal Loop:** While $t_{min} < t_{hit}$:
   a. **Process Cell:** For each primitive in the current cell, compute the intersection parameter $t_{prim}$.
   b. **Update Hit:** If $t_{prim} < t_{hit}$, update $t_{hit} = t_{prim}$. [G] Note: Standard intersection tests should be used here.
   c. **Advance Ray:** Compute the next cell boundaries. Update $t_{min}$ and $t_{max}$ for the next cell.
   d. **Check Termination:** If $t_{max} \ge t_{hit}$, terminate and return the intersection corresponding to $t_{hit}$.
3. **Return:** If the loop completes without finding an intersection, return null.

**Key Modification:**
The critical distinction from standard DDA is the termination condition and the handling of intersection candidates. In standard DDA, an intersection computed for a primitive in cell $C_i$ is often discarded if its parameter $t$ is not within the spatial bounds of $C_i$ (i.e., $t < t_{min}^{C_i}$ or $t > t_{max}^{C_i}$). Robust Grid Traversal retains $t_{hit}$ regardless of whether it falls within the current cell's spatial bounds. The traversal continues until the ray's exit parameter from the current cell ($t_{max}$) exceeds the closest hit found so far ($t_{hit}$). This ensures that even if precision errors place a valid intersection slightly outside the current cell's computed bounds, it is not discarded prematurely.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def robust_grid_traversal(ray_origin, ray_direction, grid, primitives):
    # Initialize ray state
    t_min, t_max = compute_first_cell_bounds(ray_origin, ray_direction, grid)
    t_hit = float('inf')
    
    # Traverse cells
    while t_min < t_hit:
        # Get primitives in the current cell
        cell_primitives = grid.get_primitives_at_current_cell()
        
        # Check intersections with primitives
        for prim in cell_primitives:
            t_prim = intersect(ray_origin, ray_direction, prim)
            if t_prim is not None and t_prim < t_hit:
                t_hit = t_prim
        
        # Advance to next cell
        t_min, t_max = compute_next_cell_bounds(ray_origin, ray_direction, grid)
        
        # Termination check: if the ray has passed the closest hit, stop
        if t_max >= t_hit:
            break
            
    if t_hit == float('inf'):
        return None
    else:
        return compute_intersection_point(ray_origin, ray_direction, t_hit)
```

**Parameters:**
- No specific epsilon parameters are required, as the method avoids scene-dependent epsilon tests [S1].

**Edge Cases and Pitfalls:**
- **Precision Errors:** Standard DDA may reject intersections on grid bounding boxes or implied planes if they fall outside the current cell due to numerical errors [S1]. This method mitigates that by keeping the closest intersection parameter even if it falls outside the current cell [S1].
- **Termination:** The loop must terminate when $t_{max} \ge t_{hit}$ to avoid infinite loops or unnecessary traversal.

## Subtopics

### [Cell-Local Ray Traversal](19343-cell-local-ray-traversal.md)
Cell-Local Ray Traversal is a ray-grid traversal method that computes cell-exit points and traversal distances using a normalized coordinate system local to each cell, preventing numerical drift and angle errors that accumulate when using global or canvas coordinates.

## Operating characteristics
- **Time Complexity:** Same as standard DDA, $O(N)$ where $N$ is the number of cells traversed, plus intersection tests for primitives in visited cells.
- **Memory Complexity:** Same as standard DDA.
- **Performance:** No significant performance overhead compared to standard DDA, as it avoids the need for complex epsilon tuning [S1].
- **Strengths:** Eliminates the need for scene-dependent epsilon tests, which are hard to choose correctly [S1]. Prevents missed intersections due to numerical robustness errors [S1].
- **Limitations:** Requires storing and comparing $t_{hit}$ against $t_{max}$, but this is minimal overhead.
- **When to Use:** When high numerical precision is critical, or when working with low-precision floating-point formats on mobile hardware [S1]. When standard DDA misses intersections due to precision errors.

## Background
Robust Grid Traversal was introduced to address numerical robustness issues in grid-based ray tracing. While traditional methods like those by Cleary and Wyvill [CW88] are susceptible to these errors, Robust Grid Traversal provides a simple modification to existing methods. It is used by some researchers in the field [PH04], although many sources still discard intersections [Shi00, Suf07]. The method is particularly relevant in contexts where numerical stability is paramount, such as in mobile hardware or high-precision rendering applications.

## Sources
[S1] Compact, Fast and Robust Grids for Ray Tracing, https://doi.org/10.1111/j.1467-8659.2008.01262.x, fetched 2026-10-24
