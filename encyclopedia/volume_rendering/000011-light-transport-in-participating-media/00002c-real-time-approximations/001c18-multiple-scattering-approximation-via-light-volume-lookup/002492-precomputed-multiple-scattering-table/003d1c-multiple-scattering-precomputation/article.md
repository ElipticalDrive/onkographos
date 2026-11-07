# Multiple Scattering Precomputation

Also called: K-th Order Multiple Scattering Precomputation, Precomputed Multiple Scattering Orders

## Overview
This method computes the radiative transfer equation for participating media by splitting it into incremental terms for $K$ scattering orders, allowing precomputation of multiple scattering effects that would otherwise require exponentially complex recursive Monte-Carlo evaluation [S1]. It differs from [Bounced Multiple Scattering](3532-bounced-multiple-scattering.md) by using explicit integral precomputation of auxiliary fields rather than ray-bouncing, and from [GPU Implementation of Precomputed Multiple Scattering](4706-gpu-implementation-of-precomputed-multiple-scattering.md) by focusing on the mathematical formulation and data layout rather than hardware-specific optimization.

## How it works
**Inputs:**
*   Medium properties: scattering coefficient $\sigma_{\{R,M\}}(\lambda)$, phase function $F_{\{R,M\}}(\theta)$, surface albedo $\bar{a}(\lambda)$, and extinction $t$.
*   Geometry: Observer position $P_O$, view direction $V$, light source direction $L$, surface normal $N'$ at surface point $P_g$.
*   Parameters: Maximum scattering order $K$.

**Outputs:**
*   Total scattered light intensity $I_S$ at the observer.

**Data Structures:**
1.  **Primary Lookup Table:** Stores $I_S^{(k)}$ for $k=1 \dots K$.
2.  **Auxiliary Lookup Tables (Optional but recommended for speed):**
    *   1D table for ambient intensity $A^{(k)}$.
    *   4D table for gathered intensity $I_G^{(k)}$ (dimensions: position, view direction, light direction, wavelength).
    *   *Note:* A 2D table for $I_G^{(k)}$ neglecting viewing direction is possible, losing phase function detail but incurring small error due to sphere integration averaging [S1].

**Step-by-Step Method:**

1.  **Compute First Order ($k=1$):**
    Calculate $I_S^{(1)}$ exactly using the single-scattering formulation (Eq. 7 in [S1]) and store the result in the primary lookup table.

2.  **Iterate for Higher Orders ($k = 2 \dots K$):**
    For each subsequent order, compute the scattered intensity $I_S^{(k)}$ using the recurrence relation. This requires the gathered intensity $I_G^{(k-1)}$ and ambient intensity $A^{(k-1)}$ from the previous order.

    **Formulas:**
    
    *Gathered Light Intensity* (Eq. 9):
    $$I_{G_{\{R,M\}}}^{(k)}(P, V, L, \lambda) = \int_{4\pi} F_{\{R,M\}}(\theta) I_{S_{\{R,M\}}}^{(k)}(P, \omega, L, \lambda) d\omega$$
    Where $\theta$ is the scattering angle between view direction $V$ and integration direction $\omega$.

    *Ambient Light Intensity at Surface* (Eq. 10):
    $$I_{A_{\{R,M\}}}^{(k)}(P_g, L, \lambda) = \int_{2\pi} (N' \cdot \omega) I_{S_{\{R,M\}}}^{(k)}(P_g, \omega, L, \lambda) d\omega$$

    *Composite Ambient Light Intensity at Observer* (Eq. 11):
    $$A_{\{R,M\}}^{(k)}(P_O, V, L, \lambda) = \bar{a}(\lambda) I_{A_{\{R,M\}}}^{(k)}(P_O, L, \lambda) e^{-t(P_O P_g \lambda)}$$
    *Note:* The source text contains a typo in Eq. 11, using $I_A$ on the RHS where $A$ is defined on the LHS, and $i$ vs $t$ for extinction. The formula above corrects this to standard notation consistent with the definition of composite ambient light [S1]. If the view ray $V$ intersects the planetary surface, $P_g$ is the intersection point; otherwise, the reflection term $R=0$.

    *Scattered Light Intensity at Observer* (Eq. 12):
    $$I_{S_{\{R,M\}}}^{(k)}(P_O, V, L, \lambda) = A_{\{R,M\}}^{(k-1)}(P_O, V, L, \lambda) + \frac{\sigma_{\{R,M\}}(\lambda)}{4\pi} \cdot \int_{P_a}^{P_b} I_{G_{\{R,M\}}}^{(k-1)}(P, V, L, \lambda) \rho_{\{R,M\}}(P) e^{-t(P_a P_\lambda)} ds$$
    Here, $P_a$ and $P_b$ define the integration path through the medium.

3.  **Final Summation:**
    Combine all orders to get the total scattering intensity (Eq. 13):
    $$I_S = \sum_{k=1}^K I_S^{(k)}$$

**Implementation Note:**
When computing $A^{(k)}$ and $I_G^{(k)}$ for step 2, two approaches exist:
1.  Compute them directly during the evaluation of Eq. 12 (more accurate, slower).
2.  Precompute them into auxiliary lookup tables in each iteration prior to calculating $I_S^{(k)}$ (faster, minimal quality difference) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_multiple_scattering(K, medium_params, geometry):
    """
    Computes total scattered light intensity for K orders.
    """
    # Initialize storage for scattered intensities
    # I_S[k] stores the result for order k
    I_S = [None] * (K + 1)
    
    # Optional auxiliary tables for speed
    # A_table[k] stores A^{(k)}
    # I_G_table[k] stores I_G^{(k)}
    A_table = [None] * (K + 1)
    I_G_table = [None] * (K + 1)

    # Step 1: Compute first order exactly
    # Uses Eq. 7 from source (single scattering)
    I_S[1] = compute_single_scattering(geometry, medium_params)
    
    # Precompute auxiliary fields for k=1 if using table approach
    # This involves integrating I_S[1] to get I_G[1] and A[1]
    I_G_table[1] = compute_gathered_intensity(I_S[1], medium_params.phase_function)
    A_table[1] = compute_ambient_intensity(I_S[1], medium_params.surface_albedo, geometry)

    # Step 2: Iterate for higher orders
    for k in range(2, K + 1):
        # Retrieve previous order's scattered intensity
        I_S_prev = I_S[k-1]
        
        # Compute current order's scattered intensity using Eq. 12
        # Term 1: Composite ambient from previous order
        ambient_term = A_table[k-1]
        
        # Term 2: Integral of gathered light from previous order
        # Requires I_G_table[k-1]
        gathered_term = compute_scattering_integral(
            I_G_table[k-1], 
            medium_params.scattering_coefficient,
            medium_params.extinction,
            geometry.path_bounds
        )
        
        I_S[k] = ambient_term + gathered_term
        
        # Precompute auxiliary fields for current order for next iteration
        # This is the "faster" approach mentioned in [S1]
        I_G_table[k] = compute_gathered_intensity(I_S[k], medium_params.phase_function)
        A_table[k] = compute_ambient_intensity(I_S[k], medium_params.surface_albedo, geometry)

    # Step 3: Sum all orders
    total_intensity = sum(I_S[1:K+1])
    return total_intensity

def compute_gathered_intensity(I_S_k, phase_function):
    """
    Computes Eq. 9: Integral over 4pi of phase function * scattered intensity.
    """
    # Discretize sphere integration
    integral = 0.0
    for omega in sample_sphere():
        theta = angle_between(view_direction, omega)
        integral += phase_function(theta) * I_S_k(omega) * solid_angle(omega)
    return integral

def compute_ambient_intensity(I_S_k, albedo, geometry):
    """
    Computes Eq. 10 and Eq. 11.
    1. Integrate I_S_k over 2pi at surface point Pg.
    2. Multiply by albedo and extinction to observer Po.
    """
    # Eq. 10
    surface_integral = 0.0
    for omega in sample_hemisphere(geometry.surface_normal):
        surface_integral += dot(geometry.surface_normal, omega) * I_S_k(omega) * solid_angle(omega)
        
    I_A = surface_integral
    
    # Eq. 11
    # Check if view ray intersects surface to get Pg
    if geometry.view_ray_intersects_surface():
        Pg = geometry.intersection_point
        extinction_factor = exp(-extinction_coefficient * distance(geometry.observer, Pg))
        return albedo * I_A * extinction_factor
    else:
        return 0.0 # R=0 if no intersection
```

## Operating characteristics
*   **Complexity:** The method avoids the exponential complexity of recursive Monte-Carlo by limiting scattering to $K$ orders [S1]. The cost is proportional to $K$ times the cost of evaluating the integrals for $I_G$ and $A$.
*   **Memory:** Requires storage for $K$ orders of scattered intensity, plus auxiliary tables for $A^{(k)}$ (1D) and $I_G^{(k)}$ (4D or 2D) [S1].
*   **Performance:** Precomputing auxiliary fields into lookup tables is significantly faster than computing them directly during Eq. 12 evaluation, with minimal quality difference [S1].
*   **Limitations:** Accuracy depends on $K$. Higher orders contribute less but are necessary for high-albedo environments. The approximation neglects phase function details if using the 2D $I_G$ table [S1].

## Background
This method is based on the precomputation algorithm for atmospheric scattering presented by [Bruneton and Neyret 2008] and [Elek 2009], as reviewed in [S1]. It was developed to enable real-time rendering of large-scale natural participating media (atmosphere and water) by making multiple scattering computationally feasible.

## Sources
[S1] Real-time spectral scattering in large-scale natural participating media, https://doi.org/10.1145/1925059.1925074, fetched 2026-10-28
