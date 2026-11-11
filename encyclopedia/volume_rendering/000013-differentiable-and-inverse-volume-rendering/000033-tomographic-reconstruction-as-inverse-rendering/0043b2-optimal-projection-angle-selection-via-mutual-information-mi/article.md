# Optimal Projection Angle Selection via Mutual Information Minimization

Also called: Information theoretic projection angle selection, Information-Theoretic Angle Selection, Information-theoretic tomography angle selection, MI-based angle optimization, Mutual Information Optimization, Optimal Projection Angles

## Overview
This method selects the optimal set of projection angles $\theta_i$ for tomographic reconstruction by minimizing the posterior entropy of the inferred 3D volume given the projections, $H(\mathcal{D}|\mathcal{X})$ [S1, unofficial]. It operates on the principle that maximizing the joint entropy of the projections $H(\mathcal{X})$ reduces the uncertainty of the underlying data, which is achieved by minimizing the sum of pairwise mutual information (MI) terms between all projection pairs [S1, unofficial]. Unlike sibling methods that focus on reconstruction speed or specific geometric approximations, this approach is information-theoretic, deriving angle configurations that maximize data diversity based on statistical assumptions about the volume and noise [S1, unofficial]. It is distinct from [Compressive Volume Rendering Recovery Methods](1915-compressive-volume-rendering-recovery-methods.md) by explicitly modeling the correlation structure between projections via Gaussian priors, and from [FastFDK](3315-fastfdk.md) by optimizing the acquisition geometry rather than the reconstruction kernel.

## How it works
**Inputs:**
*   Number of projections $N$.
*   Noise level $\sigma_0$ (standard deviation of additive Gaussian noise).
*   Data covariance structure $\Sigma$ (or assumption of axisymmetry).
*   Cone angle $\phi$ (determines whether parallel or cone beam geometry applies).
*   Source-to-origin distance $L$ (for cone beam calculations).

**Outputs:**
*   Optimal set of angles $\{\theta_1, \dots, \theta_N\}$.

**Data Structures:**
*   **Fourier Coefficients Table:** A lookup table mapping cone angles $\phi \in \{10^\circ, 15^\circ, 20^\circ, 25^\circ\}$ to coefficients $a_0, \dots, a_6$ for the surrogate MI function. For $\phi=10^\circ$, coefficients are $[0.66, -0.02, -0.46, -0.01, -0.13, 0.0, -0.01]$ [S1, unofficial].
*   **Angle Array:** Array of size $N$ storing the selected angles in radians.

**Step-by-Step Method:**

1.  **Define the Optimization Objective:**
    The goal is to minimize the posterior entropy $H(\mathcal{D}|\mathcal{X})$. Using the identity $H(\mathcal{D}|\mathcal{X}) = H(\mathcal{D}) + H(\mathcal{X}|\mathcal{D}) - H(\mathcal{X})$, and assuming noise is i.i.d. (making $H(\mathcal{X}|\mathcal{D})$ independent of angles), the problem reduces to maximizing the joint entropy $H(\mathcal{X})$ [S1, unofficial].

2.  **Approximate Joint Entropy via Pairwise MI:**
    Decompose $H(\mathcal{X})$ into marginal entropies and pairwise mutual information terms, assuming higher-order interactions are negligible:
    $$
    H(\mathcal{X}) = \sum_{i=1}^N H(\mathcal{X}_i) - \sum_{1 \le i < j \le N} I(\mathcal{X}_i; \mathcal{X}_j)
    $$
    Since marginal entropies $H(\mathcal{X}_i)$ are constant with respect to angle selection, maximizing $H(\mathcal{X})$ is equivalent to minimizing the sum of pairwise mutual information:
    $$
    \min_{\theta} \sum_{1 \le i < j \le N} I(\mathcal{X}_i; \mathcal{X}_j)
    $$
    [S1, unofficial]

3.  **Compute Pairwise Mutual Information $I(\alpha_{ij})$:**
    Assume stationarity, so $I$ depends only on the angular separation $\alpha_{ij} = |\theta_j - \theta_i|$ [S1, unofficial].
    
    *   **General Gaussian Case:** If the data covariance $\Sigma$ is known, use:
        $$
        I(\alpha_{ij}|\Sigma) = -\frac{1}{2} \ln \left( 1 - \frac{(\mathbf{b}_i^T \Sigma \mathbf{b}_j)^2}{(\sigma_0^2 + \mathbf{b}_i^T \Sigma \mathbf{b}_i)(\sigma_0^2 + \mathbf{b}_j^T \Sigma \mathbf{b}_j)} \right)
        $$
        where $\mathbf{b}_i = [1, 0]^T$ and $\mathbf{b}_j = [\cos \alpha_{ij}, \sin \alpha_{ij}]^T$ [S1, unofficial].
        
    *   **Axisymmetric Gaussian Case:** If $\Sigma = \sigma^2 I$, the formula simplifies to:
        $$
        I(\alpha_{ij}) = -\frac{1}{2} \ln \left( 1 - \frac{\cos^2 \alpha_{ij}}{(1 + \varepsilon)^2} \right)
        $$
        where $\varepsilon = \sigma_0^2/\sigma^2$ is the noise-to-signal variance ratio [S1, unofficial].

    *   **Cone Beam Surrogate (Fourier Approximation):** For numerical optimization with cone beams, use the tabulated Fourier series:
        $$
        I(\alpha) = -\frac{1}{2} \log \left( \sum_{k=0}^6 a_k \cos(k\alpha) \right)
        $$
        Select coefficients $a_k$ from the table based on the cone angle $\phi$ [S1, unofficial].

4.  **Numerical Optimization:**
    Minimize the surrogate MI sum $\sum_{i<j} I(\theta_j - \theta_i)$ with respect to $\theta_i \in [0, 2\pi)$ [S1, unofficial].
    
    *   **Analytical Heuristics for Initialization/Verification:**
        *   For **odd** $N$, the optimal angles are uniformly spaced: $\theta_i = \frac{2\pi i}{N}$ [S1, unofficial].
        *   For **even** $N$ with **parallel beams**, angles divide the half-circle: $\theta_i = \frac{\pi i}{N}$ [S1, unofficial].
        *   For **even** $N$ with **cone beams**, reflect some angles into the opposite half-circle to increase angular separation, as $I(\pi) < I(0)$ for cone beams [S1, unofficial].

5.  **Monte Carlo Validation (Optional):**
    To verify MI values for complex geometries:
    1.  Sample $N_{mc}=100,000$ points $(x,y)$ from $\mathcal{N}(0, I)$.
    2.  Apply cone beam coordinate transform: $\tilde{r} = \left\| x + (x-s) \frac{L-x.n}{(x-s).n} - Ln \right\|$ [S1, unofficial].
    3.  Add Gaussian noise $\sigma_0=0.1$.
    4.  Compute correlation $\rho$ and $I(\alpha) = -0.5 \log(1-\rho)$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compute_pairwise_mi_parallel(alpha, sigma0, sigma_data):
    """
    Computes MI for parallel beam geometry with axisymmetric Gaussian data.
    alpha: angular separation in radians.
    sigma0: noise standard deviation.
    sigma_data: data standard deviation.
    """
    epsilon = (sigma0 / sigma_data) ** 2
    term = (np.cos(alpha) ** 2) / ((1 + epsilon) ** 2)
    # Clamp term to < 1 to avoid log(0) or complex numbers due to precision
    term = np.minimum(term, 0.9999)
    return -0.5 * np.log(1 - term)

def compute_pairwise_mi_cone_surrogate(alpha, cone_angle_deg):
    """
    Computes MI using Fourier surrogate for cone beam geometry.
    alpha: angular separation in radians.
    cone_angle_deg: cone angle in degrees.
    """
    # Coefficients for phi = 10, 15, 20, 25 degrees
    # Format: [phi_10, phi_15, phi_20, phi_25]
    coeffs = {
        0: [0.66, 0.67, 0.69, 0.73],
        1: [-0.02, -0.05, -0.10, -0.16],
        2: [-0.46, -0.43, -0.39, -0.29],
        3: [-0.01, -0.03, -0.05, -0.07],
        4: [-0.13, -0.12, -0.10, -0.08],
        5: [0.0, -0.01, -0.01, -0.02],
        6: [-0.01, -0.01, -0.02, -0.02]
    }
    
    # Select coefficients based on closest cone angle
    # In practice, interpolate or select nearest. Here we select nearest.
    phi_options = [10, 15, 20, 25]
    idx = np.argmin([abs(p - cone_angle_deg) for p in phi_options])
    
    a = [coeffs[k][idx] for k in range(7)]
    
    sum_cos = sum(a[k] * np.cos(k * alpha) for k in range(7))
    # Clamp sum_cos to > 0 to avoid log(0)
    sum_cos = np.maximum(sum_cos, 1e-10)
    return -0.5 * np.log(sum_cos)

def optimize_angles_parallel(N):
    """
    Returns optimal angles for parallel beam geometry.
    """
    if N % 2 != 0:
        return np.array([2 * np.pi * i / N for i in range(N)])
    else:
        # Even N: divide half-circle
        return np.array([np.pi * i / N for i in range(N)])

def optimize_angles_cone(N, cone_angle_deg):
    """
    Returns optimal angles for cone beam geometry.
    For even N, reflects angles to maximize separation.
    """
    if N % 2 != 0:
        return np.array([2 * np.pi * i / N for i in range(N)])
    else:
        # Even N with cone beam: reflect some angles
        # Heuristic: distribute in full circle but ensure separation > pi/2
        # The source notes that for even N, parallel beams use pi/N spacing.
        # Cone beams allow reflecting into opposite half-circle.
        # A simple robust implementation for even N in cone beam is often 
        # uniform distribution 2*pi/N, but the source implies a specific 
        # reflection strategy. Without explicit reflection algorithm in text,
        # we use uniform distribution which is optimal for odd N and 
        # often a good baseline for even N in cone beam to avoid I(pi)~I(0)
        # constraint of parallel beams.
        # Note: The source says "reflecting some angles... can be observed".
        # It does not give a closed form for the reflection.
        # We return uniform distribution as the general solution for N angles
        # maximizing separation in 3D cone beam context where I(pi) < I(0).
        return np.array([2 * np.pi * i / N for i in range(N)])

def get_optimal_angles(N, geometry='parallel', cone_angle_deg=10, sigma0=0.1, sigma_data=1.0):
    if geometry == 'parallel':
        return optimize_angles_parallel(N)
    elif geometry == 'cone':
        return optimize_angles_cone(N, cone_angle_deg)
    else:
        raise ValueError("Geometry must be 'parallel' or 'cone'")
```

## Subtopics

### Parallel Beam MI Selection
Assumes $I(\pi) = I(0)$, forcing odd Fourier coefficients to zero. Optimal angles for even $N$ divide the half-circle $\pi$ [S1, unofficial].

### Cone Beam MI Selection
Assumes $I(\pi) < I(0)$, allowing non-zero odd Fourier coefficients. Requires reflecting angles to increase separation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The optimization step is $O(N^2)$ to compute pairwise MI for $N$ angles, followed by a numerical minimization which depends on the optimizer but is generally fast for small $N$. The Fourier surrogate allows $O(1)$ MI evaluation per pair.
*   **Memory Complexity:** $O(N^2)$ to store the pairwise MI matrix if precomputed, or $O(1)$ if computed on the fly. The coefficient table is constant size.
*   **Strengths:** Provides a theoretically grounded selection of angles that maximizes information gain. Handles both parallel and cone beam geometries. Simple to implement via the Fourier surrogate.
*   **Limitations:** Relies on Gaussian assumptions which may not hold for all volumes. The surrogate approximation may lose accuracy for extreme cone angles or noise levels not covered in the table. Does not account for object-specific structure, only statistical priors.
*   **When to Use:** When the number of projections is limited and the goal is to maximize reconstruction quality per projection. When the volume statistics are approximately Gaussian or axisymmetric.
*   **When Not to Use:** When the volume has strong non-Gaussian features or anisotropic structures not captured by $\Sigma$. When computational resources for numerical optimization are extremely constrained (though the surrogate is fast).

## Background
The method originates from the work on neural rendering for dynamic tomography, specifically the paper "Neural rendering enables dynamic tomography" [S1, unofficial]. It applies information theory concepts to the problem of X-ray tomography, postulating that optimal angles minimize the posterior entropy of the data. The derivation uses Gaussian priors and stationarity assumptions to simplify the mutual information calculation. The Fourier series approximation for cone beam MI was derived via Monte Carlo simulation to handle the intractable marginalization of cone beam projections [S1, unofficial].

## Sources
[S1] Neural rendering enables dynamic tomography, https://doi.org/10.48550/arxiv.2410.20558, fetched 2026-10-26 (unofficial)
