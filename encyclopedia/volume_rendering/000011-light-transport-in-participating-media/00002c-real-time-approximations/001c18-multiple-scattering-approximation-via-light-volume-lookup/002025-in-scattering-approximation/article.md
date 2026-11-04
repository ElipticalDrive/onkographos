# In-Scattering Approximation

Also called: Local Approximation of In-scattering

## Overview
In-scattering approximation is a technique for estimating the light scattered into a viewing ray from surrounding volume elements, treating the scattered portion of direct global light as an emittance source to capture short-range scattering effects [S1, unofficial]. It sets itself apart from its siblings by offering a computationally cheap, non-iterative approximation that accounts for phase function and depth into the medium using remapped exponential decay, thereby avoiding the expensive integration required by exact multiple scattering methods [S2, unofficial]. Unlike [Beam Spreading Width Model](1193-beam-spreading-width-model.md) which models spatial spread, or [Precomputed Multiple Scattering Table](3324-precomputed-multiple-scattering-table.md) which relies on precomputed data, this method computes the effect analytically per sample based on accumulated density and light direction [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Accumulated density along the view ray ($\rho_{accum}$ or $\tau_{acc}$), the dot product between the view direction and the light source direction ($\mu_{sun}$ or $\cos\theta$), and a base dimensional profile parameter ($P$) representing the local cloud structure or distance from the cloud boundary [S2, unofficial].
*   **Outputs:** An in-scattering factor ($S$ or $I_e$) that modulates the direct light contribution to simulate the brightening of inner cloud volumes [S2, unofficial].

**Data Structures:**
No specific complex data structures are required beyond standard scalar variables for density, direction, and profile. The method operates on per-sample values during ray marching [S2, unofficial].

**Steps:**
1.  **Accumulate Density:** During the volume ray march, accumulate the optical depth or density ($\tau_{acc}$ or $\rho_{accum}$) from the camera to the current sample point [S2, unofficial].
2.  **Compute Light Direction Dot Product:** Calculate the dot product between the ray direction and the light source direction ($\mu_{sun}$) [S2, unofficial].
3.  **Remap Parameters:** Apply a remapping function to the sun dot product and the base dimensional profile to control the exponent's sensitivity. The remapping function maps an input range $[a, b]$ to an output range $[c, d]$ [S2, unofficial].
4.  **Calculate In-Scattering Factor:** Compute the in-scattering factor using the exponential decay formula involving the accumulated density and the remapped parameters [S2, unofficial].
5.  **Modulate Light:** Multiply the direct light contribution by this in-scattering factor to approximate the multiple scattering effect [S2, unofficial].

**Formulas:**
The core approximation for the in-scattering factor $S$ is given by [S2, unofficial]:

$$ S = \exp(-\rho_{accum} \cdot R(\mu_{sun}, P)) $$

Where $R(\mu_{sun}, P)$ is a remapped value derived from the sun dot product and the base dimensional profile:

$$ R(\mu_{sun}, P) = \text{Remap}(\mu_{sun}, 0.0, 0.9, 0.25, \text{Remap}(P, 1.0, 0.0, 0.05, 0.25)) $$

The $\text{Remap}(x, a, b, c, d)$ function linearly maps $x$ from the interval $[a, b]$ to $[c, d]$:

$$ \text{Remap}(x, a, b, c, d) = c + \frac{(x - a)(d - c)}{b - a} $$

Alternatively, for a more rigorous first-order scattering approximation treating scattered direct light as emittance, the intensity $I_e(s_0)$ is calculated as [S1, unofficial]:

$$ I_e(s_0) = \int_{s_0}^{R_\Omega} q(s) e^{-\int_{s_0}^s \tau(t) dt} d\tau $$

Where $q(s)$ is the direct global light distribution, $R_\Omega$ is the scattering sphere radius, and $\tau(t)$ is the optical depth [S1, unofficial]. This integral is discretized as [S1, unofficial]:

$$ I_s = \sum_{j=0}^J \Phi_j \sum_{m=0}^M I_d(s_m) \prod_{i=0}^{m-1} (1 - \alpha(s_i)) $$

Where $J$ is the number of ray directions, $M$ is the number of sampling steps, $\Phi_j$ is the normalized light contribution from direction $j$, $I_d(s_m)$ is the direct light at sample $m$, and $\alpha(s_i)$ is the opacity [S1, unofficial].

**Parameters:**
*   **Scattering Sphere Radius ($R_\Omega$):** Defines the range over which in-scattering is integrated. Short-range effects are dominant, so a limited radius is sufficient [S1, unofficial].
*   **Phase Function:** Weights incoming light from different directions. An isotropic phase function is often used for simplicity, weighting light equally in all directions [S1, unofficial].
*   **Ray Directions ($J$):** The number of directions discretizing the scattering sphere, found by subdividing a tetrahedron, icosahedron, or octahedron [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
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
    # Remap base dimensional profile from [1.0, 0.0] to [0.05, 0.25]
    # Note: The source uses Remap(P, 1.0, 0.0, 0.05, 0.25), implying P is in [0,1] but mapped inversely or the range is specified as [1.0, 0.0].
    # Assuming standard remap where input range is [in_min, in_max].
    # If P is in [0, 1], and we want to map 1.0->0.05 and 0.0->0.25, the range is [0.0, 1.0].
    # The source code snippet: Remap(_baseDimensionalProfile, 1.0, 0.0, 0.05, 0.25)
    # This implies the input range is [1.0, 0.0] (descending) or the arguments are (value, min, max, out_min, out_max).
    # Standard remap usually assumes min < max. Let's implement the logic as described:
    # Map P from [0.0, 1.0] to [0.25, 0.05] if P=0->0.25, P=1->0.05? 
    # Source says: Remap(P, 1.0, 0.0, 0.05, 0.25). 
    # If input is 1.0, output is 0.05. If input is 0.0, output is 0.25.
    
    p_mapped = remap(base_dimensional_profile, 0.0, 1.0, 0.25, 0.05)
    
    # Remap sun_dot from [0.0, 0.9] to [0.25, ...] ? 
    # Source: Remap(_sun_dot, 0.0, 0.9, 0.25, ...)
    # The second remap output is the input to the first remap's output range?
    # Actually, the formula is: Remap(sun_dot, 0.0, 0.9, 0.25, Remap(P, 1.0, 0.0, 0.05, 0.25))
    # This syntax is ambiguous. Let's look at the code:
    # exp(-_accumulatedDensity * Remap(_sun_dot, 0.0, 0.9, 0.25, Remap(_baseDimensionalProfile, 1.0, 0.0, 0.05, 0.25)))
    # It seems the 4th argument to the outer Remap is the result of the inner Remap.
    # So: inner_val = Remap(P, 1.0, 0.0, 0.05, 0.25)
    # outer_val = Remap(sun_dot, 0.0, 0.9, 0.25, inner_val)
    
    inner_val = remap(base_dimensional_profile, 0.0, 1.0, 0.25, 0.05) # Mapping 0->0.25, 1->0.05 based on source order 1.0->0.05, 0.0->0.25
    
    # Outer remap: sun_dot in [0.0, 0.9] maps to [0.25, inner_val]
    # If sun_dot is 0.0, result is 0.25. If sun_dot is 0.9, result is inner_val.
    exponent_factor = remap(sun_dot, 0.0, 0.9, 0.25, inner_val)
    
    return exp(-accumulated_density * exponent_factor)
```

**Edge Cases and Pitfalls:**
*   **Precision:** The exponential function can underflow to zero for large accumulated densities, which is physically correct (opaque regions) but may cause numerical issues if not handled [S2, unofficial].
*   **Boundaries:** The approximation assumes a certain depth into the cloud. At the very edge, the profile parameter $P$ should be handled carefully to avoid discontinuities [S2, unofficial].
*   **Degenerate Input:** If the sun dot product is outside the assumed range $[0.0, 0.9]$, the remapping may produce unexpected results. Clamping or extending the remapping range may be necessary [S2, unofficial].

## Subtopics

### [Depth-Aware Scatter Estimator](2585-depth-aware-scatter-estimator.md)
A neural network-based module that estimates primary transmission and depth-dependent scatter contributions, primarily for X-ray imaging. It uses a learned convolutional architecture to refine scatter estimation based on depth context, unlike the analytical remapped exponential decay of the parent method.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, as it involves only basic arithmetic operations and a single exponential evaluation [S2, unofficial].
*   **Memory Complexity:** $O(1)$, requiring no additional storage beyond the current sample's density and direction [S2, unofficial].
*   **Measured Performance:** The method is described as "very cheap" and "not too bad" since direct lighting is significantly stronger, making the approximation effective for real-time rendering [S2, unofficial].
*   **Strengths:** Computationally efficient, easy to implement, and provides a visually plausible brightening of inner cloud volumes [S2, unofficial].
*   **Limitations:** Not perfectly accurate; cannot produce "Dark Edge" effects without additional techniques like Beer's-Powder attenuation [S3, unofficial]. It is an approximation that may fail in scenes with very high multiple scattering contributions relative to direct light [S2, unofficial].
*   **When to Use:** Real-time volume rendering where performance is critical and direct lighting dominates, such as in cloud rendering [S2, unofficial].
*   **When Not to Use:** When high physical accuracy is required, or when specific effects like dark edges are crucial and cannot be achieved by combining with other techniques [S3, unofficial].

## Background
The concept of approximating in-scattering by treating scattered direct light as emittance originates from early volume rendering research, such as the work by Max [Max95] and later refined in the context of Local Approximation of In-scattering (LAO) [HLY07] [S1, unofficial]. The specific remapped exponential decay formulation for cloud rendering was popularized in real-time graphics, as seen in Jack Tollenaar's volumetric cloud rendering tutorials [S2, unofficial]. The technique is widely used in game engines and real-time renderers to simulate multiple scattering effects efficiently [S3, unofficial].

## Sources
[S1] Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration, https://doi.org/10.2312/vg/vg-pbg08/105-112, fetched 2026-10-26 (unofficial)
[S2] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
[S3] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
