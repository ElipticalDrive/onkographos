# Ray-Ellipsoid Intersection
Also called: Ellipsoid Ray Intersection
## Overview
Computes the intersection of a ray with an ellipsoid defined by a rotation matrix, translation vector, and scale factors, returning the existence of an intersection and the intersection point in world space [S1, unofficial]. It differs from general implicit surface methods by exploiting the quadratic nature of the ellipsoid equation to solve for intersections analytically via a transformed coordinate system, offering a closed-form solution that is computationally cheaper than iterative root-finding [S1, unofficial].
## How it works
### Inputs and Outputs
**Inputs:**
- Ray origin $\mathbf{o}$ (3D vector)
- Ray direction $\mathbf{d}$ (3D vector)
- Ellipsoid center $\mathbf{t}$ (3D vector)
- Ellipsoid rotation matrix $\mathbf{R}$ (3x3 matrix)
- Ellipsoid scale factors $S = [S_x, S_y, S_z]$ (3D vector)
- Epsilon $\epsilon$ for numerical stability (scalar, default $10^{-5}$)

**Outputs:**
- Boolean indicating intersection existence
- Intersection point $\mathbf{p}$ in world space (3D vector) if intersection exists

### Data Structures
No complex data structures are required. The method operates on standard 3D vectors and 3x3 matrices.

### Steps
1.  **Compute Inverse Transformation:** Calculate the matrix that transforms world coordinates to the ellipsoid's local space. This matrix is the product of the inverse scale diagonal matrix and the transpose of the rotation matrix:
    $$ \mathbf{M}_{inv} = \text{diag}\left(\frac{1}{S_x}, \frac{1}{S_y}, \frac{1}{S_z}\right) \mathbf{R}^T $$
2.  **Transform Ray:** Transform the ray origin and direction into the ellipsoid's local space:
    $$ \mathbf{o}' = \mathbf{M}_{inv} (\mathbf{o} - \mathbf{t}) $$
    $$ \mathbf{d}' = \mathbf{M}_{inv} \mathbf{d} $$
3.  **Handle Degenerate Direction:** To prevent division by zero in the quadratic solver, check the first component of the transformed direction $\mathbf{d}'$. Let $A$ be this component (or the magnitude of the direction vector squared in the standard quadratic form derivation, but here specifically referenced as a component for stability in the source). If $|A| < \epsilon$, set $A = \epsilon$ [S1, unofficial].
4.  **Compute Discriminant:** Calculate the discriminant of the resulting quadratic equation. Let $B$ and $C$ be the other coefficients derived from the transformed ray and ellipsoid equation (specifically, $B = 2(\mathbf{o}' \cdot \mathbf{d}')$ and $C = \mathbf{o}' \cdot \mathbf{o}' - 1$ in the standard unit sphere intersection context, though the source implies a direct calculation). If $\text{discriminant} < 0$, there is no intersection [S1, unofficial].
5.  **Handle Near-Zero Discriminant:** If $|\text{discriminant}| < \epsilon$, set $\text{discriminant} = 0$ to avoid numerical instability in the square root [S1, unofficial].
6.  **Compute Roots:** Calculate the square root of the discriminant and the two potential intersection parameters $t_1$ and $t_2$:
    $$ \sqrt{\text{discriminant}} = \sqrt{\text{discriminant}} $$
    $$ t_1 = \frac{-B - \sqrt{\text{discriminant}}}{2A} $$
    $$ t_2 = \frac{-B + \sqrt{\text{discriminant}}}{2A} $$
7.  **Check Ray Direction:** If both $t_1 < 0$ and $t_2 < 0$, the intersections are behind the ray origin, so there is no valid intersection [S1, unofficial].
8.  **Select Valid $t$:** Choose the smallest non-negative $t$. If $t_1 \ge 0$, select $t_{intersect} = t_1$. Otherwise, select $t_{intersect} = t_2$. If $|t_1 - t_2| < \epsilon$, set $t_{intersect} = \min(t_1, t_2)$ [S1, unofficial].
9.  **Compute Intersection Point in Local Space:**
    $$ \mathbf{p}' = \mathbf{o}' + t_{intersect} \mathbf{d}' $$
10. **Transform Back to World Space:** Transform the local intersection point back to world space using the forward transformation matrix $\mathbf{M}_{fwd} = \mathbf{R} \text{diag}(S_x, S_y, S_z)$:
    $$ \mathbf{p} = \mathbf{M}_{fwd} \mathbf{p}' + \mathbf{t} $$

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def ray_ellipsoid_intersection(o, d, t, R, S, eps=1e-5):
    """
    Compute intersection of ray with ellipsoid.
    
    Args:
        o: Ray origin (3,)
        d: Ray direction (3,)
        t: Ellipsoid center (3,)
        R: Rotation matrix (3,3)
        S: Scale factors (3,)
        eps: Epsilon for numerical stability
        
    Returns:
        hit: Boolean
        p: Intersection point (3,) or None
    """
    # Step 1: Compute inverse transformation matrix
    # inv_transform_matrix = diag(1/S[0], 1/S[1], 1/S[2]) @ R.transpose()
    inv_scale = np.diag(1.0 / S)
    M_inv = inv_scale @ R.T
    
    # Step 2: Transform ray to ellipsoid space
    o_transformed = M_inv @ (o - t)
    d_transformed = M_inv @ d
    
    # Standard quadratic equation for ray-sphere intersection in local space:
    # A*t^2 + B*t + C = 0
    # where A = d'.dot(d'), B = 2*o'.dot(d'), C = o'.dot(o') - 1
    A = np.dot(d_transformed, d_transformed)
    B = 2.0 * np.dot(o_transformed, d_transformed)
    C = np.dot(o_transformed, o_transformed) - 1.0
    
    # Step 3: Handle degenerate direction (A near zero)
    if abs(A) < eps:
        A = eps
        
    # Step 4: Compute discriminant
    discriminant = B*B - 4*A*C
    
    if discriminant < 0:
        return False, None
        
    # Step 5: Handle near-zero discriminant
    if abs(discriminant) < eps:
        discriminant = 0.0
        
    # Step 6: Compute roots
    sqrt_discriminant = math.sqrt(discriminant)
    t1 = (-B - sqrt_discriminant) / (2 * A)
    t2 = (-B + sqrt_discriminant) / (2 * A)
    
    # Step 7: Check if both roots are negative
    if t1 < 0 and t2 < 0:
        return False, None
        
    # Step 8: Select valid t
    if t1 >= 0:
        t_intersect = t1
    else:
        t_intersect = t2
        
    if abs(t1 - t2) < eps:
        t_intersect = min(t1, t2)
        
    # Step 9: Compute intersection point in transformed space
    intersection_point_transformed = o_transformed + t_intersect * d_transformed
    
    # Step 10: Transform back to world space
    # transform_mat = R @ diag(S[0], S[1], S[2])
    scale_mat = np.diag(S)
    M_fwd = R @ scale_mat
    intersection_point = M_fwd @ intersection_point_transformed + t
    
    return True, intersection_point
```

### Parameters
- $\epsilon$: Epsilon for numerical stability, default $10^{-5}$ [S1, unofficial].

### Edge Cases and Pitfalls
- **Degenerate Direction:** If the transformed direction vector has a near-zero component used in the denominator, it can cause division by zero. The source specifies clamping $A$ to $\epsilon$ [S1, unofficial].
- **Near-Zero Discriminant:** A discriminant very close to zero can lead to numerical instability in the square root. It is clamped to zero if $|\text{discriminant}| < \epsilon$ [S1, unofficial].
- **Back-Facing Intersections:** Both roots being negative indicates the ellipsoid is behind the ray origin. This must be explicitly checked [S1, unofficial].
- **Precision:** Floating-point precision issues may arise when the ray is tangent to the ellipsoid or when the ellipsoid is very small or large relative to the ray origin.

## Operating characteristics
**Time Complexity:** $O(1)$ per ray-ellipsoid intersection, involving a constant number of matrix-vector multiplications and arithmetic operations.
**Memory Complexity:** $O(1)$ auxiliary space.
**Strengths:** Analytical solution is fast and robust for simple ellipsoids. No iterative convergence required.
**Limitations:** Only applicable to ellipsoids. Requires transformation of the ray, which involves matrix multiplications. Numerical stability requires careful handling of edge cases (tangent rays, degenerate directions).
**When to Use:** When rendering scenes with ellipsoidal primitives, such as in 3D Gaussian Splatting or other implicit surface representations where ellipsoids are the base primitive.
**When Not to Use:** For complex non-ellipsoidal shapes or when higher-order primitives are needed.

## Background
The method is based on the implementation found in the unofficial Taichi 3D Gaussian Splatting repository by wanmeihuali [S1, unofficial]. It applies the standard technique of transforming the ray into the local coordinate system of the ellipsoid (where it becomes a unit sphere) to simplify the intersection calculation to a quadratic equation.

## Sources
[S1] wanmeihuali/taichi_3d_gaussian_splatting, https://github.com/wanmeihuali/taichi_3d_gaussian_splatting, fetched 2026-10-27 (unofficial)
