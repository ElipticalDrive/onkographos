# Gaussian Depth Calculation via Ellipsoid Intersection

Also called: Geometric depth, Mid-point depth

## Overview
This method computes the depth of a 3D Gaussian along a ray by intersecting the ray with the ellipsoid defined by one standard deviation of the Gaussian distribution and selecting the midpoint of the resulting intersection interval [S1]. It serves as a numerically stable alternative to the cubic intersection solver used in [Quadratic Ray-Gaussian Intersection](7017-quadratic-ray-gaussian-intersection.md), avoiding instability while providing a discriminant to verify intersection existence, which the maximum-density derivative method lacks [S1].

## How it works
**Inputs:**
- Ray origin $\mathbf{o} = (o_x, o_y, o_z)$ and direction $\mathbf{d} = (d_x, d_y, d_z)$.
- Gaussian scale parameters $\mathbf{s} = (s_x, s_y, s_z)$.
- Note: $\mathbf{o}$ and $\mathbf{d}$ must be transformed into the local coordinate system of the Gaussian ellipsoid [S1].

**Outputs:**
- Depth parameter $t^{mid}$ representing the distance along the ray to the Gaussian's center.
- Boolean flag indicating intersection existence.

**Data Structures:**
- No complex structures required. Standard floating-point variables for coefficients $a, b, c$ and discriminant $\Delta$.

**Steps:**
1.  **Formulate the Ellipsoid Equation:**
    The Gaussian is modeled as an ellipsoid where the density drops to a specific threshold at distance $s_i$ from the center along axis $i$. The equation is:
    $$ \frac{(o_x + td_x)^2}{s_x^2} + \frac{(o_y + td_y)^2}{s_y^2} + \frac{(o_z + td_z)^2}{s_z^2} = 1 \quad (4) $$

2.  **Expand to Quadratic Form:**
    Substitute the ray equation $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ into the ellipsoid equation and expand to obtain the quadratic form $at^2 + bt + c = 0$:
    $$ a = \frac{d_x^2}{s_x^2} + \frac{d_y^2}{s_y^2} + \frac{d_z^2}{s_z^2} $$
    $$ b = 2 \left( \frac{o_x d_x}{s_x^2} + \frac{o_y d_y}{s_y^2} + \frac{o_z d_z}{s_z^2} \right) $$
    $$ c = \frac{o_x^2}{s_x^2} + \frac{o_y^2}{s_y^2} + \frac{o_z^2}{s_z^2} - 1 \quad (5) $$

3.  **Compute Discriminant:**
    Calculate the discriminant to determine if the ray intersects the ellipsoid:
    $$ \Delta = b^2 - 4ac $$
    If $\Delta < 0$, no real intersection exists [S1].

4.  **Calculate Mid-point Parameter:**
    The depth $t^{mid}$ corresponds to the vertex of the parabola defined by the quadratic equation, which is the midpoint of the intersection interval:
    $$ t^{mid} = -\frac{b}{2a} \quad (7) $$
    This value is equivalent to the $t$ that maximizes the Gaussian density along the ray ($t_{opt}$), but the ellipsoid formulation provides the explicit discriminant check [S1].

5.  **Compute Position (Optional):**
    The 3D position of the depth point is:
    $$ \mathbf{p}^{mid} = \mathbf{o} + t^{mid}\mathbf{d} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_depth_ellipsoid(o, d, s):
    """
    Compute the depth of a 3D Gaussian via ellipsoid intersection.
    
    Args:
        o: Ray origin (vec3)
        d: Ray direction (vec3)
        s: Gaussian scale (vec3)
        
    Returns:
        t_mid: Depth parameter, or None if no intersection
    """
    # Precompute inverse squared scales
    inv_sx2 = 1.0 / (s.x * s.x)
    inv_sy2 = 1.0 / (s.y * s.y)
    inv_sz2 = 1.0 / (s.z * s.z)
    
    # Calculate quadratic coefficients
    a = (d.x * d.x * inv_sx2) + (d.y * d.y * inv_sy2) + (d.z * d.z * inv_sz2)
    
    # Guard against degenerate zero scale or direction
    if a == 0:
        return None
        
    b = 2.0 * ((o.x * d.x * inv_sx2) + (o.y * d.y * inv_sy2) + (o.z * d.z * inv_sz2))
    
    c = (o.x * o.x * inv_sx2) + (o.y * o.y * inv_sy2) + (o.z * o.z * inv_sz2) - 1.0
    
    # Calculate discriminant
    delta = (b * b) - (4.0 * a * c)
    
    # Check for intersection
    if delta < 0:
        return None
        
    # Calculate midpoint parameter t_mid = -b / 2a
    t_mid = -b / (2.0 * a)
    
    return t_mid
```

**Edge Cases and Pitfalls:**
-   **Degenerate Scales:** If any $s_i$ is zero, division by zero occurs. In practice, Gaussian scales are regularized to be non-zero [G].
-   **No Intersection:** The discriminant $\Delta < 0$ correctly identifies rays that miss the Gaussian's effective volume.
-   **Blending Artifacts:** While this method provides a geometric depth, using it in conjunction with alpha blending can still suffer from artifacts such as depth collapse (unreasonably low depth values due to default black background colors), object misalignment (inaccurate depth from large Gaussians/floaters), and blending boundary issues (smooth edges in depth maps due to low blending weights) [S1].

## Operating characteristics
-   **Time Complexity:** $O(1)$ per Gaussian. Requires a few multiplications, additions, and one square root (if full intersection points were needed, but for mid-point only, no square root is needed as $\sqrt{\Delta}$ is not used for $t^{mid}$).
-   **Memory Complexity:** $O(1)$.
-   **Strengths:** Numerically stable compared to cubic solvers; provides an explicit check for intersection existence via $\Delta$; computationally cheaper than solving for two roots and averaging.
-   **Limitations:** It approximates the Gaussian as a hard ellipsoid boundary rather than a soft density field. This may differ slightly from the true density maximum in highly anisotropic Gaussians, though the source notes they yield the same numerical result for $t$ [S1].
-   **When to use:** When a stable, fast depth estimate for 3DGS rendering or reprojection is needed, and the numerical instability of cubic solvers is a concern.

## Background
This method is described in the context of "Mini-Splatting," which aims to represent scenes with a constrained number of Gaussians [S1]. It is presented as a practical alternative to calculating the maximum density point via derivative optimization, leveraging the geometric properties of the ellipsoid defined by the Gaussian's scale parameters.

## Sources
[S1] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
