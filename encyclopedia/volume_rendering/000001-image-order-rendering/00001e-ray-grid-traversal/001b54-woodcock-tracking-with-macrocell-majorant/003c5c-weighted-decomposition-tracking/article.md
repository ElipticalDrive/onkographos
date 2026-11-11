# Weighted Decomposition Tracking

Also called: Analog Decomposition Tracking, Decomposition Tracking

## Overview
Weighted Decomposition Tracking accelerates free-path sampling in heterogeneous volumes by decomposing the medium into a homogeneous control component and a heterogeneous residual component [S1]. It reduces the number of expensive spatially-varying extinction lookups required by standard delta tracking while producing identical noise patterns and light paths [S1]. Unlike the analog variant, the weighted formulation permits non-bounding control extinctions and free-path-sampling coefficients, enabling efficient chromatic (spectral) collision handling without requiring non-negative residual extinction [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Current position $\mathbf{x}$, direction $\omega$, path throughput weight $w$ (initialized to 1), constant control extinction coefficients $\mu_a^c, \mu_s^c$, constant free-path-sampling coefficient $\bar{\mu}$, and spatially varying functions for residual absorption $\mu_a^r(\mathbf{x})$, scattering $\mu_s^r(\mathbf{x})$, and null-collision $\mu_n(\mathbf{x})$ [S1].
*   **Output:** Updated position, direction, and throughput weight after one collision event, or emitted radiance if absorption occurs [S1].

**Data Structures:**
*   **Control Component:** Defined by constant scalar values $\mu_a^c$ and $\mu_s^c$ [S1].
*   **Residual Component:** Defined by spatially varying functions $\mu_a^r(\mathbf{x})$ and $\mu_s^r(\mathbf{x})$ [S1].
*   **Majorant:** $\bar{\mu}$ is a constant upper bound for the total extinction $\mu_t(\mathbf{x})$ across the volume or spectral band [S1]. In the weighted version, $\bar{\mu}$ does not need to be a minorant of $\mu_t(\mathbf{x})$ at every point, but must satisfy $\bar{\mu} > \mu_t^c$ [S1].
*   **Null-Collision Coefficient:** $\mu_n(\mathbf{x}) = \bar{\mu} - \mu_t^c - \mu_t^r(\mathbf{x})$ [S1].

**Parameters:**
*   $\bar{\mu}$: Free-path-sampling coefficient, typically chosen as the maximum total extinction in the scene [S1].
*   $\mu_t^c$: Control total extinction, computed as $\mu_t^c = \mu_a^c + \mu_s^c$ [S1].
*   Octree Depth: Typically 8–10 to balance lookup reduction and traversal overhead [S1].

**Formulas:**
The method samples one of five events: control absorption, control scattering, residual absorption, residual scattering, or null collision [S1]. Probabilities are defined as [S1]:
$$P_a^c = \frac{\mu_a^c}{\bar{\mu}}$$
$$P_s^c = \frac{\mu_s^c}{\bar{\mu}}$$
$$P_a^r(\mathbf{x}) = \left(1 - \frac{\mu_t^c}{\bar{\mu}}\right) \frac{|\mu_a^r(\mathbf{x})|}{|\mu_a^r(\mathbf{x})| + |\mu_s^r(\mathbf{x})| + |\mu_n(\mathbf{x})|}$$
$$P_s^r(\mathbf{x}) = \left(1 - \frac{\mu_t^c}{\bar{\mu}}\right) \frac{|\mu_s^r(\mathbf{x})|}{|\mu_a^r(\mathbf{x})| + |\mu_s^r(\mathbf{x})| + |\mu_n(\mathbf{x})|}$$
$$P_n(\mathbf{x}) = \left(1 - \frac{\mu_t^c}{\bar{\mu}}\right) \frac{|\mu_n(\mathbf{x})|}{|\mu_a^r(\mathbf{x})| + |\mu_s^r(\mathbf{x})| + |\mu_n(\mathbf{x})|}$$
The denominator sums the absolute values of the residual and null coefficients [S1].

**Algorithm Steps:**
The tracker loops indefinitely until a non-null collision occurs or emission is sampled [S1].
1. **Sample Free Path:** Draw a uniform random number $\zeta \in [0, 1)$ and compute $t = -\ln(1-\zeta)/\bar{\mu}$ [S1].
2. **Advance Position:** Update position $\mathbf{x} \leftarrow \mathbf{x} + t\omega$ [S1].
3. **Evaluate Probabilities:** Compute $P_a^c, P_s^c, P_a^r(\mathbf{x}), P_s^r(\mathbf{x}), P_n(\mathbf{x})$ using the current position $\mathbf{x}$ and constants [S1].
4. **Sample Event:** Draw a uniform random number $\xi \in [0, 1)$ and accumulate probabilities to determine the event [S1].
   * If $\xi < P_a^c$: Control absorption. The local weight factor is $\mu_a^c/(\bar{\mu}P_a^c) = 1$, so $w$ remains unchanged. Return emitted radiance $L_e(\mathbf{x}, \omega)$ scaled by $w$ [S1].
   * Else if $\xi < P_a^c + P_s^c$: Control scattering. The local weight factor is $\mu_s^c/(\bar{\mu}P_s^c) = 1$. Sample a new direction $\omega$ from the phase function $p(\omega)$ [G]. Break the loop [S1].
   * Else if $\xi < P_a^c + P_s^c + P_a^r(\mathbf{x})$: Residual absorption. Update $w \leftarrow w \cdot \mu_a^r(\mathbf{x})/(\bar{\mu}P_a^r(\mathbf{x}))$. Return emitted radiance scaled by $w$ [S1].
   * Else if $\xi < P_a^c + P_s^c + P_a^r(\mathbf{x}) + P_s^r(\mathbf{x})$: Residual scattering. Update $w \leftarrow w \cdot \mu_s^r(\mathbf{x})/(\bar{\mu}P_s^r(\mathbf{x}))$. Sample a new direction $\omega$ from $p(\omega)$ [G]. Break the loop [S1].
   * Else: Null collision. Update $w \leftarrow w \cdot \mu_n(\mathbf{x})/(\bar{\mu}P_n(\mathbf{x}))$. Continue the loop [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def WeightedDecompositionTracking(x, omega, w, mu_a_c, mu_s_c, mu_bar, 
                                  get_mu_a_r, get_mu_s_r, get_mu_n):
    """
    Performs one step of weighted decomposition tracking.
    
    Args:
        x: Current position (vec3)
        omega: Current direction (vec3)
        w: Current path throughput weight (float)
        mu_a_c: Control absorption coefficient (float, constant)
        mu_s_c: Control scattering coefficient (float, constant)
        mu_bar: Free-path-sampling coefficient (float, constant)
        get_mu_a_r: Function returning residual absorption at x (float)
        get_mu_s_r: Function returning residual scattering at x (float)
        get_mu_n: Function returning null-collision coefficient at x (float)
        
    Returns:
        Tuple (new_x, new_omega, new_w, event_type)
        event_type: 'ABSORB', 'SCATTER', or 'CONTINUE'
    """
    mu_t_c = mu_a_c + mu_s_c
    
    # 1. Sample free path
    zeta = random_uniform()
    t = -math.log(1.0 - zeta) / mu_bar
    
    # 2. Advance position
    x = x + t * omega
    
    # 3. Evaluate coefficients and probabilities
    mu_a_r = get_mu_a_r(x)
    mu_s_r = get_mu_s_r(x)
    mu_n = get_mu_n(x)
    
    # Denominator for residual/null probabilities
    denom = abs(mu_a_r) + abs(mu_s_r) + abs(mu_n)
    
    # Marginal probability of NOT sampling control
    p_not_c = 1.0 - (mu_t_c / mu_bar)
    
    # Probabilities
    P_a_c = mu_a_c / mu_bar
    P_s_c = mu_s_c / mu_bar
    
    if denom > 0:
        P_a_r = p_not_c * (abs(mu_a_r) / denom)
        P_s_r = p_not_c * (abs(mu_s_r) / denom)
        P_n = p_not_c * (abs(mu_n) / denom)
    else:
        # Degenerate case: no residual/null coefficients.
        P_a_r = 0.0
        P_s_r = 0.0
        P_n = 0.0

    # 4. Sample event
    xi = random_uniform()
    
    if xi < P_a_c:
        # Control Absorption
        return (x, omega, w, 'ABSORB')
        
    elif xi < P_a_c + P_s_c:
        # Control Scattering
        new_omega = sample_phase_function(omega)
        return (x, new_omega, w, 'SCATTER')
        
    elif xi < P_a_c + P_s_c + P_a_r:
        # Residual Absorption
        weight_factor = mu_a_r / (mu_bar * P_a_r)
        new_w = w * weight_factor
        return (x, omega, new_w, 'ABSORB')
        
    elif xi < P_a_c + P_s_c + P_a_r + P_s_r:
        # Residual Scattering
        weight_factor = mu_s_r / (mu_bar * P_s_r)
        new_w = w * weight_factor
        new_omega = sample_phase_function(omega)
        return (x, new_omega, new_w, 'SCATTER')
        
    else:
        # Null Collision
        weight_factor = mu_n / (mu_bar * P_n)
        new_w = w * weight_factor
        return (x, omega, new_w, 'CONTINUE')

def random_uniform() -> float:
    """Returns a uniform random float in [0, 1). [G]"""
    import random
    return random.random()

def sample_phase_function(omega: tuple) -> tuple:
    """Samples a new direction from the phase function p(omega). [G]"""
    # Standard phase function sampling implementation [G]
    return (0.0, 0.0, 1.0)
```

**Edge Cases and Pitfalls:**
*   **Non-Bounding Control:** If $\mu_t^c > \mu_t(\mathbf{x})$ at some point, $\mu_n(\mathbf{x})$ becomes negative. The weighted formulation handles this via absolute values in probabilities and signed weights, but a loose $\bar{\mu}$ increases the variance of the path throughput rapidly [S1].
*   **Zero Denominator:** If the residual and null coefficients are all zero (e.g., homogeneous medium where $\mu_t^c = \mu_t(\mathbf{x})$ and $\bar{\mu} = \mu_t^c$), the denominator is zero. In this case, $P_a^r, P_s^r, P_n$ are zero, and only control events are possible [S1].
*   **Precision:** Floating-point precision issues may arise if $\mu_t^c$ is very close to $\bar{\mu}$, making $P_a^c$ and $P_s^c$ dominate and residual probabilities negligible [S1].

## Subtopics

### Vectorized Decomposition Spectral Tracking
A variant designed for chromatic (spectral) rendering by vectorizing collision coefficients across wavelengths $\lambda$ [S1]. Control extinction coefficients $\mu_a^c(\lambda)$ and $\mu_s^c(\lambda)$ are set to the minimum values across all wavelengths, while $\bar{\mu}$ bounds the maximum total extinction across all wavelengths [S1]. Probabilities are computed using vectorized operations [S1]. When a control component is sampled, the local collision weight is a vector of ones $(1, \dots, 1)^{N_\lambda}$, avoiding per-wavelength weight updates for the control branch [S1].

## Operating characteristics
*   **Time Complexity:** The number of spatially-varying extinction lookups is reduced compared to standard delta tracking [S1]. The relative cost $\rho = N_{dcmp}/N_{delta} = 1 - \mu_t^c/\bar{\mu}$ when $\mu_t^c$ is a minorant [S1].
*   **Memory Complexity:** Requires storage for the octree structure (typically depth 8–10) and voxel data (e.g., VDB) [S1].
*   **Measured Performance:** In a thick cloud scene, decomposition tracking avoided 42% of extinction lookups required by delta tracking, and 58% in a thin cloud scene [S1]. With octree depth 8, lookup time reduced from 1030s to 536s, and tracker time from 299s to 280s in a thick cloud scene [S1].
*   **Strengths:** Identical noise patterns to delta tracking; significant reduction in lookup costs for heterogeneous volumes; handles chromatic coefficients efficiently in the weighted variant [S1].
*   **Limitations:** Requires tight bounds for control and residual components for optimal performance; octree traversal overhead increases with depth; if $\mu_t^c > \min(\mu_t(\mathbf{x}))$, variance increases and performance degrades [S1].
*   **When to Use:** Heterogeneous volumes with significant spatial variation in extinction, especially when spectral accuracy is required [S1].
*   **When Not to Use:** Homogeneous volumes (no benefit), or when tight bounds cannot be computed efficiently [S1].

## Background
Weighted Decomposition Tracking was introduced in the paper "Spectral and Decomposition Tracking for Rendering Heterogeneous Volumes" by Novák et al. [S1]. It builds on the concept of decomposition tracking for transmittance estimation [Novák et al. 2014] and extends it to free-path sampling [S1]. The analog version was previously limited by the requirement for non-negative residual extinction, which prevented spectral rendering [S1]. The weighted version removes this limitation, enabling efficient spectral tracking [S1]. It is used in production path tracers with VDB voxel storage and octree acceleration structures [S1].

## Sources
[S1] Spectral and Decomposition Tracking for Rendering Heterogeneous Volumes, https://media.disneyanimation.com/uploads/production/publication_asset/158/asset/SpectralAndDecompositionTracking.pdf, fetched 2026-10-30
