# Lipschitz Transformation Bounds

Also called: Distances to Transformed Objects, SDF Transformation with Lipschitz Bounds, Transformed implicit surfaces

## Overview
Lipschitz Transformation Bounds provide a method to compute valid step sizes for ray marching through implicitly defined surfaces that have undergone spatial transformations. It solves the problem that standard Euclidean distance queries return incorrect step sizes when the underlying space is distorted, preventing ray tunneling. It differs from [Uniform Scaling of SDFs](2394-uniform-scaling-of-sdfs.md) by handling non-uniform and non-linear deformations via the Jacobian norm, and from [Spatial and nD transforms](3106-spatial-and-nd-transforms.md) by explicitly deriving the Lipschitz constant required for safety rather than just applying the inverse transform.

## How it works
**Inputs:**
- A transformation function $T(\mathbf{x})$ and its inverse $T^{-1}(\mathbf{x})$.
- A base SDF function $d(\mathbf{x})$ for the untransformed primitive.
- The ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (for marching context).

**Outputs:**
- A safe step size $\Delta t$ for ray marching.
- The transformed distance value.

**Data Structures:**
- None required beyond standard vector/matrix types. The core data is the Lipschitz constant $L$, a scalar.

**Method:**
The distance to a transformed implicit surface defined by $f(T^{-1}(\mathbf{x})) = 0$ is not simply $d(T^{-1}(\mathbf{x}))$. The step size must be bounded by the inverse of the Lipschitz constant of the inverse transformation $T^{-1}$ [S1].

1.  **Identify Transformation Type:** Determine if $T$ is an isometry, uniform scale, linear deformation, taper, or twist.
2.  **Compute Lipschitz Constant ($L$):** Calculate the Lipschitz constant of $T^{-1}$. This represents the maximum stretching factor of the inverse transformation.
    -   **Isometry:** If $T$ preserves distances (rotation, translation, reflection), $L = 1$. The distance is exact: $d(T^{-1}(\mathbf{0})) = d(T^{-1}(\mathbf{x}) - T^{-1}(\mathbf{0}))$ [S1].
    -   **Uniform Scale:** If $T(\mathbf{x}) = s\mathbf{x}$, the inverse is scaling by $1/s$. The Lipschitz constant is $L = 1/s$. The distance is scaled: $d(s T^{-1}(\mathbf{0})) = \frac{1}{s} d(T^{-1}(\mathbf{x}) - T^{-1}(\mathbf{0}))$ [S1].
    -   **Linear Deformation:** For a linear matrix $M$, $L$ is the largest singular value of $M^{-1}$, found via the power method on $M^{-T}M^{-1}$ [S1].
    -   **Taper:** Defined as $\text{taper}(\mathbf{x}) = (\alpha(z)x, \alpha(z)y, z)$. The Lipschitz constant is determined by the tightest tapering: $Lip_{taper} = \min \frac{1}{\alpha(z)}$ over the domain [S1].
    -   **Twist:** Defined by rotation angle $\theta(z)$. Twisting is not Lipschitz on unbounded domains. It must be constrained to a domain like a unit cylinder along the twisting axis. The bound is $Lip_{twist} = \sqrt{4 + \frac{\theta^2}{4}}$ [S1].
3.  **Calculate Safe Step:** The maximum safe step size $\Delta t$ is given by $\Delta t = \frac{d(T^{-1}(\mathbf{x}))}{L}$ [S1].
4.  **March:** Advance the ray by $\Delta t$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_lipschitz_bound(transformation_type, params):
    """
    Returns the Lipschitz constant L for the inverse transformation.
    transformation_type: 'isometry', 'uniform_scale', 'taper', 'twist'
    params: dict containing necessary parameters (e.g., scale factor, angle)
    """
    if transformation_type == 'isometry':
        return 1.0
    elif transformation_type == 'uniform_scale':
        s = params['scale']
        return 1.0 / s
    elif transformation_type == 'taper':
        # L is the minimum of 1/alpha(z) over the relevant domain
        # For implementation, this requires evaluating alpha(z) at the 
        # tightest point of the taper within the ray's current segment.
        # Not given in the sources as a closed form for arbitrary z.
        return params['min_inverse_alpha'] 
    elif transformation_type == 'twist':
        theta = params['max_angle']
        return sqrt(4.0 + (theta * theta) / 4.0)
    else:
        raise ValueError("Unsupported transformation type")

def ray_march_transformed_sdf(ray_origin, ray_direction, sdf_func, transformation, params):
    """
    Performs one step of ray marching using Lipschitz bounds.
    """
    # 1. Get current point
    current_point = ray_origin
    
    # 2. Apply inverse transformation to find position in base space
    base_point = transformation.inverse(current_point)
    
    # 3. Query base SDF
    dist_base = sdf_func(base_point)
    
    # 4. Compute Lipschitz constant of the inverse transformation
    L = get_lipschitz_bound(transformation.type, params)
    
    # 5. Compute safe step size
    # If dist_base is negative (inside), we might want to handle it differently,
    # but for marching out, we use the magnitude.
    safe_step = abs(dist_base) / L
    
    # 6. Advance ray
    next_point = current_point + ray_direction * safe_step
    
    return next_point, safe_step
```

**Edge Cases and Pitfalls:**
-   **Twisting on Unbounded Domains:** Twisting is not Lipschitz over infinite space. It must be constrained to a bounded domain (e.g., a unit cylinder) to define a valid $L$ [S1].
-   **Precision:** The Lipschitz constant is an upper bound. Using a loose bound results in smaller steps (slower convergence) but safety. Using a bound that is too tight causes tunneling.
-   **Non-Linear Taper:** The Lipschitz constant for tapering depends on the derivative $\alpha'(z)$. It must be computed per interval or via interval analysis [S2].

## Subtopics

### [Warped-Space SDF Ray Marching](4726-warped-space-sdf-ray-marching.md)
Warped-Space SDF Ray Marching renders scenes with regularly spaced geometry (e.g., urban environments) by defining the scene in a sheared coordinate space ("road space") where buildings are axis-aligned. It differs from standard Euclidean ray marching by applying a safety factor (Lipschitz bound) to the computed distances to compensate for the non-Euclidean metric introduced by the spatial transformation, preventing the ray from tunneling through walls.

### Lipschitz Bound Computation for Tapering Transformation
A method to compute the bound of the norm of the Jacobian for a tapering transformation to determine the Lipschitz bound. For a tapering transformation $\omega^{-1}(\mathbf{p}) = (\alpha(z)x, \alpha(z)y, z)$, the Jacobian matrix is:
$$ \mathbf{J}_{\omega^{-1}} = \begin{pmatrix} \alpha & 0 & \beta \\ 0 & \alpha & 0 \\ \beta & 0 & 1 \end{pmatrix} $$
where $\beta = \alpha'$. The Euclidean norm (Lipschitz bound) is:
$$ \|\mathbf{J}_{\omega^-}\| = \frac{1}{\sqrt{2}}\sqrt{\alpha^2 + 2\beta^2 + 1 + (1 + \alpha)\sqrt{\alpha^2 - 2\alpha + 4\beta^2 + 1}} $$
For a given input interval, this norm can be obtained by developing the equation or by interval analysis [S2].

### SDF Deformations
Non-uniform spatial distortion is applied by modifying space as a function of position. An example of a twist deformation in GLSL is:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```glsl
float fScene(vec3 pt) {
    pt.y -= 1;
    float t = (pt.y + 2.5) * sin(time);
    return sdCube(vec3(
        pt.x * cos(t) - pt.z * sin(t),
        pt.y / 2,
        pt.x * sin(t) + pt.z * cos(t)), vec3(1));
}
```
This applies a rotation that varies with the y-coordinate, effectively twisting the cube [S3, unofficial].

## Operating characteristics
-   **Time Complexity:** $O(1)$ per step for isometry and uniform scale. $O(k)$ for linear deformation (power method iterations). $O(1)$ for taper/twist if precomputed or simple analytic bounds are used.
-   **Memory Complexity:** $O(1)$.
-   **Strengths:** Guarantees no tunneling for transformed primitives. Simple to implement for common transforms.
-   **Limitations:** The bound is a worst-case estimate, which can lead to smaller step sizes than necessary, increasing the number of samples. Non-linear transforms require careful evaluation of the Lipschitz constant over the ray segment.
-   **When to Use:** When rendering transformed SDF primitives (rotated, scaled, sheared, twisted) in ray marching.
-   **When Not to Use:** For simple translations/rotations where the bound is 1, standard SDF evaluation is sufficient. For highly complex non-linear deformations where computing the Jacobian norm is expensive, other methods might be preferred.

## Background
The concept originates from the geometric ray tracing method for implicit surfaces, specifically in the context of "Sphere Tracing" [S1]. The work by Williams et al. established that the Lipschitz constant of the composition of functions is bounded by the product of the component Lipschitz constants, and specifically that the inverse transformation's Lipschitz constant dictates the safe step size. Later work by [S2] extended this to segment tracing using local Lipschitz bounds for more efficient rendering. [S3, unofficial] provides practical GLSL implementations of these deformations.

## Sources
[S1] Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S2] Segment Tracing Using Local Lipschitz Bounds, https://doi.org/10.1111/cgf.13951, fetched 2026-10-26
[S3] PDF Further Graphics - University of Cambridge, https://www.cl.cam.ac.uk/teaching/1920/FGraphics/1.%20Ray%20Marching%20and%20Signed%20Distance%20Fields.pdf, fetched 2026-10-27 (unofficial)
