# Hierarchical Sampling for Hemispherical Wavefronts

Also called: Coarse-to-Fine Hemispherical Sampling, NeTF Hierarchical Sampling

## Overview
This method is a coarse-to-fine sampling scheme for 2D hemispherical wavefronts used in Non-Line-of-Sight (NLOS) imaging to balance reconstruction quality and computational overhead [S1]. It addresses the intractability of explicit volumetric resampling on a hemisphere by using a differentiable hierarchical approach, distinct from the 1D ray sampling used in NeRF [S1].

## How it works
**Inputs:**
- Detection spot coordinates $(x', y')$ and time $t$ [S1].
- Coarse network parameters [S1].
- Fine network parameters [S1].
- Number of coarse samples $N_c^2$ [S1].
- Number of fine samples $N_f$ [S1].
- Radius $r$ of the hemispherical wavefront [S1].

**Outputs:**
- Estimated transient $\tau(x', y', t)$ [S1].

**Data Structures:**
- Coarse samples: Set of $N_c^2$ uniform points on the hemisphere, defined by spherical coordinates $(\theta, \phi)$ [S1].
- Fine samples: Set of $N_f$ points generated via Markov Chain Monte Carlo (MCMC), defined by $(\theta_{ij}^f, \phi_{ij}^f)$ [S1].
- PDF $K(\theta, \phi)$: Probability density function estimated by the coarse network for importance sampling [S1].

**Steps:**
1. **Coarse Sampling:** Sample $N_c^2$ uniform scene points in the hemisphere [S1].
2. **Coarse Evaluation:** Evaluate the coarse network at these points using the estimated PDF $k(\theta, \phi)$ to compute the coarse integral $\tau_c(x', y', t)$ [S1].
3. **Fine Sampling (MCMC):** Employ the Metropolis-Hastings algorithm with a conditional Gaussian distribution for state transition of the Markov chain to produce $N_f$ fine scene points along hemispherical wavefronts intersecting the NLOS scene [S1].
4. **Fine Evaluation:** Compute the fine integral $\tau_f(x', y', t)$ using the fine samples and the PDF $K(\theta_{ij}^f, \phi_{ij}^f)$ [S1].
5. **Combination:** Combine coarse and fine samples ($N_c^2 + N_f$) to re-evaluate reconstruction quality from the fine network, calculating the final transient as the average of the coarse and fine integrals [S1].

**Formulas:**
The final transient is the average of the coarse and fine estimates [S1]:
$$
\tau(x', y', t) = \frac{\tau_c(x', y', t) + \tau_f(x', y', t)}{2}
$$

The fine integral is estimated as [S1]:
$$
\tau_f(x', y', t) = \frac{1}{r^4} \sum_{i,j} \frac{\sigma(r, \theta_{ij}^f, \phi_{ij}^f) \rho(r, \theta_{ij}^f, \phi_{ij}^f)}{K(\theta_{ij}^f, \phi_{ij}^f)}
$$

Where:
- $\tau_c(x', y', t)$ is the integral estimated with coarse samples $N_c^2$ [S1].
- $\tau_f(x', y', t)$ is the transient estimated with $N_f$ samples from MCMC [S1].
- $K(\theta_{ij}^f, \phi_{ij}^f)$ is the PDF used for importance sampling in the MCMC fine sampling step [S1].
- $\sigma$ and $\rho$ are volume density and emission/reflectance properties evaluated at the fine sample points [S1].
- $r$ is the radius of the hemispherical wavefront [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def hierarchical_sampling_hemisphere(x_prime, y_prime, t, r, N_c, N_f, coarse_pdf, fine_density, fine_emission):
    """
    Perform coarse-to-fine hierarchical sampling on a hemispherical wavefront.
    
    Args:
        x_prime, y_prime: Detection spot coordinates.
        t: Time.
        r: Radius of the hemispherical wavefront.
        N_c: Integer, number of coarse samples is N_c^2.
        N_f: Integer, number of fine samples.
        coarse_pdf: Callable returning K(theta, phi).
        fine_density: Callable returning sigma(r, theta, phi).
        fine_emission: Callable returning rho(r, theta, phi).
        
    Returns:
        tau: Estimated transient value.
    """
    # Step 1: Coarse Sampling
    # Sample N_c^2 uniform points on the hemisphere [S1]
    coarse_points = []
    for _ in range(N_c * N_c):
        theta = math.sqrt(random.random()) * math.pi / 2
        phi = random.random() * 2 * math.pi
        coarse_points.append((theta, phi))

    # Step 2: Coarse Evaluation
    # Evaluate coarse network with estimated PDF k(theta, phi) [S1]
    # Compute tau_c (source does not specify exact aggregation formula for coarse integral) [S1]
    tau_c = 0.0
    for theta, phi in coarse_points:
        tau_c += coarse_pdf(theta, phi)
    tau_c /= len(coarse_points)

    # Step 3: Fine Sampling (MCMC)
    # Use Metropolis-Hastings with conditional Gaussian distribution for state transition [S1]
    fine_points = []
    current_theta, current_phi = coarse_points[0]
    for _ in range(N_f):
        # Gaussian proposal for state transition
        proposal_theta = current_theta + random.gauss(0, 0.1)
        proposal_phi = current_phi + random.gauss(0, 0.1)
        # Clamp to hemisphere bounds
        proposal_theta = max(0.0, min(math.pi / 2, proposal_theta))
        proposal_phi = proposal_phi % (2 * math.pi)

        # Compute acceptance ratio using PDF K from coarse network
        K_current = coarse_pdf(current_theta, current_phi)
        K_proposal = coarse_pdf(proposal_theta, proposal_phi)
        alpha = min(1.0, K_proposal / K_current)

        if random.random() < alpha:
            current_theta, current_phi = proposal_theta, proposal_phi

        fine_points.append((current_theta, current_phi))

    # Step 4: Fine Evaluation
    # Compute tau_f using importance sampling formula [S1]
    fine_samples_sum = 0.0
    for theta_f, phi_f in fine_points:
        sigma_val = fine_density(r, theta_f, phi_f)
        rho_val = fine_emission(r, theta_f, phi_f)
        K_val = coarse_pdf(theta_f, phi_f)
        # Guard against division by zero
        if K_val > 1e-8:
            fine_samples_sum += (sigma_val * rho_val) / K_val
        else:
            fine_samples_sum += 0.0

    tau_f = (1.0 / (r ** 4)) * fine_samples_sum

    # Step 5: Combination
    # Average of coarse and fine integrals [S1]
    tau = (tau_c + tau_f) / 2.0

    return tau
```

**Parameters:**
- $N_c$: Controls the number of coarse samples ($N_c^2$) [S1]. Typical values not given in the sources.
- $N_f$: Number of fine samples generated by MCMC [S1]. Typical values not given in the sources.
- $r$: Radius of the hemispherical wavefront [S1].

**Edge Cases and Pitfalls:**
- Explicit volumetric representations with fixed resolution make resampling on the hemisphere intractable for previous volume-based methods, unlike this differentiable hierarchical sampling [S1].
- Spherical wavefronts only intersect with a very small portion of the NLOS scene, converging at specific patches, while contributions from the rest are negligible [S1].

## Subtopics

### [Hexagonal Multisampling Pattern Derivation](3498-hexagonal-multisampling-pattern-derivation.md)
Derives structured multisampling patterns for different geometric configurations. Chosen when deterministic grid coverage is preferred over stochastic MCMC exploration.

### [DONeRF Depth Oracle Filtering](4472-donerf-depth-oracle-filtering.md)
Filters depth estimates to guide sampling in differentiable volume rendering. Chosen when explicit depth priors are available to constrain the search space.

## Operating characteristics
**Complexity:**
- Uniform sampling complexity is $O(N^2 L)$, where $L$ is the number of hemispherical wavefronts and $N^2$ is the number of scene points per wavefront [S1].
- The hierarchical scheme reduces overhead by focusing samples on regions where wavefronts intersect the NLOS scene [S1].

**Strengths and Limitations:**
- **Strengths:** Balances reconstruction quality and computational overhead; intrinsically differentiable; handles the 2D nature of hemispherical wavefronts unlike 1D ray sampling [S1].
- **Limitations:** Previous explicit volume-based methods cannot easily apply this due to intractability of resampling on the hemisphere [S1].

**When to Use:**
- Use when performing NLOS imaging with neural transient fields where computational efficiency is critical and differentiability is required [S1].
- Use when dealing with hemispherical wavefronts rather than linear rays [S1].

## Background
**Origin:**
- Introduced in the NeTF (Neural Transient Fields) paper [S1].
- NeTF uses an MLP structure analogous to NeRF but with ReLU activations and simplified final layers [S1].
- NeTF uses 2D sampling on a hemisphere, whereas NeRF uses 1D sampling along a ray [S1].

**History and Development:**
- The method was developed to address the high computational cost of uniform sampling in NLOS imaging, where denser samples lead to higher quality but also much higher overhead [S1].
- It leverages the observation that under confocal settings, spherical wavefronts intersect only a small portion of the NLOS scene [S1].

## Sources
[S1] Non-line-of-Sight Imaging via Neural Transient Fields, https://doi.org/10.1109/tpami.2021.3076062, fetched 2026-10-26
