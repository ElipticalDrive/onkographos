# In-Scattering Approximation

Also called: Local Approximation of In-scattering

## Overview
In-scattering approximation estimates light scattered into a viewing ray from surrounding volume elements by treating the scattered portion of direct global light as an emittance source to capture short-range scattering effects [S1]. It sets itself apart from its siblings by offering a computationally cheap, non-iterative approximation that accounts for phase function and depth into the medium using remapped exponential decay, thereby avoiding the expensive integration required by exact multiple scattering methods [S2, unofficial]. Unlike [Beam Spreading Width Model](1193-beam-spreading-width-model.md) which models spatial spread, or [Precomputed Multiple Scattering Table](3324-precomputed-multiple-scattering-table.md) which relies on precomputed data, this method computes the effect analytically per sample based on accumulated density and light direction [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Accumulated density along the view ray ($\rho_{accum}$), the dot product between the view direction and the light source direction ($\mu_{sun}$), and a base dimensional profile parameter ($P$) representing local cloud structure or distance from the boundary [S2, unofficial].
*   **Outputs:** An in-scattering factor ($S$) that modulates the direct light contribution to simulate the brightening of inner cloud volumes [S2, unofficial].

**Data Structures:**
No specific complex data structures are required beyond standard scalar variables for density, direction, and profile; the method operates on per-sample values during ray marching [S2, unofficial].

**Steps:**
1.  **Accumulate Density:** During the volume ray march, accumulate the optical depth or density ($\tau_{acc}$ or $\rho_{accum}$) from the camera to the current sample point [S2, unofficial].
2.  **Compute Light Direction Dot Product:** Calculate the dot product between the ray direction and the light source direction ($\mu_{sun}$) [S2, unofficial].
3.  **Remap Parameters:** Apply a remapping function to the sun dot product and the base dimensional profile to control the exponent's sensitivity [S2, unofficial].
4.  **Calculate In-Scattering Factor:** Compute the in-scattering factor using the exponential decay formula involving the accumulated density and the remapped parameters [S2, unofficial].
5.  **Modulate Light:** Multiply the direct light contribution by this in-scattering factor to approximate the multiple scattering effect [S2, unofficial].

**Formulas:**
The core cheap approximation for the in-scattering factor $S$ is given by [S2, unofficial]:

$$ S = \exp(-\rho_{accum} \cdot \text{Remap}(\mu_{sun}, 0.0, 0.9, 0.25, \text{Remap}(P, 1.0, 0.0, 0.05, 0.25))) $$

Where $\rho_{accum}$ is accumulated density, $\mu_{sun}$ is the sun dot product, $P$ is the base dimensional profile, and $\text{Remap}(x, a, b, c, d)$ linearly maps $x$ from the interval $[a, b]$ to $[c, d]$ [S2, unofficial].

Alternatively, a rigorous first-order scattering approximation treats scattered direct light as emittance, calculating intensity $I_e(s_0)$ as [S1]:

$$ I_e(s_0) = \int_{s_0}^{R_\Omega} q(s) e^{-\int_{s_0}^s \tau(t) dt} d\tau $$

Where $q(s)$ is the direct global light distribution, $R_\Omega$ is the scattering sphere radius, and $\tau(t)$ is the optical depth [S1]. This integral is discretized as [S1]:

$$ I_s = \sum_{j=0}^J \Phi_j \sum_{m=0}^M I_d(s_m) \prod_{i=0}^{m-1} (1 - \alpha(s_i)) $$

Where $J$ is the number of ray directions, $M$ is the number of sampling steps, $\Phi_j$ is the normalized light contribution from direction $j$, $I_d(s_m)$ is the direct light at sample $m$, and $\alpha(s_i)$ is the opacity [S1].

**Parameters:**
*   **Scattering Sphere Radius ($R_\Omega$):** Defines the range over which in-scattering is integrated; short-range effects are dominant [S1].
*   **Phase Function:** Weights incoming light from different directions; an isotropic phase function is often used, weighting light equally in all directions [S1].
*   **Ray Directions ($J$):** The number of directions discretizing the scattering sphere, found by subdividing a tetrahedron, icosahedron, or octahedron [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def remap(value, in_min, in_max, out_min, out_max):
    """Linearly remaps value from [in_min, in_max] to [out_min, out_max]."""
    if in_max == in_min:
        return out_min
    t = (value - in_min) / (in_max - in_min)
    return out_min + t * (out_max - out_min)

def in_scattering_approx(base_dimensional_profile, sun_dot, accumulated_density):
    """
    Computes the in-scattering factor using remapped exponential decay.
    
    Args:
        base_dimensional_profile (float): Profile parameter representing cloud structure/distance.
        sun_dot (float): Dot product between view direction and light direction.
        accumulated_density (float): Accumulated density/optical depth from camera to current sample.
        
    Returns:
        float: In-scattering factor.
    """
    inner_val = remap(base_dimensional_profile, 1.0, 0.0, 0.05, 0.25)
    exponent_factor = remap(sun_dot, 0.0, 0.9, 0.25, inner_val)
    return math.exp(-accumulated_density * exponent_factor)
```

**Edge Cases and Pitfalls:**
*   **Precision:** The exponential function underflows to zero for large accumulated densities, which is physically correct for opaque regions [S2, unofficial].
*   **Boundaries:** The formula assumes $\mu_{sun} \in [0.0, 0.9]$ and $P \in [0.0, 1.0]$ per the remapping ranges; values outside these ranges will extrapolate linearly [S2, unofficial].

## Subtopics

### [Depth-Aware Scatter Estimator](2585-depth-aware-scatter-estimator.md)
A neural network-based module that estimates primary transmission and depth-dependent scatter contributions, primarily for X-ray imaging. It uses a learned convolutional architecture to refine scatter estimation based on depth context, unlike the analytical remapped exponential decay of the parent method.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, as it involves only basic arithmetic operations and a single exponential evaluation [S2, unofficial].
*   **Memory Complexity:** $O(1)$, requiring no additional storage beyond the current sample's density and direction [S2, unofficial].
*   **Measured Performance:** The method is described as "very cheap" and effective because direct lighting is significantly stronger [S2, unofficial].
*   **Strengths:** Computationally efficient, easy to implement, and provides a visually plausible brightening of inner cloud volumes [S2, unofficial].
*   **Limitations:** Not perfectly accurate; cannot produce "Dark Edge" effects without additional techniques like Beer's-Powder attenuation [S3, unofficial].
*   **When to Use:** Real-time volume rendering where performance is critical and direct lighting dominates, such as in cloud rendering [S2, unofficial].
*   **When Not to Use:** When high physical accuracy is required, or when specific effects like dark edges are crucial and cannot be achieved by combining with other techniques [S3, unofficial].

## Background
The concept of approximating in-scattering by treating scattered direct light as emittance originates from early volume rendering research, such as the work by Max [Max95] and later refined in the context of Local Approximation of In-scattering (LAO) [HLY07] [S1]. The specific remapped exponential decay formulation for cloud rendering was popularized in real-time graphics, as seen in Jack Tollenaar's volumetric cloud rendering tutorials [S2, unofficial]. The technique is widely used in game engines and real-time renderers to simulate multiple scattering effects efficiently [S3, unofficial].

## Sources
[S1] Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration, https://doi.org/10.2312/vg/vg-pbg08/105-112, fetched 2026-10-26
[S2] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
[S3] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
