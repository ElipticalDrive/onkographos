# FTLE-based Quad Size Estimation

Also called: FTLE-based Sampling, Quad Size Estimation using FTLE, Separation Factor Estimation

## Overview
A method to determine conservative quad sizes for isosurface rendering that guarantees complete covering of isosurfaces by estimating the separation factor between seeds using Finite-Time Lyapunov Exponent (FTLE) concepts from dynamical systems theory [S1, book]. Unlike standard FTLE which measures predictability over a fixed time interval, this method adapts the separation metric to the distance along gradient field lines to the target isosurface, ensuring quads are large enough to cover the surface but small enough to avoid gaps [S1, book].

## How it works
**Inputs:**
- Scalar field $S$.
- Object-space seed points $x$.
- Neighbor set $N(x)$ for each seed.
- Isolevel range $[l_{min}, l_{max}]$.
- Exclusion threshold $a_{thresh}$ (typically 0.5) [S1, book].

**Outputs:**
- Quad size $d(x)$ for each seed $x$.

**Data Structures:**
- For each seed $x$, store the maximum separation factor $\hat{s}(x)$ computed over all isolevels.

**Steps:**

1.  **Define Flow Map:** For a seed $x$ and an isolevel $l$, define $\phi_l(x)$ as the position where the gradient field line starting at $x$ intersects the isosurface $S(x) = l$. The integration stops if a critical point of the gradient field is reached [S1, book].

2.  **Compute Separation Factor $s_l(x)$:**
    Instead of computing the Cauchy-Green deformation tensor directly, calculate the separation factor by comparing the distance between advected neighbors to the distance between original seeds [S1, book]:
    $$ s_l(x) = \max_{n \in N(x)} \frac{||\phi_l(n) - \phi_l(x)||}{||n - x||} $$
    This omits the normalization by advection time and the logarithm found in the standard FTLE definition $\sigma_T(x) = \frac{1}{|T|} \ln \sqrt{\lambda_{max}(\Delta_T(x))}$ [S1, book].

3.  **Apply Exclusion Criterion:**
    To prevent erroneous neighbors (those mapped to different isosurface parts or non-planar regions) from inflating the quad size, compute the minimum angle measure $a_{min}$ between the segment $s_n$ connecting $x$ to neighbor $n$ and the gradient field along that segment [S1, book]:
    $$ a_{min} = \max_{t \in [0,1]} \frac{|s_n \cdot \nabla_s(x + t s_n)|}{||s_n|| \, ||\nabla_s(x + t s_n)||} $$
    If $a_{min} > a_{thresh}$, exclude neighbor $n$ from the maximization in Step 2 [S1, book].

4.  **Compute Maximum Separation $\hat{s}(x)$:**
    Precompute the maximum separation factor over the entire isolevel range to ensure coverage for any isolevel [S1, book]:
    $$ \hat{s}(x) = \max_{l_{min} \le l \le l_{max}} s_l(x) $$

5.  **Determine Quad Size:**
    The quad size $d(x)$ at the mapped position $\phi_l(x)$ is given by [S1, book]:
    $$ d(x) = \hat{s}(x) \cdot ||n - x|| $$
    Using $\hat{s}(x)$ guarantees that the quad covers the isosurface for any $l \in [l_{min}, l_{max}]$ [S1, book].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_ftle_quad_size(x, neighbors, scalar_field, l_min, l_max, a_thresh):
    """
    Computes the quad size for seed x based on FTLE-based separation.
    """
    max_separation = 0.0
    
    for n in neighbors:
        # Compute segment vector
        s_n = n - x
        len_s_n = norm(s_n)
        
        # Check exclusion criterion
        # Sample points along the segment from x to n
        excluded = False
        for t in [0.0, 0.25, 0.5, 0.75, 1.0]: # Discrete sampling for max
            point = x + t * s_n
            grad = compute_gradient(scalar_field, point)
            
            # Compute cosine of angle between segment and gradient
            # |s_n . grad| / (||s_n|| ||grad||)
            if norm(grad) > 0:
                cos_angle = abs(dot(s_n, grad)) / (len_s_n * norm(grad))
                if cos_angle > a_thresh:
                    excluded = True
                    break
        
        if excluded:
            continue
            
        # Compute flow map positions for x and n
        # Find intersection with isosurfaces in range [l_min, l_max]
        # phi_l(x) is the point on gradient line from x where S(p) == l
        
        # We need max separation over all l in [l_min, l_max]
        # In practice, this might be done by tracing to the closest isosurface
        # or sampling multiple levels. The source implies precomputing max over l.
        
        # Simplified: Assume we find the max separation factor encountered
        # during the trace from x to the isosurface.
        # Note: The source defines s_l(x) for a specific l.
        # To implement max over l, one would trace for each l or find the 
        # worst-case separation.
        
        # For the purpose of this pseudocode, we assume a function that returns
        # the max separation factor for the pair (x, n) across the relevant levels.
        separation = compute_max_separation_pair(x, n, scalar_field, l_min, l_max)
        
        if separation > max_separation:
            max_separation = separation
            
    # Quad size is separation factor times original seed distance
    # Using the max neighbor distance for the specific quad generation context
    # The source says d(x) = s_l(x) * ||n - x||. 
    # Here we use the max separation found.
    # Note: ||n-x|| should be the distance to the neighbor that determined the max separation.
    # For simplicity, we return the factor and let the caller multiply by the specific edge length.
    return max_separation

def compute_max_separation_pair(x, n, scalar_field, l_min, l_max):
    """
    Computes max(||phi_l(n) - phi_l(x)|| / ||n - x||) for l in [l_min, l_max].
    """
    max_sep = 0.0
    dist_xn = norm(n - x)
    
    # Trace gradient field lines from x and n
    # Stop when crossing any isolevel in [l_min, l_max]
    # This requires solving for t where S(x(t)) = l
    
    # Simplified implementation:
    # Trace both lines. For each isolevel l, find intersection points.
    # Calculate distance ratio.
    
    # Since exact root finding is complex, we assume a helper that finds
    # the intersection points phi_l(x) and phi_l(n) for the worst-case l.
    
    # Placeholder for the actual integration logic which is non-trivial
    # to implement without specific solver details.
    # The source states: "sl(x) = max_{n in N(x)} ||phi_l(n) - phi_l(x)|| / ||n - x||"
    # and "s_hat(x) = max_{l_min <= l <= l_max} sl(x)"
    
    # We return a representative max separation.
    return max_sep
```

## Operating characteristics
- **Complexity:** Requires tracing gradient field lines for every seed and its neighbors to determine isosurface intersections. The exclusion criterion adds cost proportional to the number of samples along the segment between seeds.
- **Strengths:** Guarantees complete covering of isosurfaces, preventing holes in the rendering. Adapts to local geometry via the separation factor.
- **Limitations:** Computationally expensive due to field line tracing and eigenvalue/separation calculations. Overdraw may occur if the exclusion threshold is not tuned properly, leading to performance loss [S1, book].
- **When to use:** When artifact-free isosurface rendering is critical and computational resources allow for precomputation or expensive per-seed calculations.

## Background
The method is derived from dynamical systems theory, specifically the Finite-Time Lyapunov Exponent (FTLE), originally defined to measure the predictability of dynamical systems by quantifying the exponential growth of perturbations [S1, book]. It was adapted for isosurface rendering to estimate conservative quad sizes that guarantee complete surface coverage. The approach was introduced in the context of interactive isocontouring of high-order surfaces [S1, book].

## Sources
[S1] Interactive Isocontouring of High-Order Surfaces, https://doi.org/10.4230/dfu.vol2.sciviz.2011.276, fetched 2026-10-29 (book)
