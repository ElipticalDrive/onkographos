# Pseudo-marginal Metropolis Light Transport (PSSMLT)

Also called: PSSMLT with ratio tracking, Pseudo-marginal ERPT, Pseudo-marginal MCMC for rendering, Unbiased estimator-based MH

## Overview
Pseudo-marginal Metropolis Light Transport (PSSMLT) is a Monte Carlo method that enables Metropolis-Hastings sampling in scenes with heterogeneous participating media by replacing the intractable exact light path contribution function with a non-negative, unbiased estimator [S1]. Unlike methods that use biased approximations (such as deterministic ray-marching) which fail to target the correct density, PSSMLT constructs an extended target distribution in an auxiliary space to ensure the marginal distribution of sampled paths remains exact [S1]. It distinguishes itself from standard PSSMLT implementations that rely on delta tracking by utilizing ratio tracking, which is significantly faster and allows for longer Markov chains and better path space exploration [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Current light path $\bar{x}_i$ in path space.
*   Current auxiliary variable $u_i$ representing the random quantities used to estimate the contribution of $\bar{x}_i$.
*   Scene description including heterogeneous extinction coefficients $\sigma_t(x)$, scattering coefficients, and phase functions.
*   Proposal distribution $q(\bar{x}_{i+1} | \bar{x}_i)$ for perturbing light paths.
*   Parameter $M$: The number of independent ratio tracking samples used to compute the unbiased estimator (typically $M=1$) [S1].
*   Parameter $\gamma$: Scale parameter for medium vertex perturbation (typical values 50 and 100) [S1].

**Outputs:**
*   A sequence of light paths $\{\bar{x}_i\}_{i \ge 1}$ distributed according to the exact target density $\pi(\bar{x}) = L(\bar{x})/Z$ [S1].

### Data Structures
The state of the Markov chain is maintained as a pair $(\bar{x}_i, \hat{L}_i)$, where:
*   $\bar{x}_i$: The current light path, defined as a sequence of vertices $\{x_0, x_1, \dots, x_k\}$ and interaction types.
*   $\hat{L}_i$: A scalar floating-point value representing the unbiased estimate of the path contribution $L(\bar{x}_i)$, computed using $M$ independent ratio tracking samples [S1].

### Algorithm Steps

1.  **Initialization**: Generate an initial light path $\bar{x}_0$ using a standard path tracing technique. Compute the initial unbiased contribution estimate $\hat{L}_0$ using $M$ independent ratio tracking samples. Set $i=0$.

2.  **Proposal**: Given the current state $(\bar{x}_i, \hat{L}_i)$, propose a new light path $\bar{x}_{i+1}$ using the proposal distribution $q(\bar{x}_{i+1} | \bar{x}_i)$.
    *   *Vertex Perturbation in Media*: To update a medium vertex $x_j$, perturb it with uniform probability forward or backward along the current edge. The perturbation distance $d$ is drawn from a distribution proportional to $\gamma \sigma_t(x_j) \exp(-\gamma \sigma_t(x_j))$, where $\sigma_t(x_j)$ is the extinction coefficient at the vertex and $\gamma$ is a scale parameter [S1]. In practice, a mixture of perturbations with different $\gamma$ values (e.g., 50 and 100) is used [S1].
    *   *Bidirectional Mutation*: For bidirectional path mutation in media, use delta tracking to propose new vertices [S1].

3.  **Estimation**: Generate a new auxiliary variable $u_{i+1}$ by computing a new unbiased estimate $\hat{L}(\bar{x}_{i+1})$ independently of previous estimates. This involves generating $M$ independent ratio tracking samples to estimate the transmittance terms along the path segments [S1].

4.  **Acceptance**: Calculate the acceptance probability $\alpha$:
    $$
    \alpha = \min \left( 1, \frac{\hat{L}(\bar{x}_{i+1})}{\hat{L}(\bar{x}_i)} \cdot \frac{q(\bar{x}_i | \bar{x}_{i+1})}{q(\bar{x}_{i+1} | \bar{x}_i)} \right)
    $$
    Note: The terms related to the auxiliary variable distribution $g(u|\bar{x})$ cancel out in the ratio if the estimation process is consistent, or are implicitly handled by the independence of the new estimate [S1]. The text notes that proposal densities cancel if delta tracking is used with respect to the luminance of transmission [S1].

5.  **Update**:
    *   If $\alpha \ge \text{random}[0,1]$, accept the new path: set $\bar{x}_{i+1}$ as the current path and $\hat{L}_{i+1}$ as the current estimate.
    *   Otherwise, reject the proposal: keep $\bar{x}_i$ and $\hat{L}_i$.
    *   Increment $i$.

### Formulas

**Extended Target Distribution:**
The method relies on an extended target distribution $\phi(\bar{x}, u)$ in the joint space of paths and auxiliary variables:
$$
\phi(\bar{x}, u) = \frac{\hat{L}(\bar{x})}{Z} g(u | \bar{x})
$$
where $\hat{L}(\bar{x})$ is the unbiased estimator, $Z$ is the normalization constant, and $g(u | \bar{x})$ is the distribution of the random quantities used to generate the estimate [S1].

**Unbiased Estimator for Total Path Contribution:**
For a path $\bar{x}_j$ with vertices $x_0, \dots, x_k$, the estimator $\hat{L}_u(\bar{x}_j)$ is:
$$
\hat{L}_u(\bar{x}_j) = L_e \cdot \prod_{i=0}^{k-1} G(x_i, x_{i+1}) \cdot \hat{T}_i(x_i, x_{i+1}) \cdot \rho(x_i) \cdot \prod_{j=1}^{k-1} \hat{T}_j(x_j, x_{j+1})
$$
where:
*   $L_e$ is the emitted radiance.
*   $G(x_i, x_{i+1})$ is the geometric term.
*   $\rho(x_i)$ is the scattering density or albedo.
*   $\hat{T}_i(x_i, x_{i+1})$ are independent unbiased estimators of transmission obtained via ratio tracking [S1].

**Acceptance Probability (General Form):**
$$
\alpha = \min \left( 1, \frac{\hat{L}(\bar{x}_{i+1}) q(\bar{x}_i | \bar{x}_{i+1}) g(u_i | \bar{x}_i)}{\hat{L}(\bar{x}_i) q(\bar{x}_{i+1} | \bar{x}_i) g(u_{i+1} | \bar{x}_{i+1})} \cdot \frac{u_{i+1}}{u_i} \right)
$$
However, since $u$ represents the random seed for the estimator and the new estimate is generated independently, the standard implementation simplifies to the ratio of the estimates and proposal densities [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pseudo_marginal_mlt_step(current_path, current_estimate, scene, proposal_params):
    """
    Performs one step of the Pseudo-marginal Metropolis Light Transport algorithm.
    
    Args:
        current_path: The current light path x_i.
        current_estimate: The unbiased estimate L_hat(x_i) from the previous step.
        scene: Scene data including sigma_t, phase functions, etc.
        proposal_params: Parameters for path perturbation (gamma values, etc.).
        
    Returns:
        new_path: The accepted or rejected path (same as current if rejected).
        new_estimate: The unbiased estimate for the new_path.
    """
    
    # 1. Propose a new path
    proposed_path = propose_path(current_path, scene, proposal_params)
    
    # 2. Generate new unbiased estimate for the proposed path
    # This corresponds to generating the auxiliary variable u_{i+1}
    # using M independent ratio tracking samples.
    # M is typically 1 for complex scenes to allow longer chains.
    M = 1 
    new_estimate = compute_unbiased_contribution(proposed_path, scene, M)
    
    # 3. Compute acceptance probability
    # Alpha = min(1, (L_hat_new / L_hat_old) * (q_old / q_new))
    # Note: If delta tracking is used with respect to luminance of transmission,
    # the proposal densities q cancel out.
    
    # Calculate proposal ratio
    # In many implementations, q(x_i | x_{i+1}) / q(x_{i+1} | x_i) is 1 
    # or computed explicitly if the proposal is asymmetric.
    proposal_ratio = compute_proposal_ratio(current_path, proposed_path, scene, proposal_params)
    
    ratio = (new_estimate / current_estimate) * proposal_ratio
    
    # 4. Accept or Reject
    if random_uniform(0, 1) < min(1.0, ratio):
        return proposed_path, new_estimate
    else:
        return current_path, current_estimate

def compute_unbiased_contribution(path, scene, M):
    """
    Computes an unbiased estimator of the path contribution using ratio tracking.
    
    Args:
        path: Light path vertices.
        scene: Scene data.
        M: Number of independent ratio tracking samples.
        
    Returns:
        unbiased_estimate: Scalar value L_hat(path).
    """
    # Initialize contribution with emitted radiance
    contribution = path.emitted_radiance
    
    # Iterate over path segments
    for i in range(len(path.vertices) - 1):
        x_i = path.vertices[i]
        x_next = path.vertices[i+1]
        
        # Geometric term
        G = geometric_term(x_i, x_next)
        contribution *= G
        
        # Scattering term (albedo/density)
        rho = scattering_density(x_i)
        contribution *= rho
        
        # Transmission term using Ratio Tracking
        # T_hat is an unbiased estimator of transmittance T(x_i, x_next)
        # It is computed by averaging M independent ratio tracking samples
        T_hat = ratio_tracking_transmittance(x_i, x_next, scene, M)
        contribution *= T_hat
        
    return contribution

def ratio_tracking_transmittance(x_start, x_end, scene, M):
    """
    Computes an unbiased estimate of transmittance using ratio tracking.
    
    Args:
        x_start, x_end: Endpoints of the segment.
        scene: Scene data.
        M: Number of samples.
        
    Returns:
        unbiased_T: Unbiased estimate of transmittance.
    """
    # Ratio tracking involves sampling interaction points and weighting them
    # to estimate the integral of exp(-int sigma_t ds).
    # This function encapsulates the logic from Novak et al. [2014].
    
    total_weight = 0.0
    for _ in range(M):
        # Sample interaction points along the segment
        # Compute weights based on sigma_t and sampling distribution
        weight = compute_ratio_tracking_weight(x_start, x_end, scene)
        total_weight += weight
        
    return total_weight / M

def propose_path(current_path, scene, params):
    """
    Proposes a new path by perturbing the current path.
    
    Args:
        current_path: Current light path.
        scene: Scene data.
        params: Perturbation parameters (gamma values).
        
    Returns:
        proposed_path: New light path.
    """
    # Copy current path
    new_path = copy_path(current_path)
    
    # Select a vertex to perturb (e.g., a medium vertex)
    vertex_index = select_medium_vertex(new_path)
    vertex = new_path.vertices[vertex_index]
    
    # Determine perturbation direction and distance
    # Distance proportional to gamma * sigma_t * exp(-gamma * sigma_t)
    sigma_t = extinction_coefficient(vertex, scene)
    
    # Mix of gamma values (e.g., 50 and 100)
    gamma = random_choice([50, 100])
    
    # Sample distance d
    # Note: Exact sampling from the distribution proportional to 
    # gamma * sigma_t * exp(-gamma * sigma_t * d) is required.
    d = sample_perturbation_distance(gamma, sigma_t)
    
    # Perturb vertex forward or backward
    direction = random_choice([-1, 1])
    new_vertex = vertex + direction * d * direction_vector(current_path, vertex_index)
    
    # Update path with new vertex
    new_path.vertices[vertex_index] = new_vertex
    
    # Re-evaluate path segments connected to the perturbed vertex
    # Ensure path validity (e.g., visibility)
    # If delta tracking is used for bidirectional mutation, 
    # new vertices are proposed via delta tracking.
    
    return new_path

def compute_proposal_ratio(old_path, new_path, scene, params):
    """
    Computes the ratio q(old | new) / q(new | old).
    
    Note: If delta tracking is used with respect to luminance of transmission,
    this ratio is 1. Otherwise, it must be computed explicitly.
    """
    # Implementation depends on the specific proposal strategy.
    # For simple vertex perturbations with symmetric distance distributions,
    # the ratio might be 1.
    return 1.0
```

### Parameters
*   **$M$**: The number of averaged ratio trackings used to estimate the transmission. A value of $M=1$ is recommended for complex scenes to allow longer Markov chains [S1].
*   **$\gamma$**: A scale parameter for medium vertex perturbation. Typical values used are 50 and 100, mixed with uniform probability [S1].

### Edge Cases and Pitfalls
*   **Poor Mixing**: If $M$ is too low, the mixing of the marginal subchain can be poor. Overestimation of the exact quantity can cause the chain to get stuck at the current light path, leading to long series of rejected samples [S1]. A rule of thumb is to choose $M$ so that the variance of $\log(\hat{L}(\bar{x}))$ is approximately one, though this assumption often does not hold in practice [S1].
*   **Bias**: Using a biased estimate of the transmission (e.g., deterministic ray-marching) causes the resulting Markov chain to not necessarily target the desired density $\pi(\bar{x})$ [S1].
*   **Precision**: The unbiased estimator must be strictly non-negative. Negative values would break the pseudo-marginal framework [S1].

## Subtopics

### PSSMLT with Ratio Tracking
*   **What it is**: An implementation of PSSMLT that uses ratio tracking [Novák et al. 2014] to generate unbiased estimates of transmittance in heterogeneous media.
*   **Difference**: It is significantly faster than PSSMLT with delta tracking [Raab et al. 2008] and produces less image noise than ERPT with Simpson quadrature ray-marching by enabling longer Markov chains.
*   **When to choose**: Use when rendering heterogeneous participating media and efficiency is critical. It is preferred over delta tracking for speed and over ray-marching for unbiasedness.
*   **Link**: [PSSMLT with Ratio Tracking](52-pssmlt-ratio-tracking.md)

### PSSMLT with Delta Tracking
*   **What it is**: An earlier implementation of PSSMLT that uses delta tracking to estimate transmittance.
*   **Difference**: Slower than ratio tracking due to the higher number of samples required for delta tracking in heterogeneous media.
*   **When to choose**: Use only if ratio tracking is not applicable or for comparison purposes.
*   **Link**: [PSSMLT with Delta Tracking](52-pssmlt-delta-tracking.md)

### Pseudo-marginal ERPT
*   **What it is**: The application of the pseudo-marginal framework to Energy Redistribution Path Tracing (ERPT).
*   **Difference**: ERPT allows for more efficient local path perturbations compared to PSSMLT, leading to better exploration of path space in some scenes.
*   **When to choose**: Use when local perturbations are effective for the scene's light paths.
*   **Link**: [Pseudo-marginal ERPT](52-pseudo-marginal-erpt.md)

## Operating characteristics

*   **Time Complexity**: The cost per iteration is dominated by the computation of the unbiased estimator $\hat{L}(\bar{x})$. Ratio tracking is significantly faster than ray-marching, allowing for more iterations per unit time [S1].
*   **Memory Complexity**: Low. Only the current path and its estimate need to be stored.
*   **Measured Performance**: In heterogeneous media scenes (e.g., glass cube with isotropic media, anisotropic media in Veach's scene), PSSMLT with ratio tracking produces less noise than ERPT with ray-marching and performs slightly better than PSSMLT with delta tracking [S1].
*   **Scaling**: Performance scales with the complexity of the media. The method is particularly effective in scenes where traditional path tracing produces very noisy results due to complex light paths [S1].
*   **Strengths**: Unbiased estimation ensures the correct target distribution. Ratio tracking provides computational efficiency.
*   **Limitations**: PSSMLT (as opposed to ERPT) lacks the ability to use efficient local path perturbations, which can limit exploration in some scenes [S1]. The mixing of the chain is sensitive to the choice of $M$ [S1].
*   **When to use**: Rendering scenes with heterogeneous participating media where unbiasedness is required and local perturbations are not sufficient or applicable.
*   **When not to use**: When biased approximations are acceptable and faster convergence is needed, or when the scene is homogeneous and standard path tracing is sufficient.

## Background

*   **Origin**: The method was introduced by Joel Kronander, Thomas B. Schön, and Jonas Unger in 2015 [S1].
*   **History**: It builds on the pseudo-marginal MCMC methods developed for Bayesian inference with intractable likelihoods by Beaumont [2003] and Andrieu and Roberts [2009] [S1]. Previous MCMC methods for participating media, such as those by Kelemen et al. [2002] (PSSMLT) and Raab et al. [2008], used biased approximations (ray-marching) or less efficient estimators (delta tracking) [S1].
*   **Usage**: Implemented in the Mitsuba renderer [Jakob 2010] [S1].

## Sources

[S1] Pseudo-marginal metropolis light transport, https://doi.org/10.1145/2820903.2820922, fetched 2026-10-30
