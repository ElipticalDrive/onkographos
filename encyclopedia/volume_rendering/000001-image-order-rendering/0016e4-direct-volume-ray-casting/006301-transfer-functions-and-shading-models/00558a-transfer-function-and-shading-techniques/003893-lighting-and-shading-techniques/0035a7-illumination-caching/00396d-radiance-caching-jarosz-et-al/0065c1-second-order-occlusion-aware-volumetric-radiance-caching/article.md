# Second-Order Occlusion-Aware Volumetric Radiance Caching

Also called: Occ. aware, Occlusion-Aware Volumetric Radiance Caching

## Overview
Second-Order Occlusion-Aware Volumetric Radiance Caching is a radiance caching algorithm for participating media that uses second-order gradient analysis (Hessian) of light transport to adaptively sample and interpolate radiance from sparse cache points, accounting for occlusion changes [S1]. It solves the problem of suboptimal cache distribution in scenes with complex shadows by deriving a Hessian-based error metric that predicts extrapolation error more accurately than first-order gradient methods [S1]. Unlike its parent [Radiance Caching (Jarosz et al.)](8493-radiance-caching-jarosz-et-al.md), which relies on spherical harmonics and first-order gradients, this method uses a proxy triangulation-based representation to compute occlusion-aware derivatives, preventing the concentration of cache points near reflecting geometry and enabling efficient capture of high-frequency shadow boundaries [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Scene geometry and participating media properties (scattering coefficient $\mu_s$, phase function $f_s$).
*   Camera ray.
*   Error tolerance $\varepsilon$.
*   Ray marching step size (default 0.1 in 3D) [S1].
*   Samples per pixel (default 16 in 3D) [S1].

**Outputs:**
*   Radiance $L(x, \omega_o)$ at shaded points $x'$ in the medium.
*   Cache data structures (octrees) containing radiance, gradients, and Hessians at cache points.

### Data Structures
The method maintains two separate octrees for efficient cache query: one for single scattering and one for multiple scattering [S1]. Each cache point $x_k$ stores:
*   Position $x_k$.
*   Radiance $L(x_k)$.
*   Gradient $\nabla L(x_k)$.
*   Hessian $HL(x_k)$.
*   Valid region radius $R_k$ (derived from the Hessian eigenvalues) [S1].

### Method Steps

1.  **Proxy Triangulation Construction:**
    For a cache point $x$, compute the incoming light field using a proxy triangulation-based representation of the radiance reflected by the surrounding medium and geometry [S1].
    *   Stratified sampling: Compute radiance and derivatives at $x$ by stratified sampling uniformly in the sphere (or circle in 2D) with equal solid angle strata [S1].
    *   Subdivision: Build the subdivision using angular samples by connecting samples from adjacent strata [S1].
    *   Ray Marching: Perform subdivision within the medium by uniformly ray-marching the medium at discrete distances around $x$, joining adjacent angular samples within each marching step [S1].
    *   Assumptions: For a sufficiently fine subdivision, assume angle $\gamma \to 0$, so the direction $\omega_i$ is constant for the segment $\ell$: $f(x, \omega_i, \omega_o) = f(x, \omega_{\ell}, \omega_o)$, where $\omega_{\ell}$ is a fixed direction from $x$ to a point in segment $\ell$ [S1]. Assume constant transmittance and radiance for all $y \in \ell$: $Tr(x, y) = Tr(x, y_{\ell})$ and $L(y, \omega_i) = L(y_{\ell}, \omega_i)$, choosing $y_{\ell}$ as the furthest point in segment $\ell$ [S1].

2.  **Derivative Computation:**
    Efficiently compute the first- and second-order derivatives of the radiance at the cache points using the proxy representation, accounting for occlusion changes via a unified 3D geometry term using slanted triangular faces [S1].
    *   The radiance contribution from segment $\ell_j$ is approximated as:
        $$ L_j(x, \omega_o) \approx f(x, \omega_{\ell j}, \omega_o) Tr(x, y_{\ell j}) L(y_{\ell j}, \omega_i) F_{\ell j}(x) $$
        where $F_{\ell j}(x)$ is the integral of the geometry term $G(x,y)$ over the segment [S1].
    *   The total radiance and its derivatives are approximated by summing over rings $R$ and segments $\ell_j \in L_i$:
        $$ L(x, \omega_o) \approx \sum_{r_i \in R} \sum_{\ell_j \in L_i} \frac{L_j(x, \omega_o)}{\text{pdf}(r_i)} $$
        $$ \nabla L(x, \omega_o) \approx \sum_{r_i \in R} \sum_{\ell_j \in L_i} \frac{\nabla L_j(x, \omega_o)}{\text{pdf}(r_i)} $$
        $$ HL(x, \omega_o) \approx \sum_{r_i \in R} \sum_{\ell_j \in L_i} \frac{HL_j(x, \omega_o)}{\text{pdf}(r_i)} $$
        [S1]
    *   Derivatives of segment radiance $L_j$ are computed using chain rule expansions involving $L_F = F_{\ell} L_r$ and $L_r = L Tr$. For Lambertian surfaces and isotropic, homogeneous media, $\nabla L = HL = \nabla f = Hf = 0$, simplifying to:
        $$ \nabla L_j \approx L f (Tr \nabla F_{\ell} + \nabla Tr F_{\ell}) $$
        $$ HL_j \approx L f (Tr H F_{\ell} + \nabla Tr \nabla | F_{\ell} + \nabla F_{\ell} \nabla | Tr + H Tr F_{\ell}) $$
        [S1]
    *   Note: Computing derivatives of the surface-to-medium form factor involves operating with $3 \times 3$ matrices [S1].

3.  **Error Metric and Cache Point Placement:**
    Use a second-order Hessian-based error metric to determine when interpolation is appropriate and to control error during reconstruction [S1].
    *   The valid region for a cache point $x$ is defined by an ellipsoid (3D) or ellipse (2D) with principal radii derived from the eigenvalues $\lambda_i$ of $HL(x)$:
        $$ R^{2D}_i = \sqrt{\frac{4 L(x) \varepsilon}{\pi |\lambda_i|}} $$
        $$ R^{3D}_i = \sqrt{\frac{15 L(x) \varepsilon}{5 |\lambda_i|}} $$
        [S1]
    *   Prepopulate the cache by uniformly sampling a ray from the camera and ray-marching along the media. Place a cache point if the point does not fulfill the error metric (i.e., if it lies outside the valid regions of existing cache points) [S1].

4.  **Rendering and Interpolation:**
    At render time, evaluate radiance at ray-marched points $x'$ in the medium by extrapolating from surrounding valid cache points [S1].
    *   Let $C$ be the set of cache points whose radii include $x'$.
    *   Interpolate in linear space using a cubic kernel $w(x_k, x') = 3d^2 - 2d^3$ with $d = 1 - \frac{\|x' - x_k\|}{R_k}$:
        $$ L(x') \approx \frac{\sum_{k \in C} [L(x_k) + \nabla L(x_k) \cdot \Delta x'] w(x_k, x')}{\sum_{k \in C} w(x_k, x')} $$
        [S1]
    *   If no valid cache points are found for $x'$, compute its radiance and derivatives (Steps 1-2) and add it to the cache [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_segment_radiance_derivatives(x, segment, media_props, geometry):
    """
    Computes L_j, grad_L_j, hess_L_j for a segment l_j.
    Assumes Lambertian surfaces and isotropic, homogeneous media.
    """
    # Constants from segment definition
    f_val = media_props.scattering_coeff * media_props.phase_function(segment.direction)
    Tr_val = geometry.transmittance(x, segment.furthest_point)
    L_val = geometry.radiance_at(segment.furthest_point)
    
    # Geometry term integrals and derivatives (F_l, grad_F_l, hess_F_l)
    # These require 3x3 matrix operations for 3D form factors [S1]
    F_l = geometry.integrate_geometry_term(x, segment)
    grad_F_l = geometry.gradient_geometry_term(x, segment)
    hess_F_l = geometry.hessian_geometry_term(x, segment)
    
    # Simplified derivatives for homogeneous media (L, f constant)
    # L_j approx L * f * (Tr * F_l)
    # grad_L_j approx L * f * (Tr * grad_F_l + grad_Tr * F_l)
    # hess_L_j approx L * f * (Tr * hess_F_l + grad_Tr * grad_F_l + grad_F_l * grad_Tr + hess_Tr * F_l)
    
    grad_Tr = geometry.gradient_transmittance(x, segment.furthest_point)
    hess_Tr = geometry.hessian_transmittance(x, segment.furthest_point)
    
    L_j = L_val * f_val * Tr_val * F_l
    grad_L_j = L_val * f_val * (Tr_val * grad_F_l + grad_Tr * F_l)
    hess_L_j = L_val * f_val * (
        Tr_val * hess_F_l + 
        outer_product(grad_Tr, grad_F_l) + 
        outer_product(grad_F_l, grad_Tr) + 
        hess_Tr * F_l
    )
    
    return L_j, grad_L_j, hess_L_j

def compute_cache_point_derivatives(x, media_props, geometry, num_angular_samples=64, num_ray_steps=16):
    """
    Computes L, grad_L, hess_L at point x using stratified sampling and proxy triangulation.
    """
    total_L = 0.0
    total_grad_L = Vector3(0, 0, 0)
    total_hess_L = Matrix3(0, 0, 0, 0, 0, 0, 0, 0, 0)
    
    # Stratified sampling in sphere/circle
    strata = stratified_sample_sphere(num_angular_samples, num_ray_steps)
    
    for stratum in strata:
        # Build proxy triangle/trapezoid from angular samples and ray march steps
        segment = build_segment(stratum, x)
        
        # Compute contribution
        L_j, grad_L_j, hess_L_j = compute_segment_radiance_derivatives(x, segment, media_props, geometry)
        
        # Accumulate (pdf is handled by stratification weights, assumed 1 for uniform stratification here)
        total_L += L_j
        total_grad_L += grad_L_j
        total_hess_L += hess_L_j
        
    return total_L, total_grad_L, total_hess_L

def get_valid_radius(L, hess_L, epsilon):
    """
    Computes the principal radii of the valid region ellipsoid.
    Returns the minimum radius (most restrictive) for isotropic caching.
    """
    eigenvalues = hess_L.eigenvalues()
    # 3D formula: R_i = sqrt(15 * L * epsilon / (5 * |lambda_i|))
    # Using the smallest eigenvalue magnitude for the smallest radius (isotropic case)
    min_abs_eigenvalue = min(abs(e) for e in eigenvalues)
    if min_abs_eigenvalue == 0:
        return float('inf')
    R = math.sqrt((15 * L * epsilon) / (5 * min_abs_eigenvalue))
    return R

def render_volumetric_radiance_caching(camera_ray, cache_single, cache_multi, epsilon, step_size):
    """
    Renders a single ray using the radiance cache.
    """
    # Ray march along the camera ray
    t = 0.0
    while t < ray_length(camera_ray):
        x = camera_ray.origin + t * camera_ray.direction * step_size
        t += step_size
        
        # Query single scattering cache
        L_single = interpolate_from_cache(x, cache_single, epsilon)
        
        # Query multiple scattering cache
        L_multi = interpolate_from_cache(x, cache_multi, epsilon)
        
        # If no valid points, compute and add to cache
        if L_single is None:
            L, grad_L, hess_L = compute_cache_point_derivatives(x, media_props, geometry)
            R = get_valid_radius(L, hess_L, epsilon)
            cache_single.insert(x, L, grad_L, hess_L, R)
            L_single = L
            
        if L_multi is None:
            L, grad_L, hess_L = compute_cache_point_derivatives(x, media_props, geometry, is_multiple_scattering=True)
            R = get_valid_radius(L, hess_L, epsilon)
            cache_multi.insert(x, L, grad_L, hess_L, R)
            L_multi = L
            
        if L_single is not None and L_multi is not None:
            return L_single + L_multi
            
    return 0.0

def interpolate_from_cache(x, octree, epsilon):
    """
    Interpolates radiance from valid cache points in the octree.
    """
    # Find all cache points whose valid radius includes x
    valid_points = octree.query_radius(x, epsilon)
    
    if not valid_points:
        return None
        
    numerator = 0.0
    denominator = 0.0
    
    for point in valid_points:
        dist = distance(x, point.position)
        d = 1.0 - dist / point.radius
        if d < 0:
            continue
            
        w = 3 * d * d - 2 * d * d * d
        L_pred = point.L + dot(point.grad_L, x - point.position)
        
        numerator += L_pred * w
        denominator += w
        
    if denominator == 0:
        return None
        
    return numerator / denominator
```

### Parameters
*   $\varepsilon$: Tolerance value controlling error in radiance caching. Typical value: $1.5 \times 10^{-4}$ for the Patio scene [S1].
*   Ray marching step size: Uniform step size of 0.1 used in 3D results unless stated otherwise [S1].
*   Samples per pixel: 16 samples per pixel used in 3D results unless stated otherwise [S1].
*   Angular samples: The accuracy of the subdivision relies on dense sampling of the angular domain. Specific numbers vary, but 256 angular samples were used for reference gradient computation in 2D validation [S1].

### Edge Cases and Pitfalls
*   **Occlusion Ignorance:** Ignoring visibility derivatives leads to poor cache distribution and imprecise extrapolation during reconstruction, failing to represent high-frequency shadows and rapid radiance changes [S1]. Occlusion-unaware methods fail to estimate gradients in complex shadows and tend to concentrate cache points near reflecting geometry, requiring radius-clamping heuristics to mitigate oversampling [S1].
*   **Penumbra Regions:** Occlusion-unaware gradients ignore light effects in penumbra regions, causing extrapolation failures and visible artifacts on shadow patterns [S1].
*   **Sampling Rate Limits:** The sampling rate limits the amount of radiance changes that can be recovered, especially for fine details like small light sources not computed using Next Event Estimation (NEE) [S1].
*   **Multiple Scattering Variance:** Multiple scattering is limited to the second bounce to reduce excessive variance when computing reference derivatives with finite differences [S1].
*   **Isotropic Cost:** Using isotropic points with the smallest principal axis of the Hessian is the most costly scenario compared to previous work because it requires more points [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time:** The method reduces computation time by up to 30% compared to previous work (Jarosz et al. [2008]) while providing improved results [S1]. In the 'Statues' scene, it took 154 minutes with ~19k cache points, compared to 155 minutes for Jarosz et al. [2008] with similar point counts [S1]. In the Patio scene, anisotropic points reduced total computation time by 30% compared to isotropic points at the same error tolerance [S1].
*   **Memory:** Memory usage is proportional to the number of cache points. The occlusion-aware method requires significantly fewer points than occlusion-unaware methods for comparable results (e.g., 3k vs 35k in the Strips scene) [S1].

### Measured Performance
*   **Statues Scene:** ~19k cache points, 16k samples/cache, 154 minutes [S1].
*   **Patio Scene:** Anisotropic cache: 21k points, 81 minutes (cache gen) + 13 min (render) = 94 min total. Isotropic cache: 32k points, 122 min (cache gen) + 13 min (render) = 135 min total. Jarosz et al. [2008]: 36k points, 124 min (cache gen) + 12 min (render) = 136 min total [S1].
*   **Strips Scene:** Occlusion-unaware required 35k points; occlusion-aware required 3k points [S1].
*   **2D Overlapping Shadows:** Anisotropic technique reduced cache size by 32% (~13k points) compared to isotropic points at the same error threshold [S1].
*   **2D Square Light:** Anisotropic cache reduced the number of cache points by up to 20% compared to isotropic cache at the same error threshold [S1].

### Strengths and Limitations
*   **Strengths:** Correctly captures complex radiance changes and high-frequency shadows by placing more points near shadow boundaries. Does not rely on radius-clamping heuristics. Interpolates in linear space, where the Hessian-based metric more accurately predicts error [S1].
*   **Limitations:** Isotropic caching is costly. Sampling rate limits recovery of fine details from small light sources. Multiple scattering is limited to two bounces to control variance [S1].

### When to Use
*   Use for scenes with complex occlusions, shadows, and penumbra regions in participating media.
*   Use when high-frequency shadow details are critical and first-order gradient methods fail.
*   Use when computational efficiency is important and anisotropic caching can be employed.

### When Not to Use
*   Avoid if the scene has very small light sources that are not handled by NEE, as sampling rate limits may cause artifacts.
*   Avoid if multiple scattering beyond two bounces is required without significant variance reduction techniques.

## Background

### Origin
Second-Order Occlusion-Aware Volumetric Radiance Caching was introduced by Julio Marco, Adrian Jarabo, Wojciech Jarosz, and Diego Gutierrez in 2018 in the paper 'Second-Order Occlusion-Aware Volumetric Radiance Caching' published in ACM Trans. Graph. 37, 2 [S1].

### History and Development
The method is based on previous work by Jarosz et al. [2008], which this work extends with occlusion-aware second-order derivatives [S1]. Previous methods ignored occlusion/visibility changes or relied only on first-order derivatives, leading to suboptimal cache distributions in scenes with occluders [S1]. This method derives a 2D theory of light transport in participating media to validate the approach before generalizing to 3D [S1].

### Usage
The method is used in ACM Transactions on Graphics, Vol. 37, No. 2, Article 20, February 2018 [S1].

## Sources

[S1] Second-Order Occlusion-Aware Volumetric Radiance Caching, https://graphics.unizar.es/papers/2018TOG_SecondOrderMediaCaching.pdf, fetched 2026-10-29
