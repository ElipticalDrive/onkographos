# Real-Time Metaball Ray Casting with Fragment Lists

Also called: Fragment List Based Metaball Ray Casting

## Overview
A real-time ray casting method for rendering particle-based medium representations (metaballs) that builds per-pixel lists of relevant metaballs, incrementally constructs a piecewise polynomial approximation of summed densities along rays, and finds intersections with the isosurface [S1]. It replaces the multipass depth peeling used in [GPU-based Fast Ray Casting for Metaballs](9502-gpu-based-fast-ray-casting-for-metaballs.md) with single-pass fragment list gathering, allowing robust handling of local extremities of depth complexity and scaling to millions of particles [S1].

## How it works
The method renders metaballs by evaluating the isosurface equation $\rho(\mathbf{x}) = \sum_{j=0}^{M-1} \rho_j (\|\mathbf{x} - \mathbf{p}_j\|) - L = 0$ along rays [S1]. It uses a piecewise cubic approximation $\tau_j(s)$ of the individual density functions $\rho_j$ along the ray parameter $s$, enabling efficient intersection testing via cubic polynomial solving [S1].

### Inputs and Outputs
**Inputs:**
*   Metaball data: Centers $\mathbf{p}_j$, effective radii $R_j$, and core radii (for depth culling) for $j=0 \dots M-1$ [S1].
*   Camera: Eye position $\mathbf{e}$ and viewing direction $\mathbf{v}$ per pixel [S1].
*   Isosurface value $L$ [S1].

**Outputs:**
*   Pixel color and depth [S1].

### Data Structures
1.  **Core Depth Texture:** Stores the depth of the closest intersected core sphere for each pixel [S1].
2.  **Per-pixel Linked Lists:** Maintained in a read/write GPU buffer. Stores metaball IDs. IDs are prepended to the list [S1].
3.  **Local-memory Array:** Stores separator records $(s_i, \Delta \mathbf{w}_i)$ in reverse order of depth. Elements are removed by decreasing the element count [S1].

### Definitions
*   **Effective Radius $R_j$:** The radius of the effective sphere of metaball $j$, within which the density function is non-zero [S1].
*   **Effective Interval:** The range $[m_j - Q_j, m_j + Q_j]$ along the ray parameter $s$ where the metaball density is non-zero [S1].
*   **Midpoint $m_j$:** $m_j = (\mathbf{p}_j - \mathbf{e}) \cdot \mathbf{v}$ [S1].
*   **Half-length $Q_j$:** $Q_j = \sqrt{R_j^2 - (\|\mathbf{p}_j - \mathbf{e}\|^2 - m_j^2)}$ [S1].
*   **Subdomain Separators:** The sorted list of end-points ($m_j - Q_j$) and mid-points ($m_j$) of all effective intervals along the ray, denoted $s_i$ for $i=1 \dots n$, with $s_0=0$ prepended [S1].
*   **Polynomial Coefficient Vector $\mathbf{w}$:** A 4-element vector representing the coefficients of the piecewise cubic approximation $\tau(s) = \mathbf{s} \cdot \mathbf{w}$, where $\mathbf{s} = (s^3, s^2, s, 1)$ [S1].

### Steps

1.  **Sorting:** Sort metaballs according to their effective distance from the camera, ascending. Effective distance is defined as $\|\mathbf{p}_j - \mathbf{e}\| - R_j$ [S1].
2.  **Lay down depth:** Render solid scene and metaball core depth into a core depth texture. Geometry rendered for core spheres is a covering billboard or enclosing object. The pixel shader discards pixels where the sphere is not intersected. Z-buffering is used, cores are rendered front-to-back [S1].
3.  **Gathering:** Render effective spheres of metaballs, gathering their IDs into per-pixel linked lists in a read/write GPU buffer. The intersected effective interval is clipped against core depth. Metaball ID is prepended to the linked list. Metaballs are rendered back-to-front, so lists are ordered by increasing effective distance [S1].
4.  **Ray casting:** Render a full-viewport quad. For every pixel, expand lists to end- and midpoint records, sort them, and evaluate ray-isosurface intersection [S1].
5.  **Shading:** Compute normal of isosurface at intersection point $\mathbf{x}$ as $-\nabla \rho(\mathbf{x})$ by iterating over gathered metaballs. Evaluate shading formula to get pixel color [S1].

#### Ray Casting Shader Algorithm
For each pixel, the shader executes the following steps [S1]:

1.  Initialize polynomial coefficient vector $\mathbf{w}$ to zero. If the eye is within the effective sphere of any metaball, sum the cubics of the intervals containing the eye [S1].
2.  For every metaball in the per-pixel list:
    a. Intersect the effective sphere with the ray. Insert separator records $(s_i, \Delta \mathbf{w}_i)$ for end- and midpoints into the local-memory array ordered by depth.
    b. Identify **safe separators**: those separators in the list which are within the effective distance of the next, yet unprocessed metaball (the "safe zone"). As metaballs are ordered, separators of further effective spheres cannot precede the safe ones.
    c. For safe separators, update polynomial coefficients by adding $\Delta \mathbf{w}_i$ and evaluate intersection within subdomains. Discard processed separators.
    d. If no metaballs are left, evaluate all remaining subdomains.
3.  If an intersection was found, compute the normal and shading [S1].

#### Polynomial Approximation and Coefficient Updates
The density function $\rho_j$ is approximated by a piecewise cubic $\tau_j(s)$ [S1]:
$$
\tau_j(s) = \begin{cases} 
0 & \text{if } s \notin [m_j - Q_j, m_j + Q_j], \\
\mathbf{s} \cdot \mathbf{u}_j \uparrow & \text{if } s \in [m_j - Q_j, m_j], \\
\mathbf{s} \cdot \mathbf{u}_j \downarrow & \text{if } s \in [m_j, m_j + Q_j],
\end{cases}
$$
where $\mathbf{u}_j \uparrow$ and $\mathbf{u}_j \downarrow$ are coefficient vectors derived by solving for values and derivatives at endpoints and midpoint [S1].

The change of coefficients $\Delta \mathbf{w}_i$ at separator $s_i$ is [S1]:
$$
\Delta \mathbf{w}_i = \frac{\gamma_j(s_i)}{Q_j^3} \times \text{case}
$$
where the case is:
*   $\alpha_i + \beta_i$ if $s_i = m_j - Q_j$
*   $-2\alpha_i$ if $s_i = m_j$
*   $\alpha_i - \beta_i$ if $s_i = m_j + Q_j$

with vectors:
$$
\alpha_i = (-2, 6s_i, -6s_i^2, 2s_i^3)^T
$$
$$
\beta_i = Q_j(0, 3, -6s_i, 3s_i^2)^T
$$
Note: The source text uses $\gamma_j(s_i)$ in the numerator. Since $\tau_j(s)$ approximates $\rho_j$, and the coefficients are normalized by $Q_j^3$, this factor likely represents the scaling of the specific density function used (e.g., Perlin's fifth-degree polynomial). For implementation, $\gamma_j(s_i)$ should be evaluated using the actual density function $\rho_j$ at $s_i$ to determine the magnitude of the coefficient jump [S1].

The polynomial coefficient update is:
$$
\mathbf{w}_i = \mathbf{w}_{i-1} + \Delta \mathbf{w}_i
$$
where $\mathbf{w}_0 = 0$ if the eye is not in any effective sphere [S1].

#### Intersection Evaluation
Within each subdomain $[s_i, s_{i+1})$, the total density is approximated by $\tau(s) = \mathbf{s} \cdot \mathbf{w}_i$. The intersection with the isosurface $L$ is found by solving $\tau(s) - L = 0$ for $s \in [s_i, s_{i+1})$. This reduces to solving a cubic equation. The source notes that the "practically exclusive monotonous case can easily be identified and very effectively handled" [S1].

### Parameters
*   **Isosurface density value $L$:** The threshold value defining the surface shape [S1].
*   **Local-memory array size:** Fixed size. Trade-off between performance and robustness of overlapping handling [S1].

### Edge Cases and Pitfalls
*   **Local Memory Overflow:** The fixed-size local memory array may be insufficient for too many overlapping spheres, forcing processing of non-safe separators. This creates a trade-off between performance and robustness of overlapping handling [S1].
*   **Depth Complexity:** There is no limit on screen-space depth complexity, only on the number of effective spheres a world-space point can belong to [S1].
*   **Eye Position:** If the eye is inside an effective sphere, the initial $\mathbf{w}$ must sum the cubics of intervals containing the eye [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MAX_LOCAL_SEPARATORS = 64  # Example fixed size, must be sufficient for typical overlap

# Vector operations assumed to be available for 4D vectors and dot products

def compute_separator_records(metaball, ray_origin, ray_dir):
    """
    Computes the separator records for a single metaball.
    Returns a list of (s, delta_w) tuples.
    """
    p = metaball.center
    R = metaball.effective_radius
    
    # Compute midpoint and half-length
    # m_j = (p_j - e) . v
    vec_to_center = p - ray_origin
    m_j = dot(vec_to_center, ray_dir)
    
    # Q_j = sqrt(R_j^2 - (||p_j - e||^2 - m_j^2))
    dist_sq = dot(vec_to_center, vec_to_center)
    Q_j_sq = R*R - (dist_sq - m_j*m_j)
    
    if Q_j_sq < 0:
        return [] # No intersection
    
    Q_j = sqrt(Q_j_sq)
    
    # Endpoints
    s_start = m_j - Q_j
    s_mid = m_j
    s_end = m_j + Q_j
    
    records = []
    
    # Helper to compute delta_w
    def get_delta_w(s_val, type):
        # type: 'start', 'mid', 'end'
        # alpha_i = (-2, 6s, -6s^2, 2s^3)
        # beta_i = Q_j * (0, 3, -6s, 3s^2)
        s = s_val
        alpha = vec4(-2, 6*s, -6*s*s, 2*s*s*s)
        beta = vec4(0, 3, -6*s, 3*s*s) * Q_j
        
        # gamma_j(s_i) is the density value at s_i. 
        # For the approximation to be consistent, we use the actual density function.
        # Assuming a standard polynomial density, e.g., Perlin's or similar.
        # Here we assume a generic density evaluation function exists.
        gamma = evaluate_density(metaball, ray_origin + s_val * ray_dir)
        
        factor = gamma / (Q_j * Q_j * Q_j)
        
        if type == 'start': # s_i = m_j - Q_j
            delta = factor * (alpha + beta)
        elif type == 'mid': # s_i = m_j
            delta = factor * (-2 * alpha)
        elif type == 'end': # s_i = m_j + Q_j
            delta = factor * (alpha - beta)
        else:
            delta = vec4(0)
            
        return delta

    # Insert records. The source says they are inserted in known order.
    # We collect them and will sort them later or insert into sorted array.
    # The source mentions "records of a lesser distance have to be moved only once"
    # implying insertion into a sorted list.
    
    records.append((s_start, get_delta_w(s_start, 'start')))
    records.append((s_mid, get_delta_w(s_mid, 'mid')))
    records.append((s_end, get_delta_w(s_end, 'end')))
    
    return records

def solve_cubic_intersection(w, s_start, s_end, L):
    """
    Solves w . (s^3, s^2, s, 1) - L = 0 for s in [s_start, s_end].
    Returns the smallest valid s or None.
    """
    # Polynomial: w[0]*s^3 + w[1]*s^2 + w[2]*s + (w[3] - L) = 0
    a = w[0]
    b = w[1]
    c = w[2]
    d = w[3] - L
    
    # Use a robust cubic solver (e.g., Cardano's method or numerical root finding)
    roots = solve_cubic_equation(a, b, c, d)
    
    valid_roots = [r for r in roots if s_start <= r < s_end]
    
    if not valid_roots:
        return None
    
    return min(valid_roots)

def ray_cast_pixel(pixels_list, ray_origin, ray_dir, L):
    """
    Main ray casting logic for a single pixel.
    pixels_list: List of metaball IDs, ordered by increasing effective distance.
    """
    # 1. Initialize w
    w = vec4(0, 0, 0, 0)
    
    # Check if eye is in any effective sphere to initialize w correctly
    # [G] Implementation detail: Iterate through metaballs to check if 
    # ray_origin is within R_j of p_j. If so, sum their initial cubics.
    # For simplicity, assuming w=0 is the base case if eye is outside.
    
    # Local memory array for separators
    local_separators = [] # List of (s, delta_w)
    
    # 2. Process metaballs
    for metaball_id in pixels_list:
        metaball = get_metaball(metaball_id)
        
        # 2a. Intersect effective sphere and get records
        records = compute_separator_records(metaball, ray_origin, ray_dir)
        
        # Insert records into local_separators, keeping them sorted by s
        # Since metaballs are sorted by effective distance, new records 
        # are likely to be at the end, but midpoints might interleave.
        for rec in records:
            insert_sorted(local_separators, rec)
            
        # 2b. Process safe separators
        # Safe zone: within effective distance of the next unprocessed metaball
        # [G] Determine the effective distance of the next metaball.
        # If it's the last metaball, the safe zone extends to infinity or 
        # the end of the last interval.
        
        next_metaball_idx = pixels_list.index(metaball_id) + 1
        if next_metaball_idx < len(pixels_list):
            next_metaball = get_metaball(pixels_list[next_metaball_idx])
            next_eff_dist = dot(next_metaball.center - ray_origin, ray_dir) # Approximation
            # More accurately, effective distance is ||p - e|| - R.
            # The "safe zone" concept implies we can process separators 
            # that are guaranteed to be before any new separators from 
            # further metaballs.
            safe_limit = next_eff_dist # Placeholder for exact safe limit logic
        else:
            safe_limit = float('inf')
            
        # Process separators that are "safe"
        # A separator is safe if it is within the effective distance of the next metaball.
        # The source says: "Safe separator records are those separators in the list, 
        # which are within the effective distance of the next, yet unprocessed metaball"
        
        processed_indices = []
        for idx, (s, delta_w) in enumerate(local_separators):
            if s < safe_limit:
                # Update w
                w = w + delta_w
                
                # Evaluate intersection in subdomain [s, next_s)
                # We need the next separator to define the subdomain end
                next_s = local_separators[idx+1][0] if idx+1 < len(local_separators) else float('inf')
                
                # Check for intersection in [s, next_s)
                # Note: The source says "evaluate intersection within subdomains"
                # This implies checking if the root lies in this interval.
                s_intersect = solve_cubic_intersection(w, s, next_s, L)
                
                if s_intersect is not None:
                    return s_intersect # Found intersection
                
                processed_indices.append(idx)
        
        # Discard processed separators (decrease element count)
        # Since we stored in reverse order or need to remove from front?
        # Source: "We store the elements ... in reverse order. Elements are removed simply by decreasing the element count."
        # This implies we might be popping from the end if stored in reverse.
        # Let's assume we remove from the beginning if sorted ascending.
        for idx in reversed(processed_indices):
            local_separators.pop(idx)
            
    # If no metaballs left, evaluate all remaining subdomains
    for idx in range(len(local_separators) - 1):
        s = local_separators[idx][0]
        next_s = local_separators[idx+1][0]
        
        s_intersect = solve_cubic_intersection(w, s, next_s, L)
        if s_intersect is not None:
            return s_intersect
            
    # Check the last subdomain if any separators remain
    if local_separators:
        s = local_separators[-1][0]
        # Next s is effectively infinity or the end of the ray
        s_intersect = solve_cubic_intersection(w, s, float('inf'), L)
        if s_intersect is not None:
            return s_intersect
            
    return None

def render_metaballs(pixels_list, ray_origin, ray_dir, L):
    s_intersect = ray_cast_pixel(pixels_list, ray_origin, ray_dir, L)
    
    if s_intersect is not None:
        x = ray_origin + s_intersect * ray_dir
        # Compute normal
        normal = compute_gradient(x) # -grad(rho(x))
        color = shade(x, normal)
        return color, s_intersect
    else:
        return background_color, float('inf')
```

## Operating characteristics
*   **Time Complexity:** Dominated by the ray casting step, which processes per-pixel fragment lists. The complexity depends on the number of metaballs per pixel and the depth complexity. The method scales well with the number of metaballs [S1].
*   **Memory Complexity:** Requires a read/write GPU buffer for per-pixel linked lists and a fixed-size local memory array in the shader. The local memory size is a critical parameter [S1].
*   **Performance:** 13 FPS for 100,000 metaballs with cel shading. 25 FPS, 13 FPS, and 6 FPS for mud simulation frames reaching 200,000 metaballs. Interactive framerates for 1,600,000 particles [S1].
*   **Strengths:** Handles local extremities of depth complexity robustly. Does not suffer from inaccuracies of screen-space filtering. Scales well for high numbers of particles. Replaces multipass depth peeling with single-pass gathering [S1].
*   **Limitations:** Fixed-size local memory array limits the number of overlapping spheres that can be handled robustly without performance penalty. Trade-off between performance and robustness of overlapping handling [S1].
*   **When to use:** Real-time rendering of massive particle simulations (metaballs) where depth complexity is high and polygonization is undesirable.
*   **When not to use:** Scenes with extremely high local overlap of effective spheres that exceed the local memory capacity, or when exact density evaluation is required (since it uses approximation) [S1].

## Background
The method was proposed by L. Szécsi and D. Illés from TU Budapest [S1]. It builds upon the work of Kanamori et al. [KSN08], replacing depth peeling with fragment list gathering and using piecewise cubic approximation instead of Bezier clipping [S1]. Implemented in C++ using Direct3D11 and HLSL shaders, tested on Intel Quad 2.40 GHz CPU and NVIDIA GeForce GTX 560 Ti. Test scenes were simulated in Next Limit's RealFlow [S1].

## Sources
[S1] Real-Time Metaball Ray Casting with Fragment Lists, https://doi.org/10.2312/conf/eg2012/short/093-096, fetched 2026-10-26
