# Gaussian Integration for Volume Rendering
Also called: Gaussian Quadrature Integration, Gaussian Quadrature for Volume Rendering

## Overview
Gaussian integration approximates the volume rendering integral as an unequally weighted sum of the integrand evaluated at $n$ unequally spaced sample points [S1, book]. It is chosen over equally spaced sampling (such as the rectangle rule) when the transfer function and density vary smoothly, as it achieves greater accuracy for the same number of sample points by exploiting the degrees of freedom in sample positions and weights [S1, book].

## How it works
**Inputs:**
*   Ray origin $\mathbf{x}$ and direction $\boldsymbol{\omega}_0$.
*   Near and far bounds $t_n, t_f$ defining the integration interval.
*   Number of quadrature points $N$.
*   Precomputed quadrature nodes $\xi_k \in [-1, 1]$ and weights $w_k$ for the chosen order $N$.

**Outputs:**
*   Estimated radiance $L(\mathbf{x}, \boldsymbol{\omega}_0)$.

**Data Structures:**
*   Quadrature nodes: Array of size $N$, values in $[-1, 1]$.
*   Quadrature weights: Array of size $N$, summing to 2 (for standard Legendre-Gauss).
*   Sample positions: Array of size $N$, computed in world space.

**Steps:**
1.  **Map nodes to ray interval:** Transform the standard quadrature nodes $\xi_k \in [-1, 1]$ to the ray parameter interval $[t_n, t_f]$. The sample position $\mathbf{x}_k$ along the ray is:
    $$ \mathbf{x}_k = \mathbf{x} + \frac{t_f - t_n}{2} (\xi_k + 1) \boldsymbol{\omega}_0 $$
    The corresponding distance step $\Delta \mathbf{x}_k$ associated with node $k$ is derived from the spacing between nodes. For a uniform mapping of the interval, the distance between adjacent quadrature points $\mathbf{x}_j$ and $\mathbf{x}_{j+1}$ is:
    $$ \Delta \mathbf{x}_j = \|\mathbf{x}_{j+1} - \mathbf{x}_j\| = \frac{t_f - t_n}{2} (\xi_{j+1} - \xi_j) $$
    Note: The boundary conditions for the first and last segments require careful handling of $\Delta \mathbf{x}_0$ and $\Delta \mathbf{x}_{N-1}$ relative to the ray bounds, or the weights $w_k$ are scaled to account for the interval length directly. The standard formulation [S2, unofficial] uses $\Delta \mathbf{x}_j$ as the distance between quadrature points $j$ and $j+1$.

2.  **Evaluate integrand:** For each sample point $k=1 \dots N$:
    *   Sample density $\sigma(\mathbf{x}_k)$ from the volume.
    *   Sample source radiance $L_s(\mathbf{x}_k, \boldsymbol{\omega}_0)$ (including emission and accumulated light).

3.  **Compute Transmittance:** Calculate the accumulated transmittance $\hat{T}(\mathbf{x}, \mathbf{x}_k)$ from the ray origin to sample $k$. This is the product of extinction factors for all preceding segments:
    $$ \hat{T}(\mathbf{x}, \mathbf{x}_k) = \prod_{j=1}^{k-1} e^{-\sigma(\mathbf{x}_j) \Delta \mathbf{x}_j} $$
    For $k=1$, $\hat{T}(\mathbf{x}, \mathbf{x}_1) = 1$.

4.  **Compute Contribution:** Calculate the contribution of each sample. The term $(1 - e^{-\sigma(\mathbf{x}_k) \Delta \mathbf{x}_k})$ represents the probability of scattering within the segment associated with sample $k$. The radiance contribution is:
    $$ C_k = \hat{T}(\mathbf{x}, \mathbf{x}_k) (1 - e^{-\sigma(\mathbf{x}_k) \Delta \mathbf{x}_k}) L_s(\mathbf{x}_k, \boldsymbol{\omega}_0) $$

5.  **Sum:** The final radiance is the sum of contributions:
    $$ L(\mathbf{x}, \boldsymbol{\omega}_0) \approx \sum_{k=1}^N C_k $$

**Formulas:**
The core approximation [S2, unofficial] is:
$$ L(\mathbf{x}, \boldsymbol{\omega}_0) \approx \sum_{k=1}^N \hat{T}(\mathbf{x}, \mathbf{x}_k) (1 - e^{-\sigma(\mathbf{x}_k) \Delta \mathbf{x}_k}) L_s(\mathbf{x}_k, \boldsymbol{\omega}_0) $$
where $\hat{T}(\mathbf{x}, \mathbf{x}_k) = \prod_{j=1}^{k-1} e^{-\sigma(\mathbf{x}_j) \Delta \mathbf{x}_j}$ and $\Delta \mathbf{x}_j = \|\mathbf{x}_{j+1} - \mathbf{x}_j\|$.

The sample locations and weights are chosen such that the integral is exact for polynomials of maximum degree $2N-1$, utilizing the $2N$ degrees of freedom in the positions and weights [S1, book].

**Parameters:**
*   $N$: Number of quadrature points. Typical values depend on the smoothness of the transfer function; higher $N$ yields higher accuracy for smooth functions.

**Edge Cases and Pitfalls:**
*   **Non-smooth Transfer Functions:** Gaussian integration performs poorly for non-smooth transfer functions selected to emphasize specific contour surfaces [S1, book]. In these cases, the integrand is not well-approximated by high-degree polynomials, and the method fails to capture sharp features accurately [S1, book].
*   **Precision:** The exponential terms in transmittance can suffer from underflow or overflow if density is very high or low; standard double-precision arithmetic is recommended.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_volume_rendering(ray_origin, ray_direction, t_near, t_far, density_field, emission_field, N):
    # 1. Get standard Legendre-Gauss nodes and weights for order N
    # nodes: array of size N in [-1, 1]
    # weights: array of size N
    nodes, weights = get_legendre_gauss_quadrature(N)
    
    # 2. Map nodes to ray interval [t_near, t_far]
    # Compute sample positions x_k
    sample_positions = []
    for xi in nodes:
        # Map xi from [-1, 1] to [t_near, t_far]
        t = 0.5 * (t_far - t_near) * (xi + 1) + t_near
        pos = ray_origin + t * ray_direction
        sample_positions.append(pos)
        
    # 3. Compute distances between adjacent quadrature points
    # delta_x[j] is distance between sample_positions[j] and sample_positions[j+1]
    delta_x = []
    for j in range(N - 1):
        dist = length(sample_positions[j+1] - sample_positions[j])
        delta_x.append(dist)
        
    # Note: The formulation in [S2, unofficial] uses delta_x_j for the segment 
    # associated with sample j. The transmittance product goes up to k-1.
    # We need to handle the boundary segments carefully. 
    # Standard Gaussian quadrature assumes integration over the interval.
    # The term (1 - exp(-sigma * delta_x)) implies a segment length.
    # For the first and last points, the segment length might be defined differently
    # or the weights absorb the interval scaling. 
    # Based on [S2, unofficial], delta_x_j is strictly distance between points j and j+1.
    # This implies the integration is split into N-1 segments? 
    # However, the sum is over k=1 to N. 
    # Let's assume the standard interpretation where the weights w_k scale the 
    # contribution, and the delta_x represents the local step size.
    # Actually, [S2, unofficial] defines delta_x_j = ||x_{j+1} - x_j||.
    # And the sum is over k. The transmittance T_hat uses product up to k-1.
    # This suggests the segments are defined by the nodes.
    
    # 4. Compute Radiance
    radiance = 0.0
    transmittance = 1.0
    
    for k in range(N):
        x_k = sample_positions[k]
        
        # Sample density and emission
        sigma = density_field.sample(x_k)
        L_s = emission_field.sample(x_k, ray_direction)
        
        # Compute transmittance to this point
        # T_hat(x, x_k) = product_{j=1}^{k-1} exp(-sigma(x_j) * delta_x_j)
        # Note: indices in formula are 1-based. k=1 means product is empty (1.0).
        # k=2 means product j=1.
        
        # Calculate current segment's scattering probability
        # For k=0 (first point), what is delta_x?
        # The formula uses delta_x_j for j < k.
        # The term (1 - exp(-sigma(x_k) * delta_x_k)) is ambiguous for the last point.
        # Usually, Gaussian quadrature weights w_k are used directly with the integrand.
        # The formula in [S2, unofficial] seems to adapt the discrete ray marching 
        # accumulation to quadrature points.
        
        # Let's follow [S2, unofficial] strictly:
        # delta_x_j is distance between j and j+1.
        # For the last point k=N-1, there is no j=k.
        # This implies the formula might assume N points define N-1 intervals, 
        # but the sum is N terms. This is a known ambiguity in adapting quadrature 
        # to discrete ray marching. 
        # However, [S1, book] states it approximates the integral as unequally weighted sum.
        # The standard Gaussian quadrature formula is integral(f) approx sum(w_k * f(x_k)).
        # The formula in [S2, unofficial] is a specific discretization.
        
        # Implementation of [S2, unofficial] formula:
        # We need delta_x for the current sample k.
        # If k < N-1, delta_x_k = distance(x_k, x_{k+1})
        # If k == N-1, delta_x_k is undefined by the pair definition.
        # Often, the last segment is handled by the weight or boundary condition.
        # Given the constraint to implement from text, and [S2, unofficial] 
        # defining delta_x_j = ||x_{j+1} - x_j||, we must assume the loop 
        # structure handles the indices.
        
        # Let's assume the standard Gaussian Quadrature application to the 
        # volume rendering integral where the integrand is the emitted radiance 
        * transmittance.
        # The formula in [S2, unofficial] is likely a specific variant for 
        # "neural volume rendering" or similar where discrete steps are used.
        
        # To be safe and implementable:
        # Use standard Gaussian Quadrature:
        # L = sum_{k=1}^N w_k * (integrand at x_k)
        # Integrand = T(x, x_k) * sigma(x_k) * L_s(x_k) ? 
        # No, the formula in [S2, unofficial] is explicit.
        
        # Re-reading [S2, unofficial]:
        # L approx sum_{k=1}^N T_hat(x, x_k) * (1 - exp(-sigma(x_k) * delta_x_k)) * L_s
        # This looks like a Riemann sum with specific points.
        
        # If we must use [S2, unofficial] exactly:
        # We need delta_x_k.
        # If k < N-1: delta_x_k = length(x_{k+1} - x_k)
        # If k == N-1: delta_x_k = length(x_k - x_{k-1})? Or half?
        # The source does not specify the boundary delta_x.
        # [G] Assume symmetric boundary handling or that N is large enough 
        # that boundary effects are negligible, or use the average of adjacent intervals.
        # However, a robust implementation for the last point often uses the 
        # distance to the previous point or a fixed step.
        # Let's use the distance to the previous point for the last segment 
        # if not defined, or assume the weights account for it.
        
        # Actually, looking at [S1, book], it mentions "unequally weighted sum".
        # The formula in [S2, unofficial] might be a specific instantiation.
        # Let's stick to the most general interpretation of Gaussian Integration 
        # for volume rendering as described in [S1, book]:
        # It uses nodes and weights to approximate the integral.
        # The integral is \int T(t) sigma(t) L_s(t) dt.
        # Approximation: sum_{k=1}^N w_k * T(t_k) * sigma(t_k) * L_s(t_k) * (t_far - t_near)/2 ?
        # The weights w_k in standard Gauss-Legendre are for the interval [-1, 1].
        # Scaling factor is (t_far - t_near)/2.
        
        # The formula in [S2, unofficial] is different. It uses (1 - exp(...)).
        # This is the discrete transmittance difference.
        # This formula is valid for piecewise constant density approximation 
        # between nodes.
        
        # Implementation of [S2, unofficial] formula with [G] for boundary:
        
        current_sigma = sigma
        current_Ls = L_s
        
        # Calculate delta_x for this sample
        if k < N - 1:
            dx = length(sample_positions[k+1] - sample_positions[k])
        else:
            # [G] For the last sample, use the distance to the previous sample
            # as a proxy for the segment length, or assume the weight handles it.
            # Given the product form for T, the last term's dx affects the 
            # scattering probability of the last segment.
            if k > 0:
                dx = length(sample_positions[k] - sample_positions[k-1])
            else:
                dx = 0.0 # Should not happen if N > 1
        
        scattering_prob = 1.0 - exp(-current_sigma * dx)
        
        contribution = transmittance * scattering_prob * current_Ls
        radiance += contribution
        
        # Update transmittance for the next sample
        # T_hat(x, x_{k+1}) = T_hat(x, x_k) * exp(-sigma(x_k) * delta_x_k)
        # Note: The formula for T_hat uses delta_x_j for j < k.
        # So after processing k, we update T for k+1 using dx of k.
        if k < N - 1:
            transmittance *= exp(-current_sigma * dx)
            
    return radiance
```

## Subtopics

### Runge-Kutta 4 (RK4) Integration
Uses higher-order polynomial approximations of the density and emission along the ray to achieve higher accuracy than simple quadrature, especially for varying density. Choose when density varies significantly within a segment and smoothness is not guaranteed. [Link](3158-runge-kutta-4-rk4-integration.md)

## Operating characteristics
*   **Time Complexity**: $O(N)$ per ray, where $N$ is the number of quadrature points. Evaluating density and emission at $N$ points dominates.
*   **Memory Complexity**: $O(1)$ auxiliary memory per ray (excluding the volume data structure).
*   **Performance**: Provides higher accuracy than equally spaced sampling (rectangle rule) for the same $N$ when the transfer function and density are smooth [S1, book].
*   **Strengths**: High accuracy for smooth integrands; fewer samples needed compared to uniform sampling for the same error tolerance in smooth regions.
*   **Limitations**: Poor performance for non-smooth transfer functions or sharp contours [S1, book]. Requires precomputed quadrature nodes and weights.
*   **When to use**: Smooth density and transfer functions.
*   **When not to use**: Sharp edges, discontinuities, or contour-based visualization where exact intersection is preferred [S1, book].

## Background
Gaussian integration is a classical numerical analysis technique applied to volume rendering to improve the accuracy of the discrete ray marching integral. It was discussed in the context of volume rendering by Williams et al. [S1, book], who noted its superiority over equally spaced sampling for smoothly varying transfer functions. The method leverages the properties of Legendre polynomials to place sample points optimally for polynomial approximation.

## Sources
[S1] Local and Global Illumination in the Volume Rendering Integral, https://avishek.net/assets/papers/nerfs/local-and-global-illumination-rendering-integral.pdf, fetched 2026-10-27 (book)
[S2] TöRF: Time-of-Flight Radiance Fields for Dynamic Scene View Synthesis, https://doi.org/10.48550/arxiv.2109.15271, fetched 2026-10-29 (unofficial)
