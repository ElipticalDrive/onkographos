# Absorption Term Restriction

Also called: Absorption-only restriction, Non-emissive absorption removal

## Overview
Absorption Term Restriction is an optical model simplification that restricts the volume density model to the absorption term only, effectively treating the medium as non-emissive and non-scattering for the purpose of path termination logic. It solves the problem of unnecessary computational overhead in participating media where emission and scattering are negligible or explicitly disabled. Unlike the full Emission-Absorption Model, it removes absorption collisions from the transport simulation, redistributing their probability mass to scattering and null-collision events to reduce variance in specific spectral tracking contexts.

## How it works
The method modifies the standard delta tracking or null-collision framework used in the parent Emission-Absorption Model. It is primarily applied in spectral tracking implementations where the medium is known to be non-emissive.

**Inputs:**
*   Absorption coefficient $\mu_a(x, \lambda)$
*   Scattering coefficient $\mu_s(x, \lambda)$
*   Total extinction coefficient $\mu_t(x, \lambda) = \mu_a(x, \lambda) + \mu_s(x, \lambda)$
*   Control bounds $\bar{\mu}_t$ bounding $\mu_t$ across all wavelengths.

**Outputs:**
*   Modified control probabilities $P_s(x_j)$ and $P_n(x_j)$ for scattering and null-collision.
*   Absorption probability $P_a(x_j)$ set to 0.

**Data Structures:**
*   **3D Texture Map:** In hardware-accelerated variants, this map contains the integral of the absorption coefficient along the viewing ray segment for the interval $[s_f, s_b]$ at position $(s_f, s_b)$ [S1].
*   **Control Probabilities:** Vectorized versions of standard probability equations, recomputed after removing the absorption term from the normalization constant [S2].

**Steps:**
1.  **Disable Absorption Collisions:** Set the absorption collision probability to zero:
    $$ P_a(x_j) = 0 $$
2.  **Update Normalization Constant:** Remove the absorption term from the normalization constant $c$ (which typically represents the maximum extinction bound or control function) [S2].
3.  **Recompute Probabilities:** Recalculate the scattering probability $P_s(x_j)$ and null-collision probability $P_n(x_j)$. Since the absorption term is removed from the denominator (the normalization constant), these probabilities increase to compensate, ensuring the total probability sums to 1 [S2].
4.  **Spectral Handling (if applicable):** In spectral tracking, set $\bar{\mu}$ to bound the maximum value of $\mu_t(x, \lambda)$ across all wavelengths $\lambda$. Set components of control coefficients $\mu^{ca}$ and $\mu^{cs}$ to the minimum values of $\mu_a$ and $\mu_s$ respectively across all wavelengths [S2].

**Pitfalls:**
*   **Increased Computational Cost:** Disabling absorption increases the computational cost because paths must be simulated longer to terminate via scattering or null collisions, as absorption events no longer provide a termination mechanism [S2].
*   **Variance Reduction Condition:** This technique reduces variance and the magnitude of outliers only if the control bound $\bar{\mu}$ does not tightly bound $\mu_t(x)$ [S2]. If $\bar{\mu}$ is loose, the variance reduction benefit may be negated by the increased path lengths.

## Subtopics

### Reduced Termination Rates for Non-Emissive Media
A variant applied specifically when the medium is non-emissive. It reduces throughput by not simulating absorption collisions and redistributing the absorption probability among scattering and null-collision types. It is chosen when variance reduction is prioritized over raw throughput in non-emissive heterogeneous volumes [S2].

## Operating characteristics
*   **Time Complexity:** Increases per-path cost due to longer simulated paths required for termination via scattering or null collisions [S2].
*   **Memory Complexity:** Requires a 3D texture map containing the integral of the absorption coefficient for hardware-based implementations [S1].
*   **Strengths:** Reduces variance and the magnitude of outliers in spectral tracking if the control bound $\bar{\mu}$ does not tightly bound $\mu_t(x)$ [S2].
*   **Limitations:** Higher computational cost per path; only applicable to non-emissive media.

## Background
The concept originates from hardware-based volume rendering techniques using register combiners, where restricting the model to the absorption term required a 3D texture map containing the integral of the absorption coefficient [S1]. The specific application to delta tracking and spectral decomposition was formalized in the context of rendering heterogeneous volumes, where it serves as an optimization for non-emissive media [S2].

## Sources
[S1] Hardware-based view-independent cell projection, https://www.semanticscholar.org/paper/Hardware-based-view-independent-cell-projection-Weiler-Kraus/36615d19709bce724437ec3257e43a1916f7de26, fetched 2026-10-29
[S2] PDF Spectral and Decomposition Tracking for Rendering Heterogeneous Volumes, https://media.disneyanimation.com/uploads/production/publication_asset/158/asset/SpectralAndDecompositionTracking.pdf, fetched 2026-10-30
