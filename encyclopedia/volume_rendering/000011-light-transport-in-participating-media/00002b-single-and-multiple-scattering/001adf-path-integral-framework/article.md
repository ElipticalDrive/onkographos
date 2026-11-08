# Path Integral Framework

Also called: PI approach, Path Integral Formulation, Path Integral Formulation of the Rendering Equation, Path Space Formulation, Path-Integral Formulation of Light Transport, Veach's path-integral formulation

## Overview
The Path Integral Framework formulates light transport simulation as a Monte Carlo integration problem over the space of light transport paths, serving as the basis for combining photon density estimation and Monte Carlo path integration [S1]. It expresses the radiometric measurement (e.g., pixel intensity) as a non-recursive integral over path space, derived from the recursive rendering equation via von Neumann series expansion [S4, unofficial]. It sets itself apart from its siblings by providing the rigorous mathematical measure and contribution function definition that allows disparate sampling techniques (such as [Delta tracking](985-delta-tracking.md) or [Photon-based Methods](5620-photon-based-methods.md)) to be unified under a single variance-reduction framework like [Multiple Importance Sampling for Volumetric Path Tracing](1167-multiple-importance-sampling-for-volumetric-path-tracing.md).

## How it works
**Inputs and Outputs**
*   **Input:** Scene geometry (surfaces $\mathcal{M}$), participating media properties ($\sigma_t, \sigma_s, \sigma_a, p(\omega, \omega')$), light source emission $L_e$, and detector sensitivity $W_e$.
*   **Output:** Radiometric measurement $I$ (e.g., pixel intensity).

**Data Structures**
A path $\bar{\mathbf{x}}$ is an ordered sequence of vertices $\bar{\mathbf{x}} = (\mathbf{x}_0, \mathbf{x}_1, \dots, \mathbf{x}_N)$ where $N \ge 1$ [S2, unofficial].
*   $\mathbf{x}_0$: Vertex on a light source.
*   $\mathbf{x}_N$: Vertex on the eye sensor (or camera plane).
*   $\mathbf{x}_i$ ($0 < i < N$): Intermediate vertices on surfaces and/or in participating media [S1].
*   The path has $N$ segments and $N+1$ vertices [S1].

**Formulas and Definitions**
The measurement $I$ is defined as the integral over the path space $\Omega$:
$$ I = \int_{\Omega} f(\bar{\mathbf{x}}) d\mu(\bar{\mathbf{x}}) $$
where $f(\bar{\mathbf{x}})$ is the measurement contribution function and $d\mu(\bar{\mathbf{x}})$ is the differential path measure [S2, unofficial].

1.  **Measurement Contribution Function:**
    $$ f(\bar{\mathbf{x}}) = L_e(\mathbf{x}_0) T(\bar{\mathbf{x}}) W_e(\mathbf{x}_N) \quad (1) $$
    where $L_e(\mathbf{x}_0)$ is the emitted radiance at the source, $W_e(\mathbf{x}_N)$ is the detector sensitivity, and $T(\bar{\mathbf{x}})$ is the path throughput [S1].

2.  **Path Throughput:**
    $$ T(\bar{\mathbf{x}}) = \left[ \prod_{i=0}^{N-1} G(\mathbf{x}_i, \mathbf{x}_{i+1}) T_{\mathbf{r}}(\mathbf{x}_i, \mathbf{x}_{i+1}) \right] \left[ \prod_{i=1}^{N-1} \rho(\mathbf{x}_i) \right] \quad (2) $$
    where:
    *   $T_{\mathbf{r}}(\mathbf{x}_i, \mathbf{x}_{i+1})$ is the transmittance along the segment, typically $e^{-\tau}$ where $\tau$ is the optical depth [S3].
    *   $\rho(\mathbf{x}_i)$ represents the scattering term (phase function and density) at intermediate vertices [S1].
    *   $G(\mathbf{x}_i, \mathbf{x}_{i+1})$ is the geometry term for the segment.

3.  **Geometry Term:**
    $$ G(\mathbf{x}_n \leftrightarrow \mathbf{x}_{n+1}) = V(\mathbf{x}_n \leftrightarrow \mathbf{x}_{n+1}) G_0(\mathbf{x}_n \leftrightarrow \mathbf{x}_{n+1}) \quad (3) $$
    where $V$ is the mutual visibility function (1 if visible, 0 otherwise) [S2, unofficial]. The visibility-free geometric term $G_0$ is:
    $$ G_0(\mathbf{x}_n \leftrightarrow \mathbf{x}_{n+1}) := \frac{|\mathbf{n}(\mathbf{x}_n) \cdot \boldsymbol{\omega}_n| |\mathbf{n}(\mathbf{x}_{n+1}) \cdot -\boldsymbol{\omega}_n|}{\|\mathbf{x}_{n+1} - \mathbf{x}_n\|^2} \quad (4) $$
    where $\boldsymbol{\omega}_n = (\mathbf{x}_{n+1} - \mathbf{x}_n)/\|\mathbf{x}_{n+1} - \mathbf{x}_n\|$ is the unit vector from $\mathbf{x}_n$ to $\mathbf{x}_{n+1}$, and $\mathbf{n}$ is the unit-normal field at the vertex [S2, unofficial].
    *   Note: For vertices in a medium, the cosine term $|\mathbf{n} \cdot \boldsymbol{\omega}|$ is replaced by 1, as there is no surface normal [S1].

4.  **Differential Path Measure:**
    $d\mu(\bar{\mathbf{x}})$ is a product measure corresponding to area integration for surface vertices and volume integration for medium vertices [S1].

**Algorithm: Monte Carlo Estimation**
To estimate $I$, one samples paths $\bar{\mathbf{x}}$ from a probability density function $p(\bar{\mathbf{x}})$ over $\Omega$ and computes the unbiased estimator:
$$ \hat{I} = \frac{1}{S} \sum_{j=1}^{S} \frac{f(\bar{\mathbf{x}}_j)}{p(\bar{\mathbf{x}}_j)} $$
where $S$ is the number of samples [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def path_integral_estimator(scene, camera, num_samples):
    """
    Estimates the radiometric measurement I using Monte Carlo integration
    over the path space defined by the Path Integral Framework.
    """
    total_estimate = 0.0
    
    for _ in range(num_samples):
        # 1. Generate a path sample x_bar from a proposal distribution p(x_bar)
        # This involves sampling x_0 from light sources, x_N from camera,
        # and intermediate vertices x_1...x_{N-1} from surfaces/medium.
        # The specific sampling strategy (e.g., volumetric path tracing)
        # determines p(x_bar).
        path = sample_path(scene, camera)
        
        if path is None:
            continue
            
        # 2. Calculate the measurement contribution function f(x_bar)
        # f(x_bar) = L_e(x_0) * T(x_bar) * W_e(x_N)
        
        # Emission at source
        L_e = scene.get_emission(path.vertices[0])
        
        # Detector sensitivity at sensor
        W_e = camera.get_sensitivity(path.vertices[-1])
        
        # Path throughput T(x_bar)
        # T(x_bar) = [Product of Geometry * Transmittance for segments] * 
        #            [Product of Scattering terms for inner vertices]
        throughput = 1.0
        
        for i in range(len(path.vertices) - 1):
            x_i = path.vertices[i]
            x_next = path.vertices[i+1]
            
            # Calculate visibility-free geometry G_0
            # Handle medium vertices: if x_i is in medium, normal term is 1
            # If x_i is on surface, normal term is |n_i . omega_i|
            if is_surface_vertex(x_i):
                omega_i = normalize(x_next - x_i)
                cos_theta_i = abs(dot(scene.get_normal(x_i), omega_i))
            else:
                cos_theta_i = 1.0
                
            if is_surface_vertex(x_next):
                omega_next = normalize(x_i - x_next)
                cos_theta_next = abs(dot(scene.get_normal(x_next), omega_next))
            else:
                cos_theta_next = 1.0
                
            dist_sq = squared_length(x_next - x_i)
            if dist_sq == 0:
                throughput = 0.0
                break
                
            G_0 = (cos_theta_i * cos_theta_next) / dist_sq
            
            # Visibility V
            V = 1.0 if scene.is_visible(x_i, x_next) else 0.0
            
            G = V * G_0
            
            # Transmittance T_r (Beer-Lambert)
            # T_r = exp(-integral of sigma_t along segment)
            T_r = compute_transmittance(scene, x_i, x_next)
            
            throughput *= G * T_r
            
            # Scattering term rho for inner vertices (not source or sensor)
            if i > 0: # x_i is an inner vertex
                # rho includes phase function p(omega_in, omega_out) and density
                # This term is part of the product in Eq 2 for i=1 to N-1
                # Note: The loop above iterates segments. The scattering term 
                # is associated with the vertex x_i.
                # We multiply by rho(x_i) here.
                # rho(x_i) = sigma_s(x_i) * p(omega_prev, omega_curr)
                # For the first inner vertex (i=1), omega_prev is from x_0 to x_1.
                # For subsequent, it's from x_{i-1} to x_i.
                pass 
                
        # Correction: The throughput calculation in the loop above accumulates 
        # segment terms. The scattering terms rho(x_i) must be multiplied 
        # for each inner vertex.
        for i in range(1, len(path.vertices) - 1):
            x_i = path.vertices[i]
            # Get incoming and outgoing directions at x_i
            omega_in = normalize(x_i - path.vertices[i-1])
            omega_out = normalize(path.vertices[i+1] - x_i)
            
            # Scattering coefficient and phase function
            sigma_s = scene.get_scattering_coefficient(x_i)
            phase = scene.get_phase_function(x_i, omega_in, omega_out)
            
            throughput *= sigma_s * phase
            
        # 3. Calculate the estimator weight
        # weight = f(x_bar) / p(x_bar)
        # p(x_bar) is the probability density of generating this specific path
        # under the sampling strategy used in sample_path().
        # This requires the sampler to return p(x_bar) or its components.
        p_path = path.probability_density()
        
        if p_path == 0:
            continue
            
        weight = (L_e * throughput * W_e) / p_path
        total_estimate += weight
        
    return total_estimate / num_samples
```

**Edge Cases and Pitfalls**
*   **Infinite-Dimensional Space:** The integration is performed over an infinite-dimensional path space using a highly non-intuitive differential measure $\mathcal{D}x$, making the mathematics exceptionally complex [S3].
*   **Visibility:** The visibility function $V$ is discontinuous, leading to high variance if not handled carefully (e.g., via shadow rays or next-event estimation) [S2, unofficial].
*   **Medium Vertices:** The definition of the geometry term $G_0$ changes for vertices inside participating media (cosine terms become 1), which must be correctly implemented to avoid bias [S1].

## Subtopics

### [Surrounding Path Contribution (WKB Expansion)](1300-surrounding-path-contribution-wkb-expansion.md)
Approximates the path integral by restricting it to paths "around" the Most Probable Path (MPP), exploiting the geometric concentration of contributions.

### [Null-Scattering Path Integral Framework](3070-null-scattering-path-integral-framework.md)
Extends the framework to heterogeneous media by explicitly including null-scattering vertices in the path configuration, enabling rigorous measure definition for variance reduction.

### [Metropolis Light Transport for Participating Media](8722-metropolis-light-transport-for-participating-media.md)
Uses Metropolis-Hastings sampling to explore path space, efficient for high-order scattering and highly anisotropic media where standard importance sampling converges slowly.

### [Monte Carlo Integration](9784-monte-carlo-integration.md)
The numerical technique used to estimate the path integral, averaging function evaluations at random points weighted by their PDF.

## Operating characteristics
*   **Complexity:** The theoretical complexity is determined by the dimensionality of the path space and the cost of evaluating the contribution function $f(\bar{\mathbf{x}})$, which involves visibility checks and transmittance integrals.
*   **Variance:** High variance is typical due to the discontinuous visibility function and the exponential decay of throughput in dense media.
*   **Strengths:** Provides a unified framework for combining multiple sampling strategies (e.g., [Delta tracking](985-delta-tracking.md) and surface path tracing) via [Multiple Importance Sampling](1167-multiple-importance-sampling-for-volumetric-path-tracing.md).
*   **Limitations:** Direct Monte Carlo integration over path space is slow for complex scenes with many bounces or highly heterogeneous media without advanced variance reduction techniques.

## Background
The Path Integral Framework was introduced by Veach [1997] for surface rendering and generalized to participating media by Pauly et al. [2000] [S1]. It reformulates the recursive rendering equation into a non-recursive integral over path space, derived via von Neumann series expansion [S4, unofficial]. This formulation is foundational for modern unbiased volumetric renderers and differentiable rendering techniques [S2, unofficial].

## Sources
[S1] Unifying points, beams, and paths in volumetric light transport simulation, https://doi.org/10.1145/2601097.2601219, fetched 2026-10-30
[S2] Path-Space Differentiable Rendering, https://core.ac.uk/download/541724175.pdf, fetched 2026-10-29 (unofficial)
[S3] PDF Practical Rendering of Multiple Scattering Effects in Participating Media, https://cseweb.ucsd.edu/~ravir/HRPIEG.pdf, fetched 2026-10-28
[S4] PDF A null-scattering path integral formulation of light transport, https://steven.schuerstedt.com/wp-content/uploads/2020/10/schuerstedt.pdf, fetched 2026-10-29 (unofficial)
