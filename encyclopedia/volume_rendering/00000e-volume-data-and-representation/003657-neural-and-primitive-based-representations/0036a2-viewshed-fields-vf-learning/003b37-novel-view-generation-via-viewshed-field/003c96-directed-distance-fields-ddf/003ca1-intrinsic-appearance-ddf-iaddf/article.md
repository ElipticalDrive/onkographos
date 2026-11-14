# Intrinsic Appearance DDF (IADDF)
Also called: DDF Path Tracing, Path Tracing with Intrinsic Appearance DDFs (IADDF), Recursive DDF Light Transport

## Overview
Intrinsic Appearance DDF (IADDF) is a rendering technique that augments a Probabilistic Directed Distance Field (PDDF) with material and lighting models to enable path tracing by recursively querying the DDF for surface intersections [S1]. It solves the problem of disentangling geometry, reflectance, and illumination in a view-consistent manner, allowing for the differentiable rendering of intrinsic images from any viewpoint [S1]. Unlike standard Neural Radiance Fields (NeRF) which act as entangled "clouds of particles," IADDF explicitly separates the scene into a geometric DDF and continuous Implicit Neural Representations (INRs) for appearance and lighting, leveraging the DDF's ability to model inter-surface light transport [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray origin $o$ and direction $\omega_o$ (camera ray) [S1].
*   **Output:** Pixel irradiance (radiance) $L$ [S1].

**Data Structures:**
The IADDF is a tuple $(G, M, L) = (d, \xi, f_B, \Psi, M_L, E_L)$ [S1]:
1.  **Geometry Field $G$:**
    *   Distance field $d: \mathbb{R}^3 \times S^2 \rightarrow \mathbb{R}^+$: Returns the distance to the next surface along a direction [S1].
    *   Visibility field $\xi: \mathbb{R}^3 \times S^2 \rightarrow [0, 1]$: Returns the probability that the surface at distance $d$ is visible (not occluded by another surface closer to the query point) [S1].
2.  **Material Appearance Model $M$:**
    *   BRDF $f_B: \mathbb{R}^3 \times S^2 \times S^2 \rightarrow \mathbb{R}^{|C|+}$: Reflectance distribution function where $C$ is the color channel set (e.g., RGB) [S1].
    *   Importance sampler $\Psi: \mathbb{R}^3 \times S^2 \rightarrow P[S^2]$: Probability distribution over outgoing directions $S^2$, conditioned on position and normal [S1].
3.  **Lighting Model $L$:**
    *   Emission field $M_L: \mathbb{R}^3 \times S^2 \rightarrow C$: Radiance emitted directly from a surface point [S1].
    *   Environment light $E_L: S^2 \rightarrow C$: Incoming radiance from distant sources (sky/background) [S1].

**Algorithm Steps:**
1.  **Initial Ray Cast:** For each pixel, cast a primary ray $r_0$ from the camera origin $o$ with direction $\omega_o$ [S1].
2.  **Intersection Query:** Query the DDF at the current ray origin $q$ and direction $\omega$ to obtain distance $d = d(q, \omega)$ and visibility $\xi = \xi(q, \omega)$ [S1].
3.  **Visibility Check:** If $\xi(q, \omega) \leq 0.5$ (or effectively 0, indicating no visible surface), the ray has missed the geometry [S1].
4.  **Surface Hit:** If $\xi > 0.5$, the ray hits a surface at point $p = q + d \cdot \omega$ [S1]. The normal $n$ is derived from the DDF gradient [S1].
5.  **Emission Accumulation:** Add the local emission $M_L(p, \omega)$ to the accumulated radiance [S1].
6.  **Bounce Sampling:** Sample a new outgoing direction $\omega'$ from the importance sampler $\Psi(p, n)$ [S1].
7.  **Recursive Transport:** Recursively compute the incoming radiance $L_i(p, -\omega')$ from direction $-\omega'$:
    $$ \mathcal{L}_i(q, \omega_i) = \begin{cases} \mathcal{L}(q - d(q, -\omega_i)\omega_i, -\omega_i) & \text{if } \xi(q, -\omega_i) > 0.5 \\ E_L(-\omega_i) & \text{otherwise} \end{cases} $$
    This step effectively queries the DDF for the *next* surface intersection along $-\omega'$ [S1].
8.  **BRDF Evaluation:** Weight the incoming radiance by the BRDF $f_B(p, \omega_i, \omega_o)$ and the importance sampling probability $\Psi(p, n)(\omega')$ [S1].
9.  **Termination:** Repeat steps 2-8 until the ray misses geometry ($\xi=0$) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_ray(origin, direction, depth, max_depth, ddf, material, lighting):
    # 1. Query DDF for distance and visibility
    dist = ddf.distance(origin, direction)
    vis = ddf.visibility(origin, direction)
    
    # 2. Check for miss (geometry not visible)
    if vis <= 0.5:
        return lighting.environment_light(direction)
    
    # 3. Calculate hit point
    hit_point = origin + dist * direction
    
    # 4. Get surface normal from DDF gradient [G]
    # Normal is derived from the distance field gradient: n = normalize(grad_p d)
    # Implementation uses standard finite difference approximation.
    normal = finite_difference_gradient(ddf.distance, hit_point, direction)
    
    # 5. Accumulate local emission
    radiance = lighting.emission_field(hit_point, direction)
    
    # 6. Termination condition
    if depth >= max_depth:
        return radiance
        
    # 7. Sample outgoing direction using importance sampler
    # Psi: R^3 x S^2 -> P[S^2]
    # Implementation uses standard Monte Carlo sampling from the distribution.
    outgoing_direction = sample_distribution(material.importance_sampler, hit_point, normal)
    
    # 8. Compute incoming radiance recursively
    # The incoming direction for the next bounce is -outgoing_direction
    incoming_radiance = trace_ray(
        hit_point, -outgoing_direction, depth + 1, max_depth, ddf, material, lighting
    )
    
    # 9. Evaluate BRDF and weight by PDF
    # fB: R^3 x S^2 x S^2 -> R^|C|+
    # Implementation evaluates the PDF of the importance sampler at the sampled direction.
    pdf = evaluate_pdf(material.importance_sampler, hit_point, normal, outgoing_direction)
    
    if pdf > 1e-6:
        brdf_value = material.brdf(hit_point, normal, -outgoing_direction, direction)
        radiance += brdf_value * incoming_radiance / pdf
        
    return radiance
```

**Parameters:**
*   $C$: Color channel set (e.g., RGB) [S1].
*   $P[S^2]$: Set of probability distributions on the unit sphere [S1].
*   Typical values for `max_depth` are not specified in the sources [G].

**Edge Cases and Pitfalls:**
*   **View-Inconsistency:** The DDF is 5D (position + direction) [S1]. Depth-rendering the same field from different viewpoints can produce incompatible geometries if the field is not properly constrained to be View Consistent (VC) [S1, Parent].
*   **Precision:** The visibility field $\xi$ uses a threshold (0.5) to determine intersection [S1]. Numerical instability near $\xi=0.5$ may cause flickering or missed bounces [G].
*   **Degenerate Input:** If the DDF returns $d=0$ or invalid normals, the ray may self-intersect or stall [G].

## Subtopics

### [Neural Depth Renderer as DDF](9287-neural-depth-renderer-as-ddf.md)
Treats a neural depth renderer as a conditional DDF, mapping camera parameters to depth and visibility images. It differs by explicitly structuring output as a DDF to inherit geometric properties like normal derivatives, whereas IADDF focuses on full light transport integration.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Naturally supports inter-surface reflections (global illumination) via recursion; disentangles intrinsic images (geometry, reflectance, illumination); view-consistent if the underlying DDF is VC [S1].
*   **Limitations:** Computationally expensive due to recursive DDF queries [G]; sensitive to view-inconsistency if the DDF is not properly regularized [S1].
*   **When to use:** When differentiable intrinsic image decomposition and physically accurate global illumination are required from a continuous representation [S1].

## Background
The concept of Intrinsic Images, separating depth, orientation, reflectance, and illumination, was introduced by Marr [88] (cited in [S1]). The IADDF method was proposed in "Probabilistic Directed Distance Fields for Ray-Based Shape Representations" (2025) [S1]. It extends the interpretation of DDFs as fields of all possible depth images to include inter-surface rays, modeling light transport as recursive DDF calls [S1].

## Sources
[S1] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
