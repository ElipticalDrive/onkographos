# Many-worlds transport

Also called: Exponential random volume transport, Random volume perspective

## Overview
Many-worlds transport is a method for preventing nonsensical scattering and shadowing between multiple potential surfaces along a ray by modeling the interaction with a tenuous volume [S1]. It solves the problem of self-shadowing and self-extinction in stochastic surface representations by diluting the density of the interaction volume by a factor of $\varepsilon$ [S1]. Unlike microflake theory, which uses directionally varying extinction and scattering coefficients from a random distribution of oriented particles, this method adopts a simpler particle model associating a single orientation $\boldsymbol{\beta}(\mathbf{x})$ with every point [S1].

## How it works
**Inputs:**
*   Ray segment defined by origin $\mathbf{0}$ and direction $\boldsymbol{\omega}$, parameterized by distance $t \in [0, s]$.
*   Occupancy function $\alpha(t)$, representing the point-wise discrete probability that position $t$ is inside the surface $S$ [S1].
*   Surface orientation field $\boldsymbol{\beta}(\mathbf{x})$, a unit vector associated with every point $\mathbf{x}$ [S1].
*   Extended outgoing radiance $L_o(\mathbf{x}, \boldsymbol{\omega})$, defined for points within the many-worlds representation [S1].
*   Dilution factor $\varepsilon$ (typically $\varepsilon \ll 1$) [S1].

**Outputs:**
*   The many-worlds radiance $L_1^\varepsilon(0)$ at the ray origin, approximating the incident radiance $L_i(0)$ while accounting for single interactions within the tenuous volume [S1].
*   The unit perturbation effect $L_1(0)$, obtained by removing the $\varepsilon$ scaling [S1].

**Data Structures:**
*   The method relies on scalar fields $\alpha(t)$ and vector fields $\boldsymbol{\beta}(t)$ sampled along the ray. No complex volumetric grids are required; the "volume" is implicit in the stochastic surface model [S1].

**Steps:**

1.  **Define Extinction Coefficient:**
    Calculate the extinction coefficient $\mu_t(\mathbf{x}, \boldsymbol{\omega})$ using the occupancy $\alpha(\mathbf{x})$ and the dot product of the ray direction and surface orientation. The model enforces nonreciprocal behavior: light interacting with a back-facing surface is masked [S1].
    $$
    \mu_t(\mathbf{x}, \boldsymbol{\omega}) = \begin{cases} 
    \varepsilon \alpha(\mathbf{x}), & \text{if } \boldsymbol{\omega} \cdot \boldsymbol{\beta}(\mathbf{x}) > 0, \\
    0, & \text{otherwise.}
    \end{cases}
    $$
    Here, $\alpha(\mathbf{x})$ acts as the number density scaled by cross-sectional area, and $\varepsilon$ ensures the volume is tenuous [S1].

2.  **Approximate Transmittance:**
    Because the volume is tenuous ($\varepsilon$ is small), the optical depth is small. The transmittance $T(t)$ is approximated by linearizing the exponential Beer-Lambert law $e^{-x} \approx 1 - x + O(x^2)$ [S1]:
    $$
    T(t) \approx 1 - \int_0^t \mu_t(t') dt'
    $$
    This approximation holds because higher-order terms involving $\mu_s \mu_t \sim \varepsilon^2$ are discarded [S1].

3.  **Compute Incident Radiance Approximation:**
    Substitute the linearized transmittance into the Radiative Transfer Equation (RTE). The incident radiance $L_i^\varepsilon(0)$ is approximated by integrating the source terms (scattering and emission) minus the out-scattering term, using the outgoing radiance $L_o(s)$ at the end of the segment for the out-scattering component [S1]:
    $$
    L_i^\varepsilon(0) \approx \int_0^s [\mu_s(t) L_s(t) + \mu_a(t) L_e(t) - \mu_t(t) L_o(s)] dt + L_o(s)
    $$
    Note: In the specific many-worlds derivation, the scattering and absorption terms are consolidated into the occupancy and orientation terms, leading to the simplified form below [S1].

4.  **Calculate Many-Worlds Radiance:**
    Substitute the specific definitions of $\mu_t$ and the relationship between scattering/absorption and the extended outgoing radiance into the integral. The resulting many-worlds radiance $L_1^\varepsilon(0)$ is:
    $$
    L_1^\varepsilon(0) = \varepsilon \int_0^s \alpha(t) [L_o(t) - L_o(s)] dt + L_o(s)
    $$
    The term $\alpha(t)$ is zero in the back-facing case ($\boldsymbol{\beta}(t) \cdot \boldsymbol{\omega} < 0$), effectively masking those parts of the volume [S1].

5.  **Compute Unit Perturbation Effect:**
    To obtain the derivative or unit perturbation effect, remove the $\varepsilon$ scaling. This yields the final transport equation for the many-worlds representation:
    $$
    L_1(0) = \int_0^s \alpha(t) [L_o(t) - L_o(s)] dt + L_o(s)
    $$
    This equation quantifies how the many-worlds representation modifies light transport along individual path segments [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def many_worlds_transport(ray_origin, ray_direction, segment_length, 
                          occupancy_field, orientation_field, 
                          outgoing_radiance_func, epsilon):
    """
    Computes the many-worlds radiance for a ray segment.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, unit direction of the ray.
        segment_length: float, length of the ray segment s.
        occupancy_field: function t -> float, returns alpha(t) for t in [0, s].
        orientation_field: function t -> 3D vector, returns beta(t) for t in [0, s].
        outgoing_radiance_func: function (point, direction) -> radiance, returns L_o.
        epsilon: float, dilution factor.
        
    Returns:
        L_1_eps: float/vector, the many-worlds radiance L_1^epsilon(0).
        L_1: float/vector, the unit perturbation effect L_1(0).
    """
    # Numerical integration placeholder
    # In practice, this integral is evaluated using quadrature (e.g., Gaussian)
    # along the ray segment [0, s].
    
    integral_sum = 0.0
    
    # Sample points along the ray
    num_samples = 100 # Implementation detail: sufficient for quadrature
    for t in linspace(0, segment_length, num_samples):
        # Get occupancy and orientation at current point
        alpha_t = occupancy_field(t)
        beta_t = orientation_field(t)
        
        # Check back-facing condition
        # Dot product of ray direction and surface orientation
        dot_product = dot(ray_direction, beta_t)
        
        if dot_product > 0:
            # Point is front-facing
            # Get outgoing radiance at current point t and at end of segment s
            # Note: outgoing_radiance_func takes position and direction
            pos_t = ray_origin + ray_direction * t
            pos_s = ray_origin + ray_direction * segment_length
            
            L_o_t = outgoing_radiance_func(pos_t, ray_direction)
            L_o_s = outgoing_radiance_func(pos_s, ray_direction)
            
            # Accumulate the integrand: alpha(t) * [L_o(t) - L_o(s)]
            integrand = alpha_t * (L_o_t - L_o_s)
            integral_sum += integrand
        else:
            # Back-facing: alpha is effectively zero for the interaction
            pass
            
    # Apply numerical integration weight (e.g., trapezoidal rule or simple average * length)
    # Assuming uniform sampling for simplicity in this generic pseudocode
    dt = segment_length / (num_samples - 1)
    integral_value = integral_sum * dt
    
    # Calculate L_1^epsilon(0)
    L_1_eps = epsilon * integral_value + outgoing_radiance_func(pos_s, ray_direction)
    
    # Calculate L_1(0) (unit perturbation)
    L_1 = integral_value + outgoing_radiance_func(pos_s, ray_direction)
    
    return L_1_eps, L_1
```

**Parameters:**
*   $\varepsilon$: A dilution factor for the tenuous volume density. It reduces scattering and extinction coefficients to ensure at most one interaction along a ray segment. Typical values are small ($\varepsilon \ll 1$) [S1].

**Edge Cases and Pitfalls:**
*   **Back-facing surfaces:** The branch condition $\boldsymbol{\omega} \cdot \boldsymbol{\beta}(\mathbf{x}) > 0$ encodes nonreciprocal behavior. Light interacting with a back-facing surface presents a nonsensical case; the extinction term masks those parts of the volume by setting $\mu_t = 0$ [S1].
*   **Multiple Interactions:** The method models only single interactions. For multi-segment paths, multiple interactions are not modeled as they would introduce second-order derivative terms beyond the scope of this approximation [S1].
*   **Precision:** The approximation $T(t) \approx 1 - \int \mu_t dt$ relies on $\varepsilon$ being small. If $\varepsilon$ is not sufficiently small, the discarded $O(\varepsilon^2)$ terms may introduce significant error [S1].

## Operating characteristics
*   **Time Complexity:** The method requires evaluating integrals along ray segments. The cost depends on the number of samples used for numerical integration of the occupancy and orientation fields. It is generally efficient as it avoids complex volumetric traversal or multiple scattering bounces.
*   **Memory Complexity:** Low. It does not require storing a full volumetric grid. It relies on implicit fields (occupancy and orientation) associated with the stochastic surface model [S1].
*   **Strengths:** Prevents nonsensical scattering and shadowing between multiple potential surfaces. Provides a physically plausible approximation of self-shadowing and self-extinction in stochastic surface representations. Simpler than microflake theory [S1].
*   **Limitations:** Approximates the RTE by discarding second-order terms. Does not model multiple scattering interactions within the volume. Accuracy depends on the smallness of $\varepsilon$ [S1].

## Background
Many-worlds transport was introduced in the context of inverse rendering and stochastic surface modeling. It builds upon the concept of modeling interactions with a tenuous volume to handle self-shadowing. The method is detailed in "Many-Worlds Inverse Rendering" [S1]. It relates to prior work on microflake theory [Jakob et al. 2010] but simplifies the particle model by associating a single orientation with every point rather than a distribution [S1].

## Sources
[S1] Many-Worlds Inverse Rendering, https://doi.org/10.1145/3767318, fetched 2026-10-26
