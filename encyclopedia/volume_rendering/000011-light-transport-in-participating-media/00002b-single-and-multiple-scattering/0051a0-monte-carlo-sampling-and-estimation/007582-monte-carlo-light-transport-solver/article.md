# Monte-Carlo Light Transport Solver

Also called: Monte-Carlo Estimator for Gaussian Surfels

## Overview
The Monte-Carlo Light Transport Solver is a method for estimating global illumination (direct and indirect) in scenes represented by Gaussian surfels. It solves the full light transport equation by performing Monte-Carlo estimation of a double integral over interaction points, importance sampling positions using the underlying Gaussian distributions [S1]. It is distinguished from its sibling [Hybrid Light Transport Solver](23291-hybrid-light-transport-solver.md) by estimating the entire light transport (including direct illumination) stochastically, whereas the hybrid solver factors out direct illumination to reduce variance [S1]. Compared to the [Center-to-Center Approximation](23291-center-to-center-approximation.md), it provides an unbiased estimate of the propagated incoming radiance integral but at the cost of higher variance and computational expense [S1].

## How it works
The solver estimates the outgoing radiance by evaluating the double integral of light transport between Gaussian surfels. It relies on the statistical estimation of the integral form of the radiative transfer equation, generating random paths (interaction points) through the medium [Parent].

**Inputs and Outputs**
*   **Input:** A set of Gaussian surfels representing the scene geometry and material properties. Each surfel $i$ is defined by its center position $\mathbf{c}_i$, covariance matrix $\Sigma_i$, and opacity $\alpha_i$.
*   **Input:** Light source distributions (position, intensity, spectral power).
*   **Input:** Camera parameters.
*   **Output:** Estimated radiance $L$ for each pixel or surfel.

**Data Structures**
*   **Gaussian Surfel:** A structure containing the mean position $\mathbf{c}$, the $3 \times 3$ covariance matrix $\Sigma$ (or its Cholesky decomposition), and scalar opacity $\alpha$.
*   **Interaction Point:** A 3D position $\mathbf{x}$ sampled from a Gaussian distribution.

**Algorithm Steps**
1.  **Sample Interaction Points:** For each surfel $i$ contributing to the radiance estimate, sample a position $x$ from its underlying Gaussian distribution $G_i(x)$. Similarly, sample a target position $x'$ from the distribution of the receiving surfel $j$, $G_j(x')$ [S1].
2.  **Estimate Double Integral:** Compute the contribution of the path between $x$ and $x'$ to the double integral. This involves calculating the transmittance between $x$ and $x'$ and the scattering phase function. The contribution is weighted by the probability density functions (PDFs) of the sampled points [S1].
3.  **Accumulate Samples:** Average the contributions over $N$ independent samples to estimate the final radiance:
    $$ L \approx \frac{1}{N} \sum_{k=1}^{N} \frac{f(x_k, x'_k)}{p(x_k) p(x'_k)} $$
    where $f(x, x')$ is the path contribution (emission, scattering, transmittance) and $p(x)$ is the PDF of the Gaussian distribution from which $x$ was sampled [S1, Parent].
4.  **Render:** Composite the estimated radiance into the final image.

**Parameters**
*   **Time Step ($\Delta t$):** A parameter used in the progressive refinement context where this solver is often embedded. In experiments, $\Delta t = 64$ was used [S1].
*   **Number of Samples ($N$):** The number of Monte-Carlo samples per surfel pair. Not explicitly fixed in the source, but variance reduction is a key concern [S1].

**Edge Cases and Pitfalls**
*   **High Variance:** The pure Monte-Carlo solver produces higher variance than hybrid approaches, especially in scenes with complex lighting or occlusions [S1].
*   **Computational Cost:** Convergence can be slow. For example, solving the full light transport with progressive refinement alone would theoretically require significant time (e.g., $0.0183s \times 100K \approx 0.5$ hours) to converge, and the Monte-Carlo solver adds overhead due to the stochastic nature of the estimation [S1].
*   **Approximation Dependency:** The solver is often used in conjunction with approximations (like the Center-to-Center Approximation) to make the optimization tractable. Removing these approximations increases the computational burden significantly [S1].

## Subtopics

### Hybrid Light Transport Solver
A hybrid approach that factors out direct illumination from the light transport equation and solves the remaining indirect illumination using progressive refinement, followed by the Monte-Carlo solver for indirect effects [S1]. It is more efficient and has lower variance for similar compute time compared to the pure Monte-Carlo solver estimating full global illumination [S1]. It reduces variance significantly at the cost of slightly more computation time [S1]. If progressive refinement were applied to the entire light transport, convergence would theoretically require $0.0183s \times 100K \approx 0.5$ hours [S1].

## Operating characteristics
*   **Time Complexity:** Higher than hybrid solvers. Training time increases by 35.77% compared to the hybrid solver [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On the "ComplexBall" scene, the pure Monte-Carlo solver decreased PSNR by 0.25 dB and increased training time by 35.77% compared to the hybrid solver [S1].
*   **Strengths:** Unbiased estimation of global illumination; conceptually simple implementation of the double integral.
*   **Limitations:** High variance; slow convergence; sensitive to geometric changes due to the constant outgoing radiance assumption in some contexts [S1].
*   **When to Use:** When unbiased global illumination is required and computational resources are sufficient to handle the variance, or as a baseline for comparison.
*   **When Not to Use:** When low variance and fast convergence are critical, in which case the [Hybrid Light Transport Solver](23291-hybrid-light-transport-solver.md) is preferred [S1].

## Background
The method is derived from the principles of differentiable light transport with Gaussian surfels, specifically in the context of efficient relighting and geometry reconstruction [S1]. It draws inspiration from light cuts and radiosity methods, adapting them to the differentiable rendering framework [S1]. The approach was introduced in the paper "Differentiable Light Transport with Gaussian Surfels via Adapted Radiosity for Efficient Relighting and Geometry Reconstruction" [S1].

## Sources
[S1] Differentiable Light Transport with Gaussian Surfels via Adapted Radiosity for Efficient Relighting and Geometry Reconstruction, https://doi.org/10.1145/3763305, fetched 2026-10-27
