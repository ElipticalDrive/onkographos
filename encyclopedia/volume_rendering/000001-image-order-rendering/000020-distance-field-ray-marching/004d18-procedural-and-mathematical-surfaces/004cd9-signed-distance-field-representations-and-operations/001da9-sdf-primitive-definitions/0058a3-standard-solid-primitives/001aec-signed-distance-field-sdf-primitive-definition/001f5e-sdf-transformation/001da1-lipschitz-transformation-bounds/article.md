# Lipschitz Transformation Bounds

Also called: Distances to Transformed Objects, SDF Transformation with Lipschitz Bounds, Transformed implicit surfaces

## Overview
Lipschitz Transformation Bounds provide a method for computing accurate distance fields for implicitly defined surfaces after applying spatial transformations (such as scaling, rotation, or deformation) by adjusting the distance estimate using the Lipschitz constant of the inverse transformation [S1]. Unlike standard Euclidean ray marching which assumes the metric is preserved, this method compensates for the distortion introduced by non-isometric transforms, preventing ray tunneling and ensuring the distance function remains a valid lower bound [S1]. It sets itself apart from [Uniform Scaling of SDFs](2394-uniform-scaling-of-sdfs.md) by handling general non-uniform and non-linear deformations, and from [Spatial and nD transforms](3106-spatial-and-nd-transforms.md) by explicitly deriving and applying the Lipschitz bound to correct the distance value rather than just transforming the query point [S1].

## How it works
**Inputs and Outputs:**
- Input: A base Signed Distance Field (SDF) function $f(\mathbf{x})$, a spatial transformation $T(\mathbf{x})$, and a query point $\mathbf{x}$ in the transformed space.
- Output: The corrected distance $d$ to the transformed implicit surface at $\mathbf{x}$.

**Data Structures:**
- The transformation $T$ is defined by its inverse $T^{-1}$, which maps the query point back to the object's local space.
- The Lipschitz constant $L$ is a scalar value derived from the maximum norm of the Jacobian of $T^{-1}$ over the relevant domain.

**Method Steps:**
1. **Define the Transformed Surface:** The transformed implicit surface is defined as the set of points where the base function evaluated at the inverse-transformed coordinates is zero [S1]:
   $$ f(T^{-1}(\mathbf{x})) = 0 \quad \text{(Eq. 44)} $$
2. **Compute Local Distance:** Evaluate the base SDF at the inverse-transformed point to get the uncorrected distance:
   $$ d_{local} = f(T^{-1}(\mathbf{x})) $$
   *Note: In practice, $T^{-1}(\mathbf{x})$ is computed, and $f$ is evaluated there.*
3. **Determine Lipschitz Constant ($L$):** Calculate the Lipschitz constant of the inverse transformation $T^{-1}$. The Lipschitz constant of the composition is no greater than the product of the component Lipschitz constants [S1]. The specific method for determining $L$ depends on the type of transformation:
   - **Isometry:** If $T$ is an isometry (rotation, translation, reflection), distances are preserved. The distance returned by the base SDF needs no adjustment [S1]:
     $$ d(T^{-1}(\mathbf{0})) = d(T^{-1}(\mathbf{x}) - T^{-1}(\mathbf{0})) \quad \text{(Eq. 45)} $$
     Here, the effective Lipschitz constant is 1.
   - **Uniform Scale:** If $T$ is a uniform scale by factor $s$, the inverse is a scale by $1/s$. The distance is scaled by $1/s$ [S1]:
     $$ d(s T^{-1}(\mathbf{0})) = \frac{1}{s} d(T^{-1}(\mathbf{x}) - T^{-1}(\mathbf{0})) \quad \text{(Eq. 47)} $$
     The Lipschitz constant of the inverse scale is $1/s$.
   - **Linear Deformation:** For a general linear transformation, the Lipschitz constant is the largest singular value of the inverse transformation matrix, found via the power method [S1, G].
   - **Taper:** For a taper deformation scaling two axes by a function $\alpha(z)$ of the third axis, the Lipschitz constant is determined by the "tightest" tapering [S1]:
     $$ Lip_{taper} = \min \frac{1}{\alpha(z)} \quad \text{(Eq. 49)} $$
     See [Lipschitz Bound Computation for Tapering Transformation](#lipchitz-bound-computation-for-tapering-transformation) for the exact Jacobian norm calculation.
   - **Twist:** For a twist deformation rotating two axes by a linear function $\theta(z)$ of the third axis, the transformation is not Lipschitz over unbounded domains. It must be constrained to a domain like a unit cylinder along the twisting axis. The Lipschitz constant is computed from the worst-case scenario within these bounds [S1]:
     $$ Lip_{twist} = \sqrt{4 + \frac{\theta^2}{4}} \quad \text{(Eq. 51)} $$
4. **Apply Bound Correction:** The final distance is the local distance divided by the Lipschitz constant $L$ of the inverse transformation. This ensures the resulting function satisfies the 1-Lipschitz condition (or a known bound), providing a safe lower bound for ray marching [S1, S3]:
   $$ d_{corrected} = \frac{d_{local}}{L} $$
   If the base SDF is not 1-Lipschitz, it must first be normalized by its own bound $K$ (see [Lipschitz Normalization](#lipchitz-normalization)) before applying the transformation bound.

**Parameters:**
- $s$: Uniform scale factor.
- $\alpha(z)$: Tapering coefficient function along the vertical axis.
- $\theta$: Twist angle parameter.

**Edge Cases and Pitfalls:**
- **Unbounded Twist:** Twisting is not Lipschitz on unbounded domains because the distance between transformed points can grow arbitrarily large relative to the original distance. It must be constrained to a specific domain (e.g., a unit cylinder) to compute a valid bound [S1].
- **Precision:** The Lipschitz constant must be computed accurately. For tapering, interval analysis or exact derivative evaluation is recommended to avoid underestimating the bound, which would cause ray tunneling [S2].
- **Composition:** When multiple transformations are composed, the total Lipschitz constant is the product of the individual constants. Care must be taken to compute the bound of the *inverse* composition, which is not simply the inverse of the forward composition's bound [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_lipschitz_bound_transformation_distance(base_sdf, inverse_transform, query_point, transform_type, params):
    """
    Computes the distance to a transformed implicit surface using Lipschitz bounds.
    
    Args:
        base_sdf: Function f(x) returning the distance to the base object.
        inverse_transform: Function T_inv(x) returning the inverse-transformed coordinates.
        query_point: The point x in world space to query.
        transform_type: String identifying the transformation ('isometry', 'uniform_scale', 'taper', 'twist').
        params: Dictionary of transformation parameters (e.g., 'scale', 'alpha_func', 'theta').
        
    Returns:
        The corrected distance to the transformed surface.
    """
    # Step 1: Transform query point to local space
    local_point = inverse_transform(query_point)
    
    # Step 2: Evaluate base SDF
    local_distance = base_sdf(local_point)
    
    # Step 3: Determine Lipschitz constant L of the inverse transformation
    if transform_type == 'isometry':
        L = 1.0
    elif transform_type == 'uniform_scale':
        s = params['scale']
        L = 1.0 / s
    elif transform_type == 'taper':
        # Compute min(1/alpha(z)) over the relevant domain
        # This requires evaluating the taper function and its derivative
        alpha_func = params['alpha_func']
        # Example: if alpha(z) is linear, find the minimum value in the domain
        # For complex cases, use interval analysis or numerical optimization
        min_alpha = compute_min_alpha_taper(params['domain'], alpha_func)
        if min_alpha == 0:
            raise ValueError("Taper function alpha(z) cannot be zero")
        L = 1.0 / min_alpha
    elif transform_type == 'twist':
        theta = params['theta']
        # Constrained to unit cylinder; worst-case bound
        L = math.sqrt(4 + (theta ** 2) / 4)
    else:
        # Generic linear deformation: compute largest singular value of Jacobian of T_inv
        # This involves computing the Jacobian matrix and finding its spectral norm
        jacobian = compute_jacobian_inverse_transform(inverse_transform, local_point)
        L = compute_spectral_norm(jacobian)
        
    # Step 4: Apply correction
    # Ensure distance is non-negative if we are outside the object
    # Note: base_sdf should return signed distance; for ray marching, we often use abs or check sign
    corrected_distance = local_distance / L
    
    return corrected_distance

def compute_min_alpha_taper(domain, alpha_func):
    """
    Computes the minimum value of alpha(z) over a given domain.
    This is a placeholder for the specific analytical or numerical method
    required for the taper function.
    """
    # For a linear taper alpha(z) = a*z + b, the min is at one of the endpoints
    # For more complex functions, use interval analysis or optimization
    # [S2] suggests developing the equation or using interval analysis
    return alpha_func(domain.min_z) # Simplified example
```

## Subtopics

### [Warped-Space SDF Ray Marching](4726-warped-space-sdf-ray-marching.md)
Warped-Space SDF Ray Marching is a technique for rendering scenes with regularly spaced geometry (such as urban environments) by defining the scene in a sheared coordinate space ("road space") where buildings are axis-aligned, simplifying distance field computation. It differs from standard Euclidean ray marching by applying a safety factor to the computed distances to compensate for the non-Euclidean metric introduced by the spatial transformation, preventing the ray from tunneling through walls. It is a specific application of Lipschitz bounds where the transformation is a shear.

### [SDF Deformations](5879-sdf-deformations.md)
SDF Deformations apply a non-uniform spatial distortion to a Signed Distance Field by warping the coordinate space, allowing for the creation of twisted, bent, or otherwise distorted geometric shapes. It solves the problem of generating complex, organic, or mechanical deformations (such as twisted cables) from simple primitive SDFs without requiring explicit mesh topology. It is distinguished from [Warped-Space SDF Ray Marching](4726-warped-space-sdf-ray-marching.md) by its explicit handling of multiphase SDFs and the mathematical constraint of the Lipschitz constant to preserve distance accuracy.

### Lipschitz Bound Computation for Tapering Transformation
A method to compute the bound of the norm of the Jacobian for a tapering transformation to determine the Lipschitz bound [S2]. For a tapering transformation $\omega^{-1}(\mathbf{p}) = (\alpha(z)x, \alpha(z)y, z)$ where $\beta = \alpha'$, the Jacobian matrix is:
$$ \mathbf{J}_{\omega^{-1}} = \begin{pmatrix} \alpha & 0 & \beta \\ 0 & \alpha & 0 \\ \beta & 0 & 1 \end{pmatrix} $$
The Euclidean norm of the Jacobian is:
$$ \|\mathbf{J}_{\omega^-}\| = \frac{1}{\sqrt{2}}\sqrt{\alpha^2 + 2\beta^2 + 1 + (1 + \alpha)\sqrt{\alpha^2 - 2\alpha + 4\beta^2 + 1}} $$
For a given input interval, this norm can be obtained by developing the equation or by interval analysis [S2].

### Lipschitz Normalization
A preprocessing step that transforms any signed distance function $f$ into a lower-bound distance function satisfying the 1-Lipschitz condition by dividing it by a known Lipschitz bound $K$ [S3]. The formula is:
$$ f_{normalized} = \frac{f}{K} $$
where $K$ is the Lipschitz bound of the original field function $f$, ensuring $\|\nabla f\| \le 1$ everywhere [S3]. The value of $K$ depends on the primitive/operator and must be computed or known a priori [S3].

## Operating characteristics
- **Time Complexity:** The core computation involves evaluating the inverse transformation and the base SDF, which is $O(1)$ per query. Computing the Lipschitz constant may require additional operations (e.g., eigenvalue decomposition for linear transforms, or interval analysis for tapering), but these are typically precomputed or cached for uniform transformations.
- **Memory Complexity:** $O(1)$ for storing the Lipschitz constant. No additional data structures are required beyond those for the base SDF.
- **Measured Performance:** Performance depends on the complexity of the base SDF and the transformation. For simple primitives, the overhead of computing the Lipschitz bound is negligible. For complex deformations, the cost of evaluating the Jacobian or its bound may become significant.
- **Scaling:** The method scales well with scene complexity as it is a local operation. However, for scenes with many different transformations, precomputing and caching Lipschitz bounds is recommended.
- **Strengths:** Provides mathematically guaranteed lower bounds for distances, preventing ray tunneling in transformed spaces. Works with any transformation where the Lipschitz constant of the inverse can be computed.
- **Limitations:** Requires the Lipschitz constant to be known or computable. For complex non-linear transformations, computing the bound can be difficult. Twisting requires domain constraints.
- **When to Use:** When rendering scenes with scaled, rotated, or deformed SDF primitives where distance accuracy is critical.
- **When Not to Use:** When the transformation is complex and the Lipschitz bound is difficult to compute or varies significantly across the domain, making a single bound too conservative.

## Background
The concept of using Lipschitz constants to bound distances in transformed spaces originates from the foundational work on Sphere Tracing by John C. Hart et al. [S1]. The paper "Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces" (1992) introduced the use of Lipschitz bounds to ensure the safety of the sphere tracing algorithm when applied to transformed implicit surfaces. The specific formulas for isometries, uniform scales, tapers, and twists were derived in this context. Later work, such as "Segment Tracing Using Local Lipschitz Bounds" [S2] and "Lipschitz Pruning: Hierarchical Simplification of Primitive‐Based SDFs" [S3], extended these ideas to more complex deformations and hierarchical scene structures.

## Sources
[S1] Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S2] Segment Tracing Using Local Lipschitz Bounds, https://doi.org/10.1111/cgf.13951, fetched 2026-10-26
[S3] Lipschitz Pruning: Hierarchical Simplification of Primitive‐Based SDFs, https://doi.org/10.1111/cgf.70057, fetched 2026-10-26
