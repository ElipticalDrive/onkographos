# GPU-based Fast Ray Casting for Metaballs

Also called: Nishita and Nakamae's Algorithm

## Overview
This method renders particle-based implicit surfaces (metaballs) directly on the GPU by evaluating the isosurface condition per pixel, avoiding polygonization and expensive precomputation or acceleration structures [S1]. It solves the problem of high rendering costs for large numbers of moving metaballs by maintaining a compact, dynamically updated list of contributing metaballs along each viewing ray and using Bézier Clipping to solve the resulting polynomial equations [S1]. Unlike [Volumetric Isosurface Rendering with Intersection Refinement](958-volumetric-isosurface-rendering-with-intersection-refinement.md) which relies on grid-based density evaluation, this method is exact for the chosen polynomial density functions and preserves small objects that might be missed by grid-based approaches [S1].

## How it works
**Inputs:**
- $N$ metaballs, each defined by center $\mathbf{p}_i \in \mathbb{R}^3$, effective radius $R_i$, density coefficient $q_i$, and a density function $f_i(r)$ with finite support [S1].
- Isosurface threshold $T$ [S1].
- Camera parameters defining the viewing ray $\mathbf{R}(t) = \mathbf{o} + t\mathbf{d}$, where $t$ is the distance from the viewpoint [S1].

**Outputs:**
- Color for each pixel, derived from the isosurface closest to the viewpoint [S1].

**Data Structures:**
- **ID List:** A fixed-size array storing the indices of metaballs contributing to the current interval. The capacity is limited to $K = 4 \times N_{buf} - 2$, where $N_{buf}$ is the number of depth peeling buffers [S1]. In experiments, $K=14$ was used [S1].
- **Stored $t$:** The $t$-parameter of the last processed ray-sphere intersection, marking the start of the current interval [S1].

**Density Function:**
The density function $f_i$ is a function of $r = \|\mathbf{x} - \mathbf{p}_i\|$ and monotonically decreases as $r$ increases. It has finite support $R_i$, such that $f_i(r) = 0$ for $r \ge R_i$ [S1]. The method supports quartic and degree-six polynomials [S1].

**The Isosurface Equation:**
The surface is defined by points $\mathbf{x}$ satisfying:
$$ f(\mathbf{x}) = \sum_{i=0}^{N-1} q_i f_i(\|\mathbf{x} - \mathbf{p}_i\|) - T = 0 \quad (1) $$
The normal vector at $\mathbf{x}$ is derived from $-\nabla f(\mathbf{x})$ [S1].

**Algorithm Steps:**
1.  **Initialize:** Set the stored $t$ to the near plane or previous intersection. Initialize the ID list as empty.
2.  **Depth Peeling (Ray-Sphere Intersection):**
    - Render the effective spheres (spheres with radius $R_i$) using depth peeling to find intersections from front to back [S1].
    - For each pixel, if a new intersection at $t_k$ with metaball ID $B_k$ is found:
        - If the ID list is not full and $B_k$ is not already in the list, insert $B_k$ into the ID list [S1].
        - If $B_k$ is already in the list, discard the fragment (optimization: in 32-bit float, duplicate checks can be omitted without visible artifacts) [S1].
        - Perform a **Ray-Isosurface Test** in the interval $[t_{k-1}, t_k]$ using the metaballs currently in the ID list plus the new $B_k$ (if added) [S1].
        - Update the stored $t$ to $t_k$ [S1].
        - If an intersection is found in the test, terminate depth peeling for this pixel and proceed to shading [S1].
        - If the ID list capacity $K$ is exceeded during insertion, treat the current ray-sphere intersection as a ray-isosurface intersection immediately to prevent holes [S1].
3.  **Ray-Isosurface Test (Bézier Clipping):**
    - Construct a 1D Bézier curve $D(u)$ representing the density function along the ray segment.
    - Solve $d(u) = 0$ using Bézier Clipping [S1].
    - **Single Case:** If only one effective sphere intersects the interval, perform a standard ray-sphere intersection test. The radius can be precomputed [S1].
    - **Multiple Case:** If two or more effective spheres intersect, construct the Bézier form equation and solve it [S1].
4.  **Shading:**
    - Evaluate the surface normal at the hit point using the gradient of the density function [S1].
    - Compute color using local lighting models (ambient, diffuse, specular) [S1].

**Bézier Clipping Solver:**
A degree-$M$ Bézier curve is defined as:
$$ \mathbf{D}(u) = (u, d(u)) = \sum_{i=0}^M \mathbf{D}_i \mathbf{B}_i^M(u), \quad (2) $$
where $\mathbf{B}_i^M(u) = \binom{M}{i} u^i (1-u)^{M-i}$ are Bernstein polynomials, and control points $\mathbf{D}_i = (i/M, d_i)$ [S1].

The algorithm iteratively clips the curve in the range $[u_{min}, u_{max}]$, where $u_{min}$ and $u_{max}$ are the points where the convex hull of the control points intersects the $u$-axis (i.e., where the $y$-coordinate of the control points changes sign or bounds the zero crossing) [S1].
- Use de Casteljau's algorithm to subdivide the curve and find the new bounding box [S1].
- If the range $[u_{min}, u_{max}]$ does not converge below a threshold, split the curve in two and recurse [S1].
- **Pitfall:** Roots may be misdetected when the convex hull barely intersects the axis. Mitigation: slightly increase the clipping range heuristic [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_metaballs(metaballs, camera, threshold, max_peel_iterations):
    # Initialize depth peeling buffers
    # Each buffer stores: (t, id_list) for each pixel
    # t: float, distance to last intersection
    # id_list: list of metaball indices
    
    prev_buffers = initialize_buffers()
    current_buffers = initialize_buffers()
    
    for k in range(1, max_peel_iterations + 1):
        # Step 1: Depth Peeling - Find k-th ray-sphere intersection
        # Render effective spheres. For each pixel, compute intersection t_k and metaball ID B_k
        intersections = depth_peel_step(metaballs, camera, prev_buffers, k)
        
        # Step 2: Update ID Lists and Perform Isosurface Test
        for each pixel p:
            if intersections[p] is None:
                current_buffers[p] = prev_buffers[p]
                continue
                
            t_k, B_k = intersections[p]
            t_prev = prev_buffers[p].t
            id_list = prev_buffers[p].id_list
            
            # Check capacity
            if len(id_list) >= K:
                # Fallback: treat ray-sphere intersection as isosurface intersection
                # This prevents holes in dense groups
                hit_t, hit_u = ray_sphere_isosurface_test(t_prev, t_k, id_list + [B_k], camera, metaballs)
                if hit_t is not None:
                    shade_pixel(p, hit_t, hit_u, metaballs)
                    current_buffers[p] = prev_buffers[p] # Mark as done
                    continue
            
            # Insert B_k if not present (optimization: skip check for 32-bit float)
            if B_k not in id_list:
                id_list.append(B_k)
            
            # Perform Ray-Isosurface Test in [t_prev, t_k]
            hit_t, hit_u = ray_isosurface_test(t_prev, t_k, id_list, camera, metaballs)
            
            if hit_t is not None:
                shade_pixel(p, hit_t, hit_u, metaballs)
                current_buffers[p] = prev_buffers[p] # Mark as done
            else:
                # Update state for next iteration
                current_buffers[p].t = t_k
                current_buffers[p].id_list = id_list
        
        prev_buffers = current_buffers
        # Check if all pixels are done to break early

def ray_isosurface_test(t_start, t_end, id_list, camera, metaballs):
    # Construct the 1D density function d(u) along the ray segment
    # u in [0, 1] maps to t in [t_start, t_end]
    # d(u) = sum(q_i * f_i(||R(t_start + u*(t_end-t_start)) - p_i||)) - T
    
    # Convert polynomial coefficients to Bézier form
    bezier_coeffs = construct_bezier_curve(id_list, t_start, t_end, camera, metaballs)
    
    # Solve d(u) = 0 using Bézier Clipping
    return bezier_clipping_solve(bezier_coeffs, t_start, t_end)

def bezier_clipping_solve(coeffs, t_start, t_end):
    # Iteratively clip the range [u_min, u_max]
    # u_min, u_max are derived from the convex hull of control points
    # where the hull crosses the u-axis (y=0)
    
    u_min, u_max = get_convex_hull_bounds(coeffs)
    
    while (u_max - u_min) > threshold:
        # Subdivide using de Casteljau's algorithm to narrow the range
        new_coeffs = de_casteljau_subdivision(coeffs, u_min, u_max)
        u_min, u_max = get_convex_hull_bounds(new_coeffs)
        
        # If range doesn't shrink, split curve
        if u_max - u_min == 0:
            mid = (u_min + u_max) / 2
            left_coeffs, right_coeffs = split_bezier_curve(coeffs, mid)
            # Recursively solve both parts
            res_left = bezier_clipping_solve(left_coeffs, t_start, t_end)
            if res_left is not None: return res_left
            return bezier_clipping_solve(right_coeffs, t_start, t_end)
            
    # Return the root within the final range
    return t_start + u_min * (t_end - t_start), u_min
```

## Subtopics

### [Real-Time Metaball Ray Casting with Fragment Lists](9945-real-time-metaball-ray-casting-with-fragment-lists.md)
A real-time ray casting method for rendering particle-based medium representations (metaballs) that builds per-pixel lists of relevant metaballs, incrementally constructs a piecewise polynomial approximation of summed densities along rays, and finds intersections with the isosurface. It builds on the core ray-casting principles but focuses on fragment list management for real-time performance [9945-real-time-metaball-ray-casting-with-fragment-lists.md].

### Metaball Culling via Occlusion Queries
Reduces the number of rendered metaballs by culling effective spheres hidden from the viewpoint by 'kernel spheres' (spheres making up the isosurface). It renders kernel spheres first, saves ray parameters to a texture, then renders effective spheres using occlusion queries to determine which were drawn, culling those that were not. This can reduce the number of metaballs to be rendered to 10-20% in dense groups [S1].

### Ray-Isosurface Test Special Case Handling
Procedural rules for handling degenerate or dense ray-metaball intersections during depth peeling to prevent duplicate IDs, holes, or occlusion errors.
- **Duplicate IDs:** When multiple effective spheres intersect at the same point, compare ray parameters and return only one ID. Checking for duplicates in the list drastically increases cost; omitting it in 32-bit float produces no visible artifacts [S1].
- **Capacity Exceeded:** When dense groups exceed the ID list upper limit $K = 4 \times N_{buf} - 2$ (e.g., $K=14$), treat the ray-sphere intersection as a ray-isosurface intersection to prevent holes [S1].
- **Delayed Intersection:** If the ray intersects the isosurface only after crossing other metaballs, repeat depth peeling until termination, provided $K$ is not exceeded [S1].

### Depth Peeling for Ray-Sphere Intersection
A technique to obtain ray-sphere intersections in parallel from front to back by drawing effective spheres and storing IDs and ray parameters into a texture, discarding fragments whose parameters are less than previous ones [S1].

## Operating characteristics
- **Time Complexity:** Depends on screen resolution and the number of depth peeling iterations. The most expensive part is Bézier Clipping [S1].
- **Memory Complexity:** Limited by GPU memory for depth peeling buffers and the ID list per pixel ($K$ entries) [S1].
- **Performance:** An animation with 1,000 moving metaballs was rendered in 640×480 at 23.1 fps [S1].
- **Strengths:** No polygonization artifacts; preserves small objects; no acceleration structures needed; handles high-degree polynomials (quartic, degree-six) accurately [S1].
- **Limitations:** Performance degrades with very dense groups if $K$ is exceeded (requires fallback); Bézier Clipping is computationally expensive; texture-based precomputation of Bézier coefficients showed little coherence and hurt performance [S1].
- **When to use:** Real-time visualization of particle-based simulations (fluid, molecular dynamics) where small details matter and polygonization is unsuitable [S1].
- **When not to use:** Scenes with extremely dense metaball clusters where $K$ is frequently exceeded and fallback causes visual artifacts or performance hits; scenes where precomputed vertex attributes (like Loop and Blinn's method) are feasible for fewer metaballs [S1].

## Background
The method is based on Nishita and Nakamae's algorithm [NN94], which used Bézier Clipping to solve quartic and degree-six polynomial equations for isosurface tests [S1]. It was adapted for the GPU by [S1] to handle large numbers of moving metaballs, addressing the limitations of polygonization (missing small objects, high memory cost) and previous GPU methods (Loop and Blinn's method, which is fast for few metaballs but slow for many due to neighbor-dependent vertex attributes; van Kooten et al.'s method, which suffers from small objects due to buffer resolution limits) [S1].

## Sources
[S1] GPU‐based Fast Ray Casting for a Large Number of Metaballs, https://doi.org/10.1111/j.1467-8659.2008.01132.x, fetched 2026-11-01
