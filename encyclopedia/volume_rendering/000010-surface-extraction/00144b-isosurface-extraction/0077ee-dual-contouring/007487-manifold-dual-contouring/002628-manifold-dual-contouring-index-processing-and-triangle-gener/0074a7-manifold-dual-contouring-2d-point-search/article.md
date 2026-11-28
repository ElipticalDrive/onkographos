# Manifold Dual Contouring - 2D Point Search

Also called: 2D Point Search

## Overview
The 2D Point Search is a procedure within Manifold Dual Contouring (MDC) that computes auxiliary points on grid faces to approximate local surface normals more accurately than the implicit function's gradient [S1]. It solves the problem where the gradient only fits the infinitesimal region and may not represent the surface geometry within the grid cell, potentially leading to incorrect 3D point identification [S1]. By capturing actual local geometry on the face, it provides suitable local flat planes for the Quadric Error Function [S1]. It differs from its siblings, Edge Contouring and Face Contouring, by specifically targeting the refinement of normals on grid faces using a line-binary search strategy [S1].

## How it works
**Inputs:** Two 1D points $p_1$ and $p_2$ located on the same grid face $f$. The occupancy function $O(\mathbf{x})$ which returns inside/outside labels.
**Outputs:** A single 2D point $p_{2D}$ on the grid face $f$.

**Data Structures:**
*   Points are represented as 3D coordinates $(x, y, z)$.
*   Rays are defined by an origin $\mathbf{o}$ and a direction vector $\mathbf{d}$.
*   Lines are defined by two points or a point and a direction.

**Parameters:**
*   Line search samples: 4 uniform samples to locate the initial interval.
*   Binary search iterations: 11 iterations within the located interval.
*   Total occupancy evaluations per 2D point: 45. This is calculated as $(4 + 11) \times 3$ evaluations for the three rays involved ($r$, $r_1$, and $r_2$) [S1].

**Algorithm Steps:**

1.  **Define Geometry:** Let line $l$ be the segment connecting $p_1$ and $p_2$. Let $m$ be the midpoint of $p_1$ and $p_2$.
2.  **Orthogonal Ray Search:** Define ray $r$ originating from $m$ with direction orthogonal to $l$. The direction is chosen towards the side where a grid vertex has a different inside/outside label compared to $m$'s label [S1].
3.  **Find Closest Crossing $q$:** Detect the closest surface-crossing point $q$ along ray $r$ using a line-binary search (see below).
    *   **Edge Case:** If $q$ coincides with $m$, set $p_{2D} = m$ and terminate [S1].
4.  **Parallel Rays:** Starting at $q$, shoot two rays $r_1$ and $r_2$ in opposite directions parallel to line $l$.
    *   Ray $r_1$ is directed towards $p_1$ from $m$.
    *   Ray $r_2$ is directed towards $p_2$ from $m$ [S1].
5.  **Find Intersections $q_1, q_2$:** Find the closest intersection point of each ray with the surface $S$, denoting them as $q_1$ (for $r_1$) and $q_2$ (for $r_2$), using line-binary search [S1].
    *   **Note:** Under the flat plane assumption, either $q_1$ or $q_2$ may coincide with $q$ [S1].
6.  **Compute Intersection:** Determine lines $l_1$ passing through $p_1$ and $q_1$, and $l_2$ passing through $p_2$ and $q_2$.
    *   If $l_1$ and $l_2$ are identical, $p_{2D}$ is the middle point $m$.
    *   Otherwise, $p_{2D}$ is the unique intersection point $u$ of $l_1$ and $l_2$ [S1].

**Line-Binary Search Implementation:**
Standard binary search cannot be used directly because it finds *any* crossing, but the algorithm requires the *closest* point from the ray's origin [S1]. The method combines uniform sampling and binary search:
1.  Sample points along the ray at uniform intervals (4 samples) to locate an initial interval where the endpoints have distinct inside/outside labels [S1].
2.  Iteratively halve the search space within that interval, ensuring endpoints retain distinct labels, for 11 iterations [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def line_binary_search(ray_origin, ray_direction, occupancy_func, num_samples=4, num_iterations=11):
    """
    Finds the closest surface intersection along a ray using line-binary search.
    Returns the point q on the ray closest to ray_origin where the surface is crossed.
    """
    # Step 1: Uniform sampling to find initial bracketing interval
    # We sample points along the ray to find two adjacent points with different occupancy
    step_size = 1.0 # Arbitrary initial step, adjusted based on grid scale if known
    prev_point = ray_origin
    prev_label = occupancy_func(prev_point)
    
    found_interval = False
    interval_start = prev_point
    interval_end = prev_point
    
    for i in range(num_samples):
        current_point = ray_origin + (i + 1) * step_size * ray_direction
        current_label = occupancy_func(current_point)
        
        if prev_label != current_label:
            # Found an interval with different labels
            interval_start = prev_point
            interval_end = current_point
            found_interval = True
            break
        
        prev_point = current_point
        prev_label = current_label

    if not found_interval:
        # Fallback or error handling if no crossing found in initial samples
        # In the context of MDC, a crossing is expected if the ray direction is valid
        return ray_origin 

    # Step 2: Binary search within the interval
    for i in range(num_iterations):
        mid_point = (interval_start + interval_end) / 2.0
        mid_label = occupancy_func(mid_point)
        
        # Determine which half contains the crossing
        # We want the closest point to ray_origin. 
        # Since we started with a bracketing interval, we shrink it.
        # To ensure we find the *closest* crossing, we must be careful.
        # The standard binary search on a bracketing interval [a, b] where O(a)!=O(b)
        # will converge to *a* crossing. Since we sampled uniformly from the origin,
        # the first interval we found is the one closest to the origin containing a crossing.
        # Within this interval, any crossing is valid, but typically there is one.
        
        if prev_label != mid_label:
            interval_end = mid_point
        else:
            interval_start = mid_point
            prev_label = mid_label # Update prev_label for next iteration logic if needed, 
                                   # though strictly we just need the bracket
            
    # Return the midpoint of the final interval as the approximation of the crossing
    return (interval_start + interval_end) / 2.0

def compute_2d_point(p1, p2, occupancy_func):
    """
    Computes the 2D point p2D on the grid face defined by 1D points p1 and p2.
    """
    # 1. Define line l and midpoint m
    l_direction = p2 - p1
    m = (p1 + p2) / 2.0
    
    # 2. Determine orthogonal direction for ray r
    # In 2D face, orthogonal to l is simple. In 3D grid face, we need the normal of the face
    # and the cross product with l_direction to get the orthogonal direction in the plane.
    # Assuming p1, p2 are on a grid face with known normal n_face.
    # orthogonal_dir = cross(n_face, l_direction)
    # Normalize orthogonal_dir.
    # Choose direction towards side with different label than m.
    m_label = occupancy_func(m)
    
    # Check labels of grid vertices to determine direction
    # This part requires access to grid vertex labels, which are implicit in the "side" description.
    # For implementation, we test both orthogonal directions and pick the one pointing to 
    # a vertex with label != m_label.
    
    # Let's assume we have determined the correct direction vector 'dir_r'
    # dir_r is orthogonal to l_direction and lies in the grid face plane.
    
    # 3. Find closest crossing q along ray r
    # Ray r: origin m, direction dir_r
    q = line_binary_search(m, dir_r, occupancy_func)
    
    # Edge case: if q coincides with m
    if distance(q, m) < epsilon:
        return m
        
    # 4. Define rays r1 and r2 parallel to l
    # r1 directed towards p1 from m
    dir_r1 = normalize(p1 - m)
    # r2 directed towards p2 from m
    dir_r2 = normalize(p2 - m)
    
    # 5. Find intersections q1 and q2
    q1 = line_binary_search(q, dir_r1, occupancy_func)
    q2 = line_binary_search(q, dir_r2, occupancy_func)
    
    # 6. Determine p2D as intersection of lines l1 (p1-q1) and l2 (p2-q2)
    # Line l1 passes through p1 and q1
    # Line l2 passes through p2 and q2
    
    # Check if lines are identical
    # Lines are identical if they are collinear and share a point.
    # In 2D face, we can solve for intersection.
    
    # Vector form:
    # L1(t) = p1 + t * (q1 - p1)
    # L2(s) = p2 + s * (q2 - p2)
    
    # Solve for t, s such that L1(t) = L2(s)
    # This is a linear system. If determinant is zero, lines are parallel.
    # If parallel, check if they are the same line.
    
    # If identical, return m
    # Else, return intersection u
    
    # Implementation detail: Solve 2x2 system in the local 2D coordinate system of the face
    # Or use 3D intersection logic with constraint that result lies on the face.
    
    # Simplified 3D intersection for coplanar lines:
    # If cross(q1-p1, q2-p2) is zero, lines are parallel.
    # If parallel, check if (p2-p1) is parallel to (q1-p1). If so, identical.
    
    v1 = q1 - p1
    v2 = q2 - p2
    
    cross_v = cross(v1, v2)
    
    if norm(cross_v) < epsilon:
        # Lines are parallel
        # Check if they are the same line
        # Check if vector between points (p2-p1) is parallel to v1
        v_between = p2 - p1
        cross_between = cross(v_between, v1)
        if norm(cross_between) < epsilon:
            return m
        else:
            # Parallel but distinct? This should not happen under flat plane assumption
            # Fallback to m or handle error
            return m
    else:
        # Lines intersect
        # Solve p1 + t*v1 = p2 + s*v2
        # t*v1 - s*v2 = p2 - p1
        # This is a system of 3 equations, 2 unknowns. Since they are coplanar, 
        # we can project to 2D or use least squares / specific 2D projection.
        
        # Project to 2D local coordinates of the face
        # Let u, v be basis vectors for the face plane.
        # p1_2d, p2_2d, q1_2d, q2_2d are 2D coordinates.
        # Solve for intersection in 2D.
        
        # Using 2D cross product method for lines defined by points
        # Line 1: p1_2d, q1_2d
        # Line 2: p2_2d, q2_2d
        
        # Intersection formula for 2D lines
        # x = ( (x1*y2 - y1*x2)*(x3-x4) - (x1-x2)*(x3*y4 - y3*x4) ) / denom
        # y = ( (x1*y2 - y1*x2)*(y3-y4) - (y1-y2)*(x3*y4 - y3*x4) ) / denom
        
        # Map back to 3D
        return u
```

**Pitfalls:**
*   Standard binary search fails because it finds any crossing, not the closest one from the origin [S1]. The line-binary search combination is required [S1].
*   Precision issues may arise if $q$ coincides with $m$ or if lines $l_1$ and $l_2$ are nearly parallel [S1].

## Operating characteristics
*   **Time Complexity:** The algorithm performs a fixed number of occupancy function evaluations. For each 2D point, it performs 45 evaluations (4 line search samples + 11 binary search iterations) $\times$ 3 rays [S1].
*   **Measured Performance:** At a resolution of $128^3$, processing approximately 33,000 surface-intersecting faces in parallel on a 12GB VRAM GPU takes 1.3 seconds total [S1]. A single occupancy evaluation takes approximately 0.03 seconds [S1].
*   **Scalability:** The method is highly parallelizable on GPU, as all surface-crossing faces can be processed simultaneously [S1].
*   **Strengths:** Captures actual local geometry within grid faces, providing better normal approximations than gradients [S1].
*   **Limitations:** Requires multiple occupancy evaluations per point (45), which can be costly if the occupancy function is expensive [S1].

## Background
The 2D Point Search is part of the Occupancy-Based Dual Contouring method introduced in "Occupancy-Based Dual Contouring" [S1]. It addresses limitations in using implicit function gradients for normal estimation in dual contouring schemes. The method was developed to improve mesh quality by ensuring the local flat planes used in the Quadric Error Function accurately reflect the surface geometry.

## Sources
[S1] Occupancy-Based Dual Contouring, https://arxiv.org/abs/2409.13418v1, fetched 2026-10-26
