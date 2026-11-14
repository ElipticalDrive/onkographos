# Real-Time Metaball Ray Casting with Fragment Lists

Also called: Fragment List Based Metaball Ray Casting, Metaballs as Volumes

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
2.  **Per-pixel Linked Lists:** Maintained in a read/write GPU buffer [S1]. Stores metaball IDs [S1]. IDs are prepended to the list [S1].
3.  **Local-memory Array:** Stores separator records $(s_i, \Delta \mathbf{w}_i)$ in reverse order of depth [S1]. Elements are removed by decreasing the element count [S1].

### Definitions
*   **Effective Radius $R_j$:** The radius of the effective sphere of metaball $j$, within which the density function is non-zero [S1].
*   **Effective Interval:** The range $[m_j - Q_j, m_j + Q_j]$ along the ray parameter $s$ where the metaball density is non-zero [S1].
*   **Midpoint $m_j$:** $m_j = (\mathbf{p}_j - \mathbf{e}) \cdot \mathbf{v}$ [S1].
*   **Half-length $Q_j$:** $Q_j = \sqrt{R_j^2 - (\|\mathbf{p}_j - \mathbf{e}\|^2 - m_j^2)}$ [S1].
*   **Subdomain Separators:** The sorted list of end-points ($m_j - Q_j$) and mid-points ($m_j$) of all effective intervals along the ray, denoted $s_i$ for $i=1 \dots n$, with $s_0=0$ prepended [S1].
*   **Polynomial Coefficient Vector $\mathbf{w}$:** A 4-element vector representing the coefficients of the piecewise cubic approximation $\tau(s) = \mathbf{s} \cdot \mathbf{w}$, where $\mathbf{s} = (s^3, s^2, s, 1)$ [S1].

### Steps

1.  **Sorting:** Sort metaballs according to their effective distance from the camera, ascending [S1]. Effective distance is defined as $\|\mathbf{p}_j - \mathbf{e}\| - R_j$ [S1].
2.  **Lay down depth:** Render solid scene and metaball core depth into a core depth texture [S1]. Geometry rendered for core spheres is a covering billboard or enclosing object [S1]. The pixel shader discards pixels where the sphere is not intersected [S1]. Z-buffering is used, cores are rendered front-to-back [S1].
3.  **Gathering:** Render effective spheres of metaballs, gathering their IDs into per-pixel linked lists in a read/write GPU buffer [S1]. The intersected effective interval is clipped against core depth [S1]. Metaball ID is prepended to the linked list [S1]. Metaballs are rendered back-to-front, so lists are ordered by increasing effective distance [S1].
4.  **Ray casting:** Render a full-viewport quad [S1]. For every pixel, expand lists to end- and midpoint records, sort them, and evaluate ray-isosurface intersection [S1].
5.  **Shading:** Compute normal of isosurface at intersection point $\mathbf{x}$ as $-\nabla \rho(\mathbf{x})$ by iterating over gathered metaballs [S1]. Evaluate shading formula to get pixel color [S1].

#### Ray Casting Shader Algorithm
For each pixel, the shader executes the following steps [S1]:

1.  Initialize polynomial coefficient vector $\mathbf{w}$ to zero [S1]. If the eye is within the effective sphere of any metaball, sum the cubics of the intervals containing the eye [S1].
2.  For every metaball in the per-pixel list:
    a. Intersect the effective sphere with the ray [S1]. Insert separator records $(s_i, \Delta \mathbf{w}_i)$ for end- and midpoints into the local-memory array ordered by depth [S1].
    b. Identify **safe separators**: those separators in the list which are within the effective distance of the next, yet unprocessed metaball (the "safe zone") [S1]. As metaballs are ordered, separators of further effective spheres cannot precede the safe ones [S1].
    c. For safe separators, update polynomial coefficients by adding $\Delta \mathbf{w}_i$ and evaluate intersection within subdomains [S1]. Discard processed separators [S1].
    d. If no metaballs are left, evaluate all remaining subdomains [S1].
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
*   $\alpha_i + \beta_i$ if $s_i = m_j - Q_j$ [S1]
*   $-2\alpha_i$ if $s_i = m_j$ [S1]
*   $\alpha_i - \beta_i$ if $s_i = m_j + Q_j$ [S1]

with vectors:
$$
\alpha_i = (-2, 6s_i, -6s_i^2, 2s_i^3)^T
$$
$$
\beta_i = Q_j(0, 3, -6s_i, 3s_i^2)^T
$$
Note: The source text uses $\gamma_j(s_i)$ in the numerator [S1]. Since $\tau_j(s)$ approximates $\rho_j$, and the coefficients are normalized by $Q_j^3$, this factor represents the scaling of the specific density function used (e.g., Perlin's fifth-degree polynomial) [S1]. For implementation, $\gamma_j(s_i)$ should be evaluated using the actual density function $\rho_j$ at $s_i$ to determine the magnitude of the coefficient jump [S1].

The polynomial coefficient update is:
$$
\mathbf{w}_i = \mathbf{w}_{i-1} + \Delta \mathbf{w}_i
$$
where $\mathbf{w}_0 = 0$ if the eye is not in any effective sphere [S1].

#### Intersection Evaluation
Within each subdomain $[s_i, s_{i+1})$, the total density is approximated by $\tau(s) = \mathbf{s} \cdot \mathbf{w}_i$ [S1]. The intersection with the isosurface $L$ is found by solving $\tau(s) - L = 0$ for $s \in [s_i, s_{i+1})$ [S1]. This reduces to solving a cubic equation [S1]. The source notes that the "practically exclusive monotonous case can easily be identified and very effectively handled" [S1].

### Parameters
*   **Isosurface density value $L$:** The threshold value defining the surface shape [S1].
*   **Local-memory array size:** Fixed size [S1]. Trade-off between performance and robustness of overlapping handling [S1].

### Edge Cases and Pitfalls
*   **Local Memory Overflow:** The fixed-size local memory array may be insufficient for too many overlapping spheres, forcing processing of non-safe separators [S1]. This creates a trade-off between performance and robustness of overlapping handling [S1].
*   **Depth Complexity:** There is no limit on screen-space depth complexity, only on the number of effective spheres a world-space point can belong to [S1].
*   **Eye Position:** If the eye is inside an effective sphere, the initial $\mathbf{w}$ must sum the cubics of intervals containing the eye [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cubic_roots(a, b, c, d):
    """Returns real roots of a*x^3 + b*x^2 + c*x + d = 0."""
    if abs(a) < 1e-12:
        # Quadratic or lower
        if abs(b) < 1e-12:
            if abs(c) < 1e-12:
                return []
            return [-d / c]
        disc = c*c - 4*b*d
        if disc < 0: return []
        sq = math.sqrt(disc)
        return [(-c - sq)/(2*b), (-c + sq)/(2*b)]
    
    # Normalize
    b /= a; c /= a; d /= a
    p = c - b*b/3
    q = 2*b*b*b/27 - b*c/3 + d
    disc = q*q/4 + p*p*p/27
    
    roots = []
    if disc > 0:
        u = -q/2 + math.sqrt(disc)
        v = -q/2 - math.sqrt(disc)
        u = math.cbrt(u) if u >= 0 else -math.cbrt(-u)
        v = math.cbrt(v) if v >= 0 else -math.cbrt(-v)
        roots = [u + v - b/3]
    elif disc == 0:
        if q == 0:
            roots = [-b/3]
        else:
            u = math.cbrt(-q/2)
            roots = [2*u - b/3, -u - b/3]
    else:
        r = math.sqrt(-p*p*p/27)
        theta = math.acos(-q/(2*r))
        m = 2 * math.cbrt(r)
        roots = [m*math.cos(theta/3) - b/3,
                 m*math.cos((theta + 2*math.pi)/3) - b/3,
                 m*math.cos((theta + 4*math.pi)/3) - b/3]
    return [r for r in roots if abs(r) > 1e-9]

def evaluate_density(metaball, point):
    """Evaluates the actual density function rho_j at point."""
    # Placeholder for the specific kernel (e.g., Perlin's 5th degree)
    dist = math.sqrt(sum((p - q)**2 for p, q in zip(metaball.center, point)))
    return metaball.density_func(dist)

def get_metaball(id):
    """Retrieves metaball data from buffer."""
    return metaball_database[id]

def compute_gradient(point, metaballs):
    """Computes -grad(rho(x)) by iterating over gathered metaballs."""
    grad = [0.0, 0.0, 0.0]
    for m in metaballs:
        # Simplified gradient approximation for demonstration
        dist = math.sqrt(sum((p - q)**2 for p, q in zip(m.center, point)))
        if dist > 0:
            factor = m.gradient_func(dist) / dist
            for k in range(3):
                grad[k] -= factor * (point[k] - m.center[k])
    return grad

def shade(point, normal):
    """Evaluates shading formula to get pixel color."""
    # Placeholder for lighting model
    return [0.8, 0.8, 0.8]

background_color = [0.0, 0.0, 0.0]

def insert_sorted(arr, item):
    """Inserts item into sorted list maintaining order."""
    arr.append(item)
    arr.sort(key=lambda x: x[0])

def ray_cast_pixel(pixels_list, ray_origin, ray_dir, L):
    """Main ray casting logic for a single pixel."""
    w = [0.0, 0.0, 0.0, 0.0]
    
    # [G] If eye is inside an effective sphere, sum initial cubics here.
    # For standard cases outside spheres, w remains zero.
    
    local_separators = []
    
    for idx, mid in enumerate(pixels_list):
        metaball = get_metaball(mid)
        p = metaball.center
        R = metaball.effective_radius
        
        vec_to_center = [p[k] - ray_origin[k] for k in range(3)]
        m_j = sum(vec_to_center[k] * ray_dir[k] for k in range(3))
        dist_sq = sum(v*v for v in vec_to_center)
        Q_j_sq = R*R - (dist_sq - m_j*m_j)
        
        if Q_j_sq < 0:
            continue
            
        Q_j = math.sqrt(Q_j_sq)
        s_start = m_j - Q_j
        s_mid = m_j
        s_end = m_j + Q_j
        
        def get_delta_w(s_val, type):
            alpha = [-2, 6*s_val, -6*s_val*s_val, 2*s_val*s_val*s_val]
            beta = [0, 3, -6*s_val, 3*s_val*s_val]
            beta = [b * Q_j for b in beta]
            
            point = [ray_origin[k] + s_val * ray_dir[k] for k in range(3)]
            gamma = evaluate_density(metaball, point)
            factor = gamma / (Q_j * Q_j * Q_j)
            
            if type == 'start':
                return [factor * (alpha[k] + beta[k]) for k in range(4)]
            elif type == 'mid':
                return [factor * (-2 * alpha[k]) for k in range(4)]
            elif type == 'end':
                return [factor * (alpha[k] - beta[k]) for k in range(4)]
            return [0.0]*4

        local_separators.append((s_start, get_delta_w(s_start, 'start')))
        local_separators.append((s_mid, get_delta_w(s_mid, 'mid')))
        local_separators.append((s_end, get_delta_w(s_end, 'end')))
        
        # Safe limit: effective distance of next metaball
        if idx + 1 < len(pixels_list):
            next_m = get_metaball(pixels_list[idx+1])
            next_p = next_m.center
            next_vec = [next_p[k] - ray_origin[k] for k in range(3)]
            next_eff_dist = math.sqrt(sum(v*v for v in next_vec)) - next_m.effective_radius
        else:
            next_eff_dist = float('inf')
            
        safe_indices = []
        for i, (s, delta) in enumerate(local_separators):
            if s < next_eff_dist:
                w = [w[k] + delta[k] for k in range(4)]
                safe_indices.append(i)
            else:
                break
                
        for i in safe_indices:
            s_curr = local_separators[i][0]
            s_next = local_separators[i+1][0] if i+1 < len(local_separators) else float('inf')
            
            a, b, c, d = w[0], w[1], w[2], w[3] - L
            roots = cubic_roots(a, b, c, d)
            valid = [r for r in roots if s_curr <= r < s_next]
            
            if valid:
                return min(valid)
                
        for i in reversed(safe_indices):
            local_separators.pop(i)
            
    for i in range(len(local_separators) - 1):
        s_curr = local_separators[i][0]
        s_next = local_separators[i+1][0]
        a, b, c, d = w[0], w[1], w[2], w[3] - L
        roots = cubic_roots(a, b, c, d)
        valid = [r for r in roots if s_curr <= r < s_next]
        if valid:
            return min(valid)
            
    if local_separators:
        s_curr = local_separators[-1][0]
        a, b, c, d = w[0], w[1], w[2], w[3] - L
        roots = cubic_roots(a, b, c, d)
        valid = [r for r in roots if r >= s_curr]
        if valid:
            return min(valid)
            
    return None

def render_metaballs(pixels_list, ray_origin, ray_dir, L):
    s_intersect = ray_cast_pixel(pixels_list, ray_origin, ray_dir, L)
    if s_intersect is not None:
        x = [ray_origin[k] + s_intersect * ray_dir[k] for k in range(3)]
        normal = compute_gradient(x, pixels_list)
        color = shade(x, normal)
        return color, s_intersect
    return background_color, float('inf')
```

## Operating characteristics
*   **Time Complexity:** Dominated by the ray casting step, which processes per-pixel fragment lists [S1]. The complexity depends on the number of metaballs per pixel and the depth complexity [S1]. The method scales well with the number of metaballs [S1].
*   **Memory Complexity:** Requires a read/write GPU buffer for per-pixel linked lists and a fixed-size local memory array in the shader [S1]. The local memory size is a critical parameter [S1].
*   **Performance:** 13 FPS for 100,000 metaballs with cel shading [S1]. 25 FPS, 13 FPS, and 6 FPS for mud simulation frames reaching 200,000 metaballs [S1]. Interactive framerates for 1,600,000 particles [S1].
*   **Strengths:** Handles local extremities of depth complexity robustly [S1]. Does not suffer from inaccuracies of screen-space filtering [S1]. Scales well for high numbers of particles [S1]. Replaces multipass depth peeling with single-pass gathering [S1].
*   **Limitations:** Fixed-size local memory array limits the number of overlapping spheres that can be handled robustly without performance penalty [S1]. Trade-off between performance and robustness of overlapping handling [S1].
*   **When to use:** Real-time rendering of massive particle simulations (metaballs) where depth complexity is high and polygonization is undesirable [S1].
*   **When not to use:** Scenes with extremely high local overlap of effective spheres that exceed the local memory capacity [S1].

## Background
The method was proposed by L. Szécsi and D. Illés from TU Budapest [S1]. It builds upon the work of Kanamori et al. [KSN08], replacing depth peeling with fragment list gathering and using piecewise cubic approximation instead of Bezier clipping [S1]. Implemented in C++ using Direct3D11 and HLSL shaders, tested on Intel Quad 2.40 GHz CPU and NVIDIA GeForce GTX 560 Ti [S1]. Test scenes were simulated in Next Limit's RealFlow [S1]. Rendering metaballs as volumes can be enabled in Houdini's mantra renderer via the 'Metaballs as Volume' checkbox on the Geometry sub-tab of the Render tab of a Geometry object [S2]. Point attributes on the metaballs are interpolated to the shading position in the same manner as for metaball surfaces [S2]. A surface material that takes into account volume primitives, such as basic smoke, must be applied [S2].

## Sources
[S1] Real-Time Metaball Ray Casting with Fragment Lists, https://doi.org/10.2312/conf/eg2012/short/093-096, fetched 2026-10-26
[S2] Rendering volumes - SideFX, https://www.sidefx.com/docs/houdini/render/volumes.html, fetched 2026-10-29
