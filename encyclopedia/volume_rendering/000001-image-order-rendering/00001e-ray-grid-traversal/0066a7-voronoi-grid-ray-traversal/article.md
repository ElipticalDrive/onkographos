# Voronoi Grid Ray Traversal

Also called: 3D Voronoi Grid Ray Tracing, Ray casting through Voronoi cells, Straight path calculation through 3D Voronoi grid, Straight path traversal in Voronoi grid, Voronoi ray marching

## Overview
Voronoi Grid Ray Traversal is a method for computing straight-line paths through an unstructured 3D Voronoi tessellation to determine radiation field properties in radiative transfer (RT) simulations [S1]. It solves the problem of accurately tracking photon energy absorption and path length within irregular, convex polyhedral cells, avoiding the directional spread errors and "fuzzy" distance concepts inherent in Delaunay edge approximation methods [S1]. Unlike uniform grid traversal [Uniform Grid Traversal](9652-uniform-grid-traversal.md) which relies on axis-aligned voxel boundaries, this method calculates precise intersection points with arbitrary planar cell walls defined by the perpendicular bisectors of Delaunay edges [S1].

## How it works
The algorithm advances a ray through a sequence of Voronoi cells by calculating the intersection of the ray line with the planar boundaries of the current cell. It relies on the topological neighbor list of the current cell rather than explicit vertex/edge geometry [S1].

### Inputs and Outputs
**Inputs:**
*   Ray origin $\mathbf{r}_0$ and direction unit vector $\mathbf{k}$ [S1].
*   Domain boundaries $\mathbf{D}_{min}$ and $\mathbf{D}_{max}$ defining the cuboidal spatial extent [S1].
*   Site positions $\{\mathbf{p}_m\}_{m=1}^M$ defining the Voronoi tessellation [S1].
*   Neighbor indices $\{m_{n,i}\}$ for each site $n$, where positive indices denote neighboring sites and negative indices denote domain walls [S1].
*   Current point $\mathbf{r}$ (initially $\mathbf{r}_0$) and current cell index $m_r$ (initially determined by point location) [S1].

**Outputs:**
*   A sequence of path segments, each defined by the entry point, exit point, distance traveled within the cell, and the index of the next cell [S1].

### Data Structures
*   **Site Positions:** Array of 3D vectors $\mathbf{p}_m$ [S1].
*   **Neighbor Lists:** For each cell $m_r$, a list of integers $\{m_i\}$ representing adjacent cells or walls [S1].
*   **Domain Boundaries:** Two 3D vectors $\mathbf{D}_{min}, \mathbf{D}_{max}$ [S1].

### Algorithm Steps

1.  **Initialization:**
    *   Set current point $\mathbf{r} = \mathbf{r}_0$ [S1].
    *   Determine initial cell index $m_r$ by finding the site nearest to $\mathbf{r}_0$ using function $C(\mathbf{r})$ [S1].

2.  **Loop (Exit Point Calculation):**
    *   Retrieve neighbor indices $\{m_i\}$ for current cell $m_r$ [S1].
    *   For each neighbor $m_i$, calculate the intersection parameter $s_i$ where the ray line $L(s) = \mathbf{r} + s\mathbf{k}$ intersects the plane separating cell $m_r$ and neighbor $m_i$ [S1].
    *   **Case 1: Regular Neighbor ($m_i > 0$):**
        *   Calculate normal vector: $\mathbf{n} = \mathbf{p}(m_i) - \mathbf{p}(m_r)$ [S1].
        *   Calculate point on plane: $\mathbf{p}_{plane} = \frac{\mathbf{p}(m_i) + \mathbf{p}(m_r)}{2}$ [S1].
        *   Calculate intersection parameter:
            $$
            s_i = \frac{\mathbf{n} \cdot (\mathbf{p}_{plane} - \mathbf{r})}{\mathbf{n} \cdot \mathbf{k}}
            $$
            [S1]
    *   **Case 2: Domain Wall ($m_i < 0$):**
        *   Use the specific normal $\mathbf{n}$ and point $\mathbf{p}_{plane}$ for the domain boundary. For example, for the left wall ($x$-min):
            $$
            \mathbf{n} = (-1, 0, 0), \quad \mathbf{p}_{plane} = (D_{min,x}, 0, 0)
            $$
            $$
            s_i = \frac{D_{min,x} - r_x}{k_x}
            $$
            [S1]
    *   **Optimization:** Do not accumulate all $s_i$. Track only the smallest non-negative value $s_q = \min \{ s_i \mid s_i > 0 \}$ [S1].
    *   **Optimization:** Skip calculation if $\mathbf{n} \cdot \mathbf{k} \le 0$. Since $\mathbf{p}(m_r)$ and $\mathbf{r}$ are on the same side of the plane, the numerator $\mathbf{n} \cdot (\mathbf{p}_{plane} - \mathbf{r})$ is always non-negative. If $\mathbf{n} \cdot \mathbf{k} \le 0$, the intersection is behind the ray or parallel, so $s_i$ cannot be the smallest positive exit [S1].

3.  **Termination Check:**
    *   If no positive $s_i$ exists (i.e., no valid exit point found), handle computational inaccuracy:
        *   Do not add a path segment.
        *   Advance $\mathbf{r}$ by an infinitesimal distance $\epsilon = \|\mathbf{D}_{max} - \mathbf{D}_{min}\|$ in direction $\mathbf{k}$ [S1].
        *   Recalculate $m_r = C(\mathbf{r})$ [S1].
        *   Continue loop.
    *   If $s_q$ corresponds to a domain wall, terminate the traversal [S1].

4.  **Update:**
    *   Calculate exit point $\mathbf{q} = \mathbf{r} + s_q \mathbf{k}$ [S1].
    *   Record path segment from $\mathbf{r}$ to $\mathbf{q}$ with length $s_q$ [S1].
    *   Update current point $\mathbf{r} = \mathbf{q}$ [S1].
    *   Update current cell index $m_r$ to the neighbor index $m_q$ corresponding to $s_q$ [S1].
    *   Repeat Loop.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voronoi_ray_traversal(r0, k, domain_min, domain_max, sites, neighbors):
    """
    r0: 3D vector, ray origin
    k: 3D vector, ray direction (unit vector)
    domain_min: 3D vector, minimum domain boundary
    domain_max: 3D vector, maximum domain boundary
    sites: list of 3D vectors, site positions p_m
    neighbors: list of lists, neighbors[m] contains indices of adjacent cells/walls
    """
    
    # Helper to find cell containing point r
    def find_cell(r):
        # Find index m minimizing ||sites[m] - r||
        # [G] Implementation detail: use spatial index or brute force for small M
        min_dist = float('inf')
        best_m = -1
        for m, p in enumerate(sites):
            dist = sum((p[i] - r[i])**2 for i in range(3))
            if dist < min_dist:
                min_dist = dist
                best_m = m
        return best_m

    # Initialization
    r = list(r0)
    mr = find_cell(r)
    
    path_segments = []
    
    while True:
        current_neighbors = neighbors[mr]
        s_min = float('inf')
        m_exit = -1
        
        for mi in current_neighbors:
            if mi > 0:
                # Regular neighbor
                p_mr = sites[mr]
                p_mi = sites[mi]
                
                # Normal vector n = p_mi - p_mr
                n = [p_mi[i] - p_mr[i] for i in range(3)]
                
                # Dot product n . k
                n_dot_k = sum(n[i] * k[i] for i in range(3))
                
                # Optimization: if n . k <= 0, intersection is behind or parallel
                if n_dot_k <= 0:
                    continue
                
                # Point on plane p_plane = (p_mi + p_mr) / 2
                p_plane = [(p_mi[i] + p_mr[i]) / 2.0 for i in range(3)]
                
                # Numerator n . (p_plane - r)
                num = sum(n[i] * (p_plane[i] - r[i]) for i in range(3))
                
                # si = num / (n . k)
                si = num / n_dot_k
                
                if si > 0 and si < s_min:
                    s_min = si
                    m_exit = mi
            else:
                # Domain wall (mi < 0)
                # Determine wall based on negative index convention
                # [G] Assuming specific mapping of negative indices to walls
                # For example, -1: x-min, -2: x-max, -3: y-min, etc.
                # This part depends on the specific implementation of 'neighbors'
                # Here we assume a generic wall calculation function
                wall_result = calculate_wall_intersection(mi, r, k, domain_min, domain_max)
                if wall_result is not None:
                    si, m_exit = wall_result
                    if si > 0 and si < s_min:
                        s_min = si
                        # m_exit might be a special wall marker, but for loop logic
                        # we just need to know it's a wall to terminate later
                        # In this pseudocode, we treat wall exit as a valid s_min
                        pass

        if s_min == float('inf'):
            # No exit point found (computational inaccuracy)
            # Advance by epsilon
            domain_size = sum((domain_max[i] - domain_min[i])**2 for i in range(3))
            epsilon = domain_size ** 0.5
            r = [r[i] + epsilon * k[i] for i in range(3)]
            mr = find_cell(r)
            continue
            
        # Exit point found
        q = [r[i] + s_min * k[i] for i in range(3)]
        
        # Record segment
        path_segments.append({
            'entry': list(r),
            'exit': list(q),
            'distance': s_min,
            'next_cell': m_exit
        })
        
        # Check if exit is a domain wall
        # [G] Logic to determine if m_exit corresponds to a wall
        if m_exit < 0:
            break
            
        # Update state
        r = q
        mr = m_exit
        
    return path_segments

def calculate_wall_intersection(wall_index, r, k, domain_min, domain_max):
    """
    Calculates intersection with domain walls.
    Returns (s, wall_marker) or None.
    """
    # Example mapping: -1: x-min, -2: x-max, -3: y-min, -4: y-max, -5: z-min, -6: z-max
    if wall_index == -1:
        if k[0] == 0: return None
        s = (domain_min[0] - r[0]) / k[0]
        return (s, -1)
    elif wall_index == -2:
        if k[0] == 0: return None
        s = (domain_max[0] - r[0]) / k[0]
        return (s, -2)
    elif wall_index == -3:
        if k[1] == 0: return None
        s = (domain_min[1] - r[1]) / k[1]
        return (s, -3)
    elif wall_index == -4:
        if k[1] == 0: return None
        s = (domain_max[1] - r[1]) / k[1]
        return (s, -4)
    elif wall_index == -5:
        if k[2] == 0: return None
        s = (domain_min[2] - r[2]) / k[2]
        return (s, -5)
    elif wall_index == -6:
        if k[2] == 0: return None
        s = (domain_max[2] - r[2]) / k[2]
        return (s, -6)
    else:
        return None
```

### Parameters and Edge Cases
*   **Epsilon:** The infinitesimal advance $\epsilon$ is defined as the diagonal length of the domain $\|\mathbf{D}_{max} - \mathbf{D}_{min}\|$ [S1].
*   **Parallel Planes:** If $\mathbf{n} \cdot \mathbf{k} = 0$, $s_i$ becomes $\pm \infty$. In IEEE 754 arithmetic, this value is never selected as the minimum positive $s_q$, so no special test is required [S1].
*   **Point Location:** The function $C(\mathbf{r})$ finds the site closest to $\mathbf{r}$. This is computationally expensive if done naively; optimized implementations use a coarse spatial grid (blocks) to restrict the search to candidate cells [S1].

## Subtopics

### [Point location in Voronoi grid](22962-point-location-in-voronoi-grid.md)
The method for identifying the specific Voronoi cell containing a query point $\mathbf{r}$ by finding the nearest site. It uses a coarse spatial grid to optimize the search, avoiding complex tree structures.

## Operating characteristics
*   **Performance:** Shooting photon packages through a 3D Voronoi grid is approximately three times slower than an equivalent octree grid with the same number of cells [S1].
*   **Topology:** For sites chosen randomly from a uniform distribution, the expected number of nearest neighbors (cells sharing a face) is 15.54 [S1]. This determines the average number of plane intersection calculations per cell traversal.
*   **Accuracy:** The method calculates high-precision straight path segments inside each cell, ensuring accurate energy absorption tracking and high-resolution image production, unlike Delaunay edge approximation methods which suffer from directional spread [S1].
*   **Limitations:** The method requires pre-computed neighbor lists and site positions. It is less efficient than structured grid methods due to the irregular geometry and the cost of point location for initialization or error recovery [S1].

## Background
The method was presented by Camps et al. in 2013 for use in the SKIRT dust radiative transfer code [S1]. It addresses challenges in RT simulations using unstructured grids, such as those derived from smoothed particle hydrodynamics (SPH) or adaptive mesh refinement, where regular voxel traversal is not applicable [S1]. The implementation relies on the open-source library Voro++ to obtain Voronoi grid cell properties based solely on generating points [S1].

## Sources
[S1] Using 3D Voronoi grids in radiative transfer simulations, https://doi.org/10.1051/0004-6361/201322281/pdf, fetched 2026-10-25
