# Ray-splat Intersection with Numerical Stabilization

Also called: Linear Fallback for Ray-Gaussian Intersection, Stable Ray-Gaussian Intersection

## Overview
This method computes the intersection depths between camera rays and quadric splats, providing a numerically stable solution when the quadratic coefficient is near zero. It differs from standard quadratic solvers by detecting degenerate cases where the quadric surface is nearly flat or the ray is nearly perpendicular to the principal axes, switching to a linear approximation to prevent floating-point overflow [S1]. It builds on [Ray-Splat/gaussian Intersection](7916-ray-splat-gaussian-intersection.md) by handling the specific numerical instability inherent in second-order primitives.

## How it works
**Inputs and Outputs**
- **Inputs**: Ray origin $\hat{\mathbf{o}} = [\hat{o}^1, \hat{o}^2, \hat{o}^3]^T$, ray direction $\hat{\mathbf{r}} = [\hat{r}^1, \hat{r}^2, \hat{r}^3]^T$, and quadric parameters $s_1, s_2, s_3$ defining the surface geometry in the Gaussian local coordinate system.
- **Outputs**: Intersection depth $t$.

**Data Structures**
The method operates on scalar values derived from the ray and quadric parameters. No complex data structures are required beyond standard floating-point variables.

**Steps**
1.  **Compute Quadratic Coefficients**: Calculate the coefficients $A$, $B$, and $C$ for the quadratic equation $At^2 + Bt + C = 0$ using the ray and quadric parameters:
    $$
    A = \frac{\text{sign}(s_1)}{s_1^2} (\hat{r}^1)^2 + \frac{\text{sign}(s_2)}{s_2^2} (\hat{r}^2)^2
    $$
    $$
    B = \frac{\text{sign}(s_1)}{s_1^2} 2\hat{o}^1\hat{r}^1 + \frac{\text{sign}(s_2)}{s_2^2} 2\hat{o}^2\hat{r}^2 - \frac{1}{s_3}\hat{r}^3
    $$
    $$
    C = \frac{\text{sign}(s_1)}{s_1^2} (\hat{o}^1)^2 + \frac{\text{sign}(s_2)}{s_2^2} (\hat{o}^2)^2 - \frac{1}{s_3}\hat{o}^3
    $$
2.  **Check for Numerical Stability**: Evaluate the magnitude of $A$. If $|A| < \epsilon$ (where $\epsilon = 10^{-6}$), the quadratic term is negligible.
3.  **Solve Linear Equation (Fallback)**: If $|A| < \epsilon$, solve the linear equation $Bt + C = 0$:
    $$
    t = -\frac{C}{B}
    $$
    This solution corresponds to the intersection with the tangent plane at the point where the perpendicular from the camera center meets the quadric surface [S1].
4.  **Solve Quadratic Equation (Standard)**: If $|A| \ge \epsilon$, solve the quadratic equation using the stable formula:
    $$
    t_{n,f} = \frac{-B \pm \text{sign}(A) \cdot \sqrt{B^2 - 4AC}}{2A}
    $$
    Select the appropriate root based on the viewing direction (typically the positive root for forward rendering) [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_splat_intersection_stable(o, r, s1, s2, s3):
    """
    Compute intersection depth with numerical stabilization.
    
    Args:
        o: Ray origin [o1, o2, o3]
        r: Ray direction [r1, r2, r3]
        s1, s2, s3: Quadric parameters
        
    Returns:
        t: Intersection depth
    """
    # Compute coefficients A, B, C
    sign_s1 = 1.0 if s1 > 0 else -1.0
    sign_s2 = 1.0 if s2 > 0 else -1.0
    
    A = (sign_s1 / (s1**2)) * (r[0]**2) + (sign_s2 / (s2**2)) * (r[1]**2)
    B = (sign_s1 / (s1**2)) * 2 * o[0] * r[0] + (sign_s2 / (s2**2)) * 2 * o[1] * r[1] - (1.0 / s3) * r[2]
    C = (sign_s1 / (s1**2)) * (o[0]**2) + (sign_s2 / (s2**2)) * (o[1]**2) - (1.0 / s3) * o[2]
    
    # Numerical stabilization threshold
    epsilon = 1e-6
    
    if abs(A) < epsilon:
        # Linear fallback
        if abs(B) < epsilon:
            # Degenerate case: ray parallel to surface or invalid geometry
            # Return a large value or handle as no intersection
            return float('inf')
        t = -C / B
    else:
        # Quadratic solution
        discriminant = B**2 - 4 * A * C
        if discriminant < 0:
            # No real intersection
            return float('inf')
        
        sqrt_disc = math.sqrt(discriminant)
        sign_A = 1.0 if A > 0 else -1.0
        
        # Two roots
        t1 = (-B + sign_A * sqrt_disc) / (2 * A)
        t2 = (-B - sign_A * sqrt_disc) / (2 * A)
        
        # Select the front-most intersection (positive t)
        # In forward rendering, we typically want the smallest positive t
        if t1 > 0 and t2 > 0:
            t = min(t1, t2)
        elif t1 > 0:
            t = t1
        elif t2 > 0:
            t = t2
        else:
            # No positive intersection
            return float('inf')
            
    return t
```

**Parameters**
- $\epsilon = 10^{-6}$: Threshold for determining when $A$ is negligible [S1].

**Edge Cases and Pitfalls**
- **Numerical Overflow**: When $|A|$ is very small, the standard quadratic formula can cause floating-point overflow, resulting in NaN or Inf values, particularly during backpropagation [S1]. The linear fallback prevents this.
- **Degenerate Geometry**: If both $A$ and $B$ are near zero, the ray is parallel to the surface or the geometry is invalid. The method should handle this by returning a large depth or marking as no intersection [G].
- **No Real Intersection**: If the discriminant $B^2 - 4AC < 0$, there are no real intersection points. The method should return a value indicating no intersection [G].

## Operating characteristics
- **Time Complexity**: $O(1)$ per ray-splat intersection. The linear fallback is slightly faster than the quadratic solution due to the absence of square root and division by $2A$.
- **Memory Complexity**: $O(1)$ additional memory.
- **Strengths**: Prevents numerical instability during training and rendering, especially for flat splats or rays nearly perpendicular to the principal axes. Ensures stable gradients during backpropagation.
- **Limitations**: The linear approximation is less accurate than the quadratic solution when $A$ is small but not negligible. The choice of $\epsilon$ affects the transition between linear and quadratic solutions.
- **When to Use**: Always use this method for ray-splat intersection in Gaussian Splatting variants that use quadric primitives, especially when training or when high precision is not critical for the fallback case.
- **When Not to Use**: Not applicable; this is a robust general-purpose method for the specified intersection problem.

## Background
This method was introduced in "Quadratic Gaussian Splatting: High Quality Surface Reconstruction with Second-Order Geometric Primitives" [S1]. It addresses the numerical instability inherent in solving quadratic equations for ray-splat intersection, which can lead to NaN or Inf values during backpropagation. The linear fallback is geometrically interpreted as the intersection with the tangent plane at the point where the perpendicular from the camera center meets the quadric surface.

## Sources
[S1] Quadratic Gaussian Splatting: High Quality Surface Reconstruction with Second-Order Geometric Primitives, https://doi.org/10.1109/ICCV51701.2025.02624, fetched 2026-10-30
