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
- For each seed $x$, store the maximum separation factor $\hat{s}(x)$ computed over all isolevels [S1, book].

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
import math

def norm(v):
    return math.sqrt(sum(c**2 for c in v))

def dot(a, b):
    return sum(ai * bi for ai, bi in zip(a, b))

def compute_ftle_quad_size(x, neighbors, scalar_field, l_min, l_max, a_thresh):
    """
    Computes the maximum separation factor s_hat(x) for seed x.
    """
    max_separation = 0.0

    for n in neighbors:
        s_n = [ni - xi for xi, ni in zip(x, n)]
        len_s_n = norm(s_n)

        # Exclusion criterion: check angle between segment and gradient
        excluded = False
        # Source defines a_min as max over t in [0,1].
        # Implementation samples t discretely; exact max requires root finding [G].
        for t in [0.0, 0.25, 0.5, 0.75, 1.0]:
            point = [xi + t * si for xi, si in zip(x, s_n)]
            grad = evaluate_gradient(scalar_field, point) # [G]
            grad_len = norm(grad)
            if grad_len > 0:
                cos_angle = abs(dot(s_n, grad)) / (len_s_n * grad_len)
                if cos_angle > a_thresh:
                    excluded = True
                    break

        if excluded:
            continue

        # Compute separation factor s_l(x) for relevant isolevels
        current_max_s_l = 0.0
        # Source implies precomputing max over l. Discrete sampling used here [G].
        for l in [l_min, (l_min + l_max) / 2, l_max]:
            phi_l_x = trace_field_line(scalar_field, x, l) # [G]
            phi_l_n = trace_field_line(scalar_field, n, l) # [G]
            if phi_l_x is not None and phi_l_n is not None:
                dist_advected = norm([phi_l_n[i] - phi_l_x[i] for i in range(len(x))])
                s_l = dist_advected / len_s_n
                if s_l > current_max_s_l:
                    current_max_s_l = s_l

        if current_max_s_l > max_separation:
            max_separation = current_max_s_l

    return max_separation
```

## Operating characteristics
- **Complexity:** Requires tracing gradient field lines for every seed and its neighbors to determine isosurface intersections [S1, book]. The exclusion criterion adds cost proportional to the number of samples along the segment between seeds [S1, book].
- **Strengths:** Guarantees complete covering of isosurfaces, preventing holes in the rendering [S1, book]. Adapts to local geometry via the separation factor [S1, book].
- **Limitations:** Computationally expensive due to field line tracing and separation calculations [S1, book]. Overdraw may occur if the exclusion threshold is not tuned properly, leading to performance loss [S1, book].
- **When to use:** When artifact-free isosurface rendering is critical and computational resources allow for precomputation or expensive per-seed calculations [S1, book].

## Background
The method is derived from dynamical systems theory, specifically the Finite-Time Lyapunov Exponent (FTLE), originally defined to measure the predictability of dynamical systems by quantifying the exponential growth of perturbations [S1, book]. It was adapted for isosurface rendering to estimate conservative quad sizes that guarantee complete surface coverage [S1, book]. The approach was introduced in the context of interactive isocontouring of high-order surfaces [S1, book].

## Sources
[S1] Interactive Isocontouring of High-Order Surfaces, https://doi.org/10.4230/dfu.vol2.sciviz.2011.276, fetched 2026-10-29 (book)
