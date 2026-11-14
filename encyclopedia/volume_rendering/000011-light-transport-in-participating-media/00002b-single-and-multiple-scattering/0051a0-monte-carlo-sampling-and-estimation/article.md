# Monte Carlo Sampling and Estimation

Also called: Stochastic Light Transport, Random Walk Methods

## Overview
Monte Carlo methods solve the radiative transfer equation by estimating radiance through random sampling of light paths and interaction points [S1]. They are distinguished from deterministic approaches (like the Zonal Method) by their ability to handle complex, heterogeneous geometries and high-dimensional integrals without discretizing the spatial domain into fixed grids [S1]. Unlike deterministic solvers, these methods converge to the correct solution as the number of samples increases, regardless of the scene's geometric complexity, making them the standard for high-fidelity volumetric rendering.

## Shared principles
All methods in this family rely on the statistical estimation of the integral form of the radiative transfer equation. The core mechanism involves generating random paths through the medium and estimating the contribution of each path to the final image.

**1. Random Path Generation**
Paths are constructed by sampling interaction points (scattering or absorption events) along rays. The distance to the next interaction is sampled from the probability density function (PDF) defined by the extinction coefficient $\sigma_t$. For homogeneous media, this is an exponential distribution [G]. For heterogeneous media, techniques like Delta Tracking are used to sample the continuous distribution exactly [S4].

**2. Estimator Formulation**
The radiance $L$ is estimated as an average of $N$ independent samples $X_i$:
$$ L \approx \frac{1}{N} \sum_{i=1}^{N} \frac{f(X_i)}{p(X_i)} $$
where $f(X_i)$ is the contribution of the path (including emission, scattering, and transmittance) and $p(X_i)$ is the probability of generating that path [G]. Unbiased estimators require that $E[L] = L_{true}$ [G].

**3. Variance Reduction**
To improve convergence, methods employ variance reduction techniques:
*   **Importance Sampling:** Sampling interaction points or directions according to a distribution that approximates the integrand's shape (e.g., sampling near light sources or boundaries) [S1].
*   **Multiple Importance Sampling (MIS):** Combining multiple sampling strategies (e.g., light sampling vs. path sampling) to reduce variance in difficult lighting scenarios [S1167].
*   **Biased Sampling:** Deliberately biasing the random walk to exit the medium faster (e.g., Dwivedi Sampling) to reduce the number of "lost" paths, accepting a small bias for significant variance reduction [S5537].

## Subtopics

### [Delta tracking](985-delta-tracking.md)
Delta tracking is an unbiased Monte Carlo method for sampling the distance to the next interaction in heterogeneous media. It solves the problem that standard exponential sampling requires a constant extinction coefficient. Unlike ray marching, which introduces bias by discretizing space, Delta Tracking uses rejection sampling against a constant upper bound (majorant) to provide an exact sample of the continuous free-path distribution. It handles heterogeneity without grid constraints, distinguishing it from [Homogeneous Medium Sampling](980-homogeneous-medium-sampling.md). See [Delta Tracking](985-delta-tracking.md).

### [Multiple Importance Sampling for Volumetric Path Tracing](1167-multiple-importance-sampling-for-volumetric-path-tracing.md)
This method reduces variance in direct lighting estimation by combining contributions from phase function sampling and next event estimation (light sampling). It specifically addresses the discontinuity in sampling history caused by index-matching boundaries, where the sampling strategy generating the vertex differs from the strategy connecting to the light. It caches PDFs and transmittance factors across bounces to ensure correct MIS weights. See [Multiple Importance Sampling for Volumetric Path Tracing](1167-multiple-importance-sampling-for-volumetric-path-tracing.md).

### [Dwivedi Sampling](5537-dwivedi-sampling.md)
Dwivedi sampling is a variance reduction technique for high-scattering, isotropic scenarios. It biases the random walk to exit the medium quickly by approximating the local geometry as a slab and analytically estimating the exit likelihood. This reduces "lost" paths wandering deep within the volume. It differs from standard distance sampling by incorporating geometric boundary information into the sampling PDF. See [Dwivedi Sampling](5537-dwivedi-sampling.md).

### [Optimized Monte Carlo Integrator](9013-optimized-monte-carlo-integrator.md)
This method computes single-scattering radiance in box-shaped media for real-time rendering. It decomposes the medium into trapezoidal segments and caches integration variables to allow importance sampling proportional to geometry-dependent transmittance. It avoids redundant geometric traversal and analytic integration of unoccluded single scattering, setting it apart from standard ray marching and Delta Tracking. See [Optimized Monte Carlo Integrator](9013-optimized-monte-carlo-integrator.md).

## Background
Monte Carlo methods for participating media were formalized in the early 2000s as a response to the limitations of deterministic solvers in complex scenes. The survey by [S1] categorizes these methods, distinguishing between those using constant step distances and those sampling functions of optical depth $\kappa t$. Key early implementations include Light Tracing [S1], Bidirectional Path Tracing [S1], and Photon Maps [S1]. The field has since evolved to include advanced variance reduction techniques like MIS for index-matching boundaries [S1167] and geometry-aware sampling [S5537, S9013].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
