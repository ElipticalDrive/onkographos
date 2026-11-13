# Point Query × Point Data, 3D Blur

Also called: Conventional Radiance Estimation, Standard volumetric radiance estimate, Jensen and Christensen estimate

## Overview
Point Query × Point Data, 3D Blur estimates the in-scattered radiance at a specific query point by aggregating the energy of discrete photon points stored within a surrounding three-dimensional volume [S1, S2]. It solves the problem of approximating the integral of incoming radiance from all directions by converting a continuous density estimation problem into a discrete summation over a spherical kernel [S2]. It differs from its sibling, the [Extended Balance Heuristic](5601-extended-balance-heuristic.md), by relying on a fixed spherical kernel and standard density estimation rather than balance heuristics to combine multiple estimators [S2]. It differs from its child, [Beam Query × Beam Data, 2D Blur](4555-beam-query-beam-data-2d-blur.md), by using discrete point data and blurring in 3D, which introduces bias along the light transport direction that the 2D blur variant avoids [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A query position $\mathbf{x}$, a view direction $\vec{\omega}$, a photon map containing discrete photon points, a spherical kernel radius $r$, and the scattering coefficient $\sigma_s$ of the medium.
*   **Output:** The estimated in-scattered radiance $L_i(\mathbf{x} \rightarrow \vec{\omega})$ at the query point.

**Data Structures**
The photon map stores discrete photon points. Each photon point $p$ contains:
*   $\mathbf{x}_p$: Position in world space.
*   $\Phi_p$: Power (flux) of the photon.
*   $\vec{\omega}_p$: Direction of the photon (used to compute the phase function angle).

**Algorithm**
1.  **Query Region Definition:** Define a three-dimensional query region $R$ centered at the query point $\mathbf{x}$. The standard region is a sphere of radius $r$. The measure of this region is $\mu_R(r^3) = \frac{4}{3}\pi r^3$ [S2].
2.  **Photon Search:** Identify all photons $p$ in the photon map such that their position $\mathbf{x}_p$ lies within the region $R$ [S2].
3.  **Phase Function Evaluation:** For each photon $p$ found in $R$, compute the phase function value $f(\theta_p)$, where $\theta_p$ is the scattering angle between the incoming photon direction and the view direction $\vec{\omega}$ at $\mathbf{x}$ [S2].
4.  **Summation:** Sum the product of the phase function and the photon power for all photons in $R$ [S2].
5.  **Normalization:** Divide the sum by the scattering coefficient $\sigma_s$ and the volume measure $\mu_R(r^3)$ [S2].

**Formula**
The in-scattered radiance $L_i(\mathbf{x} \rightarrow \vec{\omega})$ is estimated as [S2]:

$$
L_i(\mathbf{x} \rightarrow \vec{\omega}) \approx \frac{1}{\sigma_s \mu_R(r^3)} \sum_{p \in R} f(\theta_p) \Phi_p
$$

Where:
*   $\sigma_s$: Scattering coefficient of the medium ($m^{-1}$) [S2].
*   $\mu_R(r^3)$: Volume measure of the query region. For a sphere of radius $r$, $\mu_R(r^3) = \frac{4}{3}\pi r^3$ [S2].
*   $R$: The set of photon indices $p$ such that $\|\mathbf{x}_p - \mathbf{x}\| \le r$ [S2].
*   $f(\theta_p)$: Phase function value for the scattering angle $\theta_p$ [S2].
*   $\Phi_p$: Power of photon $p$ [S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_query_point_data_3d_blur(query_point, view_direction, photon_map, radius, scattering_coefficient):
    """
    Estimates in-scattered radiance at query_point using Point Query x Point Data, 3D Blur.
    
    Args:
        query_point (Vector3): The position x where radiance is estimated.
        view_direction (Vector3): The view direction omega.
        photon_map (List[Photon]): List of photon points.
        radius (float): The spherical kernel radius r.
        scattering_coefficient (float): The scattering coefficient sigma_s.
        
    Returns:
        float: Estimated in-scattered radiance.
    """
    # 1. Define the query region measure (volume of sphere)
    # mu_R(r^3) = 4/3 * pi * r^3
    volume_measure = (4.0 / 3.0) * math.pi * (radius ** 3)
    
    # 2. Initialize sum
    total_weighted_power = 0.0
    
    # 3. Search for photons within the spherical kernel
    for photon in photon_map:
        # Check if photon is within radius r of query_point
        distance_sq = vector_length_sq(photon.position - query_point)
        if distance_sq <= radius * radius:
            # 4. Compute phase function f(theta_p)
            # theta_p is the angle between the photon's incoming direction 
            # and the view direction at the query point.
            # Note: Photon direction in map is typically the direction 
            # the photon traveled. The scattering angle is between 
            # -photon.direction (incoming) and view_direction (outgoing).
            incoming_direction = normalize(-photon.direction)
            cos_theta = dot(incoming_direction, view_direction)
            # Clamp to [-1, 1] to handle floating point errors
            cos_theta = clamp(cos_theta, -1.0, 1.0)
            phase_value = phase_function(cos_theta)
            
            # 5. Accumulate f(theta_p) * Phi_p
            total_weighted_power += phase_value * photon.power
            
    # 6. Normalize by sigma_s and volume measure
    if scattering_coefficient == 0 or volume_measure == 0:
        return 0.0
        
    radiance = total_weighted_power / (scattering_coefficient * volume_measure)
    
    return radiance
```

**Parameters**
*   $r$: The radius of the spherical kernel. Typical values depend on photon density; it must be large enough to contain sufficient photons to reduce variance but small enough to preserve spatial detail [S1, S2].
*   $\sigma_s$: The scattering coefficient of the medium [S2].

**Edge Cases and Pitfalls**
*   **Empty Kernel:** If no photons fall within the radius $r$, the estimate is zero. This results in dark regions in the volume [S2].
*   **Heterogeneous Media:** The standard formula assumes a constant $\sigma_s$ within the kernel. In heterogeneous media, $\sigma_s$ varies, and the simple division by $\sigma_s$ is an approximation [S2].
*   **Boundary Conditions:** The spherical kernel may extend outside the participating medium. Photons outside the medium should not contribute, or the transmittance/interaction logic must account for the boundary [S2].
*   **Bias:** The use of a fixed spherical kernel introduces bias, particularly if the photon distribution is non-uniform [S2].

## Subtopics

### [Beam Query × Beam Data, 2D Blur](4555-beam-query-beam-data-2d-blur.md)
Uses continuous beam data and 2D blur to reduce bias along the beam direction compared to the 3D blur of point data. Choose this when high accuracy in anisotropic scattering is required and computational cost allows for beam-based structures.

### [Point Query × Beam Data, 3D Blur](4556-point-query-beam-data-3d-blur.md)
Uses discrete point query but beam data with 3D blur. It integrates transmittance along the beam within the kernel. Choose this when photon beams are available and a 3D blur is desired for the query point.

### [Beam Query × Point Data, 3D Blur](4557-beam-query-point-data-3d-blur.md)
Uses beam query (integrating along the eye ray) and point data with 3D blur. It computes accumulated radiance along the camera ray directly. Choose this when estimating beam radiance for volume rendering along eye rays.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per query point if a brute-force search is used, where $N$ is the number of photons. With a spatial acceleration structure (e.g., k-d tree), the search complexity approaches $O(\log N + k)$, where $k$ is the number of photons in the kernel [S2].
*   **Memory Complexity:** $O(N)$ to store the photon map [S2].
*   **Strengths:** Simple to implement; widely used as a baseline for volumetric photon mapping [S1, S2].
*   **Limitations:** Introduces bias due to the fixed kernel size and shape; sensitive to photon density; requires careful tuning of the radius $r$ [S1, S2].
*   **When to use:** When a simple, robust estimate is needed and photon density is sufficient to fill the kernel [S2].
*   **When not to use:** When high accuracy in anisotropic scattering is required (use Beam variants) or when photon density is very low (use balance heuristics) [S2].

## Background
The method was introduced by Jensen and Christensen in 1998 as the standard volumetric radiance estimate for photon mapping in participating media [S2]. It is based on the principle of density estimation, where the radiance is approximated by the average power of photons within a local volume [S2]. It is a fundamental component of volumetric photon mapping algorithms [S1].

## Sources
[S1] Real-time volume rendering for three-dimensional fetal ultrasound using volumetric photon mapping., https://doi.org/10.1186/s42492-024-00177-4, fetched 2026-10-26
[S2] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
