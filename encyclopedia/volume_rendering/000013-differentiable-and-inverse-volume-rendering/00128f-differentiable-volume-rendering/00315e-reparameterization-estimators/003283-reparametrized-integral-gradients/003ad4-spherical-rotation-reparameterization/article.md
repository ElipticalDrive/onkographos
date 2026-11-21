# Spherical Rotation Reparameterization

Also called: Discontinuity removal via rotation, Reparameterization with spherical rotations, Rotation-based change of variables

## Overview
Spherical Rotation Reparameterization is a technique for differentiable volume rendering that eliminates discontinuities in gradient estimates by applying a spherical rotation to the integration domain [S1][S2, unofficial]. It solves the problem of non-differentiable integrals caused by scene parameters $\theta$ shifting discontinuity points (e.g., object silhouettes) across the sphere [S1][S2, unofficial]. Unlike [Warped-Area Sampling](8835-warped-area-sampling.md), which explicitly samples and integrates over discontinuity boundaries, this method makes the discontinuity point parameter-independent through a change of variables, thereby removing the need for explicit boundary sampling [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An integrand $f(\omega, \theta)$ defined on the unit sphere $S^2$, where $\omega$ is a direction and $\theta$ represents scene parameters (e.g., object position, camera pose). The integrand typically has discontinuities dependent on $\theta$.
*   **Output:** A differentiable estimate of the integral $I = \int_{S^2} f(\omega, \theta) d\omega$ and its gradient with respect to $\theta$.

**Data Structures:**
*   **Rotation $R(\omega, \theta)$:** A function mapping a direction $\omega$ to a rotated direction $\omega'$. It is parameterized by $\theta$.
*   **Sampling Distribution $p(\omega, \theta_0)$:** The probability density function used to generate samples $\omega_i$. It uses a fixed parameter $\theta_0$ independent of the current $\theta$ to ensure sample independence from the gradient path [S1].

**Steps:**
1.  **Identify Discontinuity Motion:** Determine how the discontinuity point $T(\pi)$ moves with respect to the parameter $\pi$ (where $\pi$ is a component of $\theta$) [S2, unofficial].
2.  **Construct Rotation $R$:** Choose a spherical rotation $R(\omega, \pi)$ such that its derivative with respect to $\pi$ matches the derivative of the discontinuity trajectory:
    $$ \frac{\partial R}{\partial \pi} = \frac{\partial T}{\partial \pi} $$
    This ensures that the transformed discontinuity point $\omega_0$, defined by $R(\omega_0, \pi) = T(\pi)$, is independent of $\pi$ [S2, unofficial].
3.  **Reparameterize Integral:** Apply the change of variables to the integral:
    $$ I = \int_{S^2} f(\omega, \theta) d\omega = \int_{S^2} f(R(\omega, \theta), \theta) d\omega $$
    The Jacobian determinant of this spherical rotation is 1 [S1].
4.  **Monte Carlo Estimation:** Generate samples $\omega_i$ from a distribution $p(\omega, \theta_0)$ using a fixed parameter $\theta_0$ (typically the previous frame's parameter or a static default) [S1]. Compute the estimate:
    $$ E = \frac{1}{N} \sum_{i=1}^{N} \frac{f(R(\omega_i, \theta), \theta)}{p(\omega_i, \theta_0)} $$
    Using $\theta_0$ instead of $\theta$ in the sampling distribution removes the dependency of the samples on $\theta$, preventing discontinuous jumps in the sample set [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spherical_rotation_reparameterization(f, theta, theta_0, N, sample_func):
    """
    Estimate integral of f(w, theta) using spherical rotation reparameterization.
    
    Args:
        f: Integrand function f(omega, theta).
        theta: Current scene parameters.
        theta_0: Fixed parameters for sampling distribution.
        N: Number of samples.
        sample_func: Function that samples omega from p(omega, theta_0).
        
    Returns:
        Estimate of the integral I.
    """
    # 1. Determine rotation R based on theta
    # Note: In practice, R is constructed to approximate the motion of discontinuities.
    # The specific construction of R depends on the scene geometry and is detailed in [S1][S2, unofficial].
    R = construct_rotation(theta)
    
    total = 0.0
    for i in range(N):
        # 2. Sample from fixed distribution p(omega, theta_0)
        omega_i = sample_func(theta_0)
        
        # 3. Evaluate rotated integrand
        # The rotation R maps omega_i to the rotated domain
        omega_rotated = R(omega_i, theta)
        
        # 4. Compute importance weight
        # p(omega_i, theta_0) is the PDF of the sampling distribution at theta_0
        pdf_value = evaluate_pdf(omega_i, theta_0)
        
        # 5. Accumulate estimate
        # f is evaluated at the rotated direction and current theta
        f_val = f(omega_rotated, theta)
        total += f_val / pdf_value
        
    return total / N
```

**Parameters:**
*   **$\theta_0$:** The fixed parameter used for sampling. Typically set to the parameter value from the previous frame or a static initial value [S1].
*   **Rotation Approximation:** The rotation $R$ must accurately approximate the displacement of discontinuities. If the approximation is poor, bias remains [S1].

**Edge Cases and Pitfalls:**
*   **Large Support:** The method assumes the integrand has small angular support (vanishes outside a small neighborhood). If the support is large, the rotation approximation produces significant bias because the discontinuity structure is too complex to be modeled by a single rotation [S1].
*   **Precision:** The Jacobian is exactly 1, so no scaling factor is needed, but numerical precision in computing the rotation matrix and its inverse (if needed) is critical [S1].

## Subtopics

### Spherical Convolution for Large Support
A method to handle integrands with large support by convolving the integrand $f$ with a spherical kernel $k$, allowing the small-support rotation technique to be applied to the inner integral [S1]. It is chosen when the integrand's discontinuities are spread over a large area of the sphere, making direct rotation ineffective [S1]. The integral identity used is $\int_{S^2} f(\omega) d\omega = \int_{S^2} \int_{S^2} f(\mu) k(\mu, \omega) d\mu d\omega$, where $k$ satisfies $\int_{S^2} k(\mu, \omega) d\mu = 1$ [S1]. The kernel $k$ can be any smooth distribution, such as a von Mises-Fisher distribution, with a concentration parameter controlling the trade-off between bias and variance [S1]. Small kernels reduce bias (better rotation approximation) but increase variance (lower probability of hitting discontinuities) [S1]. The final estimator samples $\omega_i$ for the outer integral and $\mu_i \sim k(\cdot, \omega_i)$ for the inner integral, yielding:
$$ I \approx E = \frac{1}{N} \sum \frac{f(R_i(\mu_i, \theta), \theta) k(R_i(\mu_i, \theta), \omega_i(\theta), \theta)}{p(\omega_i(\theta), \theta) p_k(\mu_i)} $$
[S1].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of samples. The cost of constructing the rotation $R$ depends on the complexity of the scene geometry but is typically $O(1)$ per sample if precomputed or approximated [S1][S2, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory per sample, excluding the storage for the scene geometry required to determine discontinuity motion [S1].
*   **Measured Performance:** Produces gradient images that closely match ground truth for scenes with small-support integrands [S1]. Performance degrades with large-support integrands due to bias [S1].
*   **Strengths:** Eliminates the need for explicit boundary sampling; simple to implement; Jacobian is 1 [S1][S2, unofficial].
*   **Limitations:** High bias for large-support integrands; requires accurate estimation of discontinuity motion [S1][S2, unofficial].
*   **When to Use:** Suitable for differentiable rendering of scenes with sharp, localized discontinuities (e.g., specular highlights, small object silhouettes) [S1].
*   **When Not to Use:** Unsuitable for diffuse shading or large-area lights where the integrand support is large [S1].

## Background
Spherical Rotation Reparameterization was introduced by Loubet et al. in 2019 [S1][S2, unofficial]. It is part of the broader field of differentiable rendering, specifically addressing the challenge of differentiating through discontinuous integrals [S1][S2, unofficial]. The method builds on the idea of reparameterization estimators, which transform the integration domain to remove parameter dependency from discontinuities [S2, unofficial]. It is used in differentiable volume rendering and path tracing to enable gradient-based optimization of scene parameters [S1][S2, unofficial].

## Sources
[S1] Reparameterizing discontinuous integrands for differentiable rendering, https://doi.org/10.1145/3355089.3356510, fetched 2026-10-26
[S2] A Survey on Physics-based Differentiable Rendering - arXiv.org, https://arxiv.org/pdf/2504.01402, fetched 2026-10-29 (unofficial)
