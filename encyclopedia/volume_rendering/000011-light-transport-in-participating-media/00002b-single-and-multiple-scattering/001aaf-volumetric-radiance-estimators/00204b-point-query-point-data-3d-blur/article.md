# Point Query × Point Data, 3D Blur

Also called: Conventional Radiance Estimation, Standard volumetric radiance estimate, Jensen and Christensen estimate

## Overview
This method estimates the in-scattered radiance at a specific query point $\mathbf{x}$ by searching for discrete photon points stored within the medium and averaging their energy over a three-dimensional spherical kernel of radius $r$ [S1, S2]. It differs from its sibling, the [Extended Balance Heuristic](5601-extended-balance-heuristic.md), by using a fixed-density spatial average rather than a balance heuristic that accounts for the relative probabilities of different path connections [S2]. Unlike the [Beam Query × Point Data, 3D Blur](4555-beam-query-beam-data-2d-blur.md) variant, which integrates transmittance along the camera ray to compute beam radiance directly, this method computes the local incident radiance at a point, requiring subsequent ray marching or integration to determine the final pixel color [S2].

## How it works
**Inputs and Outputs**
*   **Input:** A query point $\mathbf{x}$ in the medium, a view direction $\vec{\omega}$, a photon map containing discrete photon points, a spherical kernel radius $r$, and medium properties ($\sigma_t, \sigma_s$) [S2].
*   **Output:** The estimated in-scattered radiance $L_i(\mathbf{x} \rightarrow \vec{\omega})$ at the query point [S2].

**Data Structures**
*   **Photon Map:** A spatial index storing photon points [G]. Each photon point $p$ contains:
    *   Position $\mathbf{x}_p$ (3D vector) [S2].
    *   Power $\Phi_p$ (scalar, radiometric flux) [S2].
    *   Direction $\vec{\omega}_p$ (unit vector, incident direction from which the photon arrived) [S2].

**Algorithm Steps**
1.  **Query:** Identify all photon points $p$ within the spherical query region $R$ centered at $\mathbf{x}$ with radius $r$ [S2].
2.  **Iterate:** For each photon $p \in R$:
    a. Calculate the angle $\theta_p$ between the photon's incident direction $\vec{\omega}_p$ and the view direction $\vec{\omega}$ [S2].
    b. Evaluate the phase function $f(\theta_p)$ at the scattering angle [S2].
    c. Retrieve the photon power $\Phi_p$ [S2].
3.  **Sum:** Compute the weighted sum of photon powers multiplied by the phase function values [S2].
4.  **Normalize:** Divide the sum by the volume measure of the query region, $\mu_R(r^3)$ [S2]. For a sphere, $\mu_R(r^3) = \frac{4}{3}\pi r^3$ [S2].
5.  **Scale:** Multiply by the scattering coefficient $\sigma_s$ if the phase function $f$ is normalized to unity over the sphere, or divide by $\sigma_s$ if the standard definition of radiance density is used as per Equation 7 in [S2] [S2]. *Clarification: Equation 7 in [S2] divides by $\sigma_s \mu_R(r^3)$. However, Equation 1 in [S1] divides only by volume, attributing the term to the material shading function $s$. The sources differ on the inclusion of $\sigma_s$ in the denominator [S1, S2]. We follow [S2]'s explicit formula for the "standard" estimate.*

**Formulas**
The in-scattered radiance $L_i$ at query point $\mathbf{x}$ for direction $\vec{\omega}$ is:

$$ L_i(\mathbf{x} \rightarrow \vec{\omega}) \approx \frac{1}{\sigma_s \mu_R(r^3)} \sum_{p \in R} f(\theta_p) \Phi_p $$

Where:
*   $\sigma_s$: Scattering coefficient ($m^{-1}$) [S2].
*   $\mu_R(r^3)$: Volume measure of the query region $R$. For a sphere of radius $r$, $\mu_R(r^3) = \frac{4}{3}\pi r^3$ [S2].
*   $f(\theta_p)$: Phase function value for the scattering angle $\theta_p$ between the photon's incoming direction and the view direction [S2].
*   $\Phi_p$: Power (flux) of photon $p$ [S2].
*   $R$: The set of photons within distance $r$ of $\mathbf{x}$ [S2].

*Alternative Formulation (from [S1]):*
$$ L_{ms}(x,\omega_0) \approx \sum_{i=1}^{n} s(x',\omega_i,\omega_0) \frac{\Delta \Phi(x',\omega_i)}{\frac{4}{3}\pi r^3} $$
Where $s$ is the material shading function (which may include $\sigma_s$ and the phase function) and $n$ is the number of closest photons [S1].

**Parameters**
*   $r$: Kernel radius. Typical values depend on photon density; not given in sources [S1, S2].
*   $n$: Number of closest photons. In the standard 3D blur, all photons within radius $r$ are used, so $n$ is variable [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def phase_function(cos_theta):
    """Standard isotropic phase function evaluation."""
    return 1.0 / (4.0 * math.pi)

def estimate_point_query_point_data_3d_blur(query_point, view_direction, all_photons, radius, scattering_coefficient):
    """
    Estimates in-scattered radiance at a point using 3D blur of photon points.
    
    Args:
        query_point: 3D vector, position x
        view_direction: 3D unit vector, omega
        all_photons: List of dicts with keys 'pos', 'dir', 'power'
        radius: float, r
        scattering_coefficient: float, sigma_s
        
    Returns:
        float, estimated radiance L_i
    """
    sum_weighted_power = 0.0
    
    # 1. Query photons within spherical kernel
    for photon in all_photons:
        dx = photon['pos'][0] - query_point[0]
        dy = photon['pos'][1] - query_point[1]
        dz = photon['pos'][2] - query_point[2]
        dist_sq = dx*dx + dy*dy + dz*dz
        
        if dist_sq <= radius * radius:
            # 2. Calculate scattering angle theta
            cos_theta = (photon['dir'][0]*view_direction[0] + 
                         photon['dir'][1]*view_direction[1] + 
                         photon['dir'][2]*view_direction[2])
            
            # 3. Evaluate phase function
            phase_value = phase_function(cos_theta)
            
            # 4. Accumulate weighted power
            sum_weighted_power += phase_value * photon['power']
    
    # 5. Normalize by volume measure and scattering coefficient
    volume_measure = (4.0 / 3.0) * math.pi * (radius ** 3)
    
    if scattering_coefficient > 0:
        radiance = sum_weighted_power / (scattering_coefficient * volume_measure)
    else:
        radiance = 0.0
        
    return radiance
```

**Edge Cases and Pitfalls**
*   **Empty Query Region:** If no photons are found within radius $r$, the estimate is zero [G].
*   **Heterogeneous Media:** The standard point estimate assumes a local density [S2]. In heterogeneous media, the transmittance from the photon to the query point is not explicitly handled in the basic point estimate (Eq 7), unlike the beam estimates [S2].
*   **Spatially Adaptive Kernels:** The notation may suggest a restriction to spatially constant query regions, but this is for simplicity; spatially adaptive blurring kernels are possible [S2].
*   **Boundary Conditions:** The spherical kernel may extend outside the medium [G].

## Subtopics

### [Beam Query × Beam Data, 2D Blur](4555-beam-query-beam-data-2d-blur.md)
Uses continuous beam data and 2D blur to reduce bias along the beam direction. Choose when high accuracy in participating media with varying beam directions is required and computational cost allows for beam-based integration.

### [Extended Balance Heuristic](5601-extended-balance-heuristic.md)
Uses a balance heuristic to combine multiple estimators. Choose when unbiased results are critical and multiple path connections are available.

## Operating characteristics
*   **Time Complexity:** Not given in the sources [G].
*   **Memory Complexity:** Not given in the sources [G].
*   **Performance:** Fast per-sample evaluation compared to beam methods that require integration [S2].
*   **Strengths:** Simple to implement; well-understood; efficient for homogeneous media where the point estimate is sufficient for local incident radiance [S2].
*   **Limitations:** Biased estimator; requires careful tuning of kernel radius $r$; suffers from noise in sparse regions; does not directly compute beam radiance, requiring additional integration [S2].

## Background
The method was introduced by Henrik Wann Jensen and Hans H. Christensen in 1998 as the "Conventional Radiance Estimation" for volumetric photon mapping [S1, S2]. It is based on the principle of density estimation using spherical kernels [S2]. The method is a direct application of the radiance estimation framework to participating media, extending the point-to-point photon mapping used for surfaces to volumetric interactions [S2]. It is widely used in volumetric photon mapping implementations as the standard estimator for in-scattered radiance [S1, S2].

## Sources
[S1] Real-time volume rendering for three-dimensional fetal ultrasound using volumetric photon mapping., https://doi.org/10.1186/s42492-024-00177-4, fetched 2026-10-26
[S2] A comprehensive theory of volumetric radiance estimation using photon points and beams, https://doi.org/10.1145/1899404.1899409, fetched 2026-10-26
