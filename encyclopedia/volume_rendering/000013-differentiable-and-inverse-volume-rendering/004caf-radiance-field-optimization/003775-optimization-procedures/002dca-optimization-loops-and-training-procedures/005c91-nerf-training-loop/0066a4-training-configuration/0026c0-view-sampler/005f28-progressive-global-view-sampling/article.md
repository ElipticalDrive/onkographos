# Progressive Global View Sampling

Also called: Moving beta distribution, Probabilistic Gradient Vector Sampling

## Overview
Progressive Global View Sampling (PGVS) is a camera view sampling strategy for 3D object reconstruction that progressively shifts the sampling distribution from in-boundary (IB) concentration to a uniform distribution over training iterations using a Beta distribution [S1, unofficial]. It sets itself apart from static view samplers by dynamically adjusting the probability density of camera angles to prioritize frontal views early in training and back views later, ensuring comprehensive coverage without requiring pre-defined view lists [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Initial Beta distribution parameters $\alpha_0, \beta_0$; total training iterations $T$; current iteration $t$; range of camera angles $\text{range}(\theta, \phi)$; set of pre-rendered positions $\mathcal{P}$; in-boundary region definition.
*   **Outputs:** Sampled camera angles $(\theta, \phi)$; selected camera position from $\mathcal{P}$ or a new random position.

**Data Structures**
*   **Beta Distribution Parameters:** Two floating-point values $\alpha(t)$ and $\beta(t)$ representing the shape parameters of the Beta distribution at iteration $t$.
*   **Pre-rendered Positions:** A list or set of 3D coordinates $\mathcal{P} = \{p_1, p_2, ..., p_n\}$ corresponding to previously rendered views.

**Formulas**
The decay of the Beta distribution parameters is linear with respect to a normalized time factor $u_0 t$:
$$ \alpha(t) = \alpha_0 + (1 - \alpha_0)u_0 t $$
$$ \beta(t) = \beta_0 + (1 - \beta_0)u_0 t $$
where $t$ is the current iteration step and $u_0$ is a scaling factor [S1, unofficial].

The **uniform point** $t_u$ is defined as the iteration where both $\alpha(t)$ and $\beta(t)$ decay to 1. At this point, the Beta distribution becomes uniform [S1, unofficial].

For iterations $t < t_u$, camera angles $\theta, \phi$ are sampled from the corresponding Probability Density Function (PDF) $f(x; t)$ using inverse transform sampling:
$$ \theta, \phi = \left( \int f(x; t)dx \right)^{-1} (U_{\theta,\phi}(0, 1)) \cdot \text{range}(\theta, \phi) $$
where $U_{\theta,\phi}(0, 1)$ is a uniform random variable in $[0, 1]$ [S1, unofficial].

**Steps**
1.  **Initialize:** Set initial parameters $\alpha_0, \beta_0$. Determine the uniform point $t_u$ where $\alpha(t_u) = 1$ and $\beta(t_u) = 1$.
2.  **Check Iteration:** If current iteration $t \ge t_u$, proceed to Step 4. Otherwise, proceed to Step 3.
3.  **Progressive Sampling:**
    *   Calculate current $\alpha(t)$ and $\beta(t)$ using the decay formulas.
    *   Sample raw angles from the Beta distribution defined by $\alpha(t)$ and $\beta(t)$.
    *   Map these angles to the desired range $\text{range}(\theta, \phi)$ using the inverse CDF method described in the formula.
    *   *Note:* In the initial phase, this concentrates samples in the frontal part; as $t$ increases, samples shift toward the back part [S1, unofficial].
4.  **Uniform Sampling:** If $t \ge t_u$, sample $\theta, \phi$ uniformly from $\text{range}(\theta, \phi)$.
5.  **Boundary Handling:**
    *   Check if the sampled camera position lies within the in-boundary (IB) region.
    *   If yes, select the nearest pre-rendered position from $\mathcal{P}$ to avoid unnecessary computational burden [S1, unofficial].
    *   If no, use the newly sampled position.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_global_view_sampling(t, T, alpha_0, beta_0, range_theta, range_phi, pre_rendered_positions):
    """
    t: current iteration
    T: total iterations (used to normalize t if u_0 is defined as 1/T)
    alpha_0, beta_0: initial beta distribution parameters
    range_theta, range_phi: tuples (min, max) for angles
    pre_rendered_positions: list of 3D points
    """
    # Determine uniform point t_u where alpha and beta become 1
    # alpha(t) = alpha_0 + (1 - alpha_0) * u_0 * t = 1
    # => u_0 * t = (1 - alpha_0) / (1 - alpha_0) = 1
    # Assuming u_0 = 1/T for normalization, t_u = T / (1 - alpha_0) ? 
    # The source defines u_0 t as the factor. Let's assume u_0 is a constant 
    # or 1/T. The source says "u_0 t" is the term. 
    # If alpha(t) = 1, then alpha_0 + (1-alpha_0)u_0 t_u = 1
    # (1-alpha_0)u_0 t_u = 1 - alpha_0
    # u_0 t_u = 1
    # So t_u = 1 / u_0.
    
    # For implementation, we calculate current alpha and beta
    # u_0 is typically 1/T if t is normalized 0..1, or just a constant.
    # Let's assume t is normalized [0,1] and u_0=1 for simplicity based on "u_0 t"
    # If t is integer iteration, u_0 might be 1/T.
    
    # Let's define u_0 = 1.0 / T if t is integer, or just use t if normalized.
    # The source formula: alpha(t) = alpha_0 + (1 - alpha_0)u_0 t
    # If t goes from 0 to T, and we want alpha(T) = 1, then u_0 must be 1/T?
    # alpha(T) = alpha_0 + (1-alpha_0)u_0 T = 1 => u_0 T = 1 => u_0 = 1/T.
    
    u_0 = 1.0 / T
    alpha_t = alpha_0 + (1 - alpha_0) * u_0 * t
    beta_t = beta_0 + (1 - beta_0) * u_0 * t
    
    # Check if we have reached the uniform point
    # Uniform point is when alpha=1 and beta=1.
    # This happens when u_0 * t = 1 => t = T.
    # So for t < T, we use Beta distribution. For t >= T, uniform.
    
    if t >= T:
        theta = uniform_sample(range_theta)
        phi = uniform_sample(range_phi)
    else:
        # Sample from Beta distribution with alpha_t, beta_t
        # Map to range
        theta_raw = sample_beta(alpha_t, beta_t)
        phi_raw = sample_beta(alpha_t, beta_t) # Assuming independent or joint? Source implies angles from PDF.
        
        # Inverse CDF mapping
        theta = cdf_inverse_beta(theta_raw, alpha_t, beta_t) # This is redundant if sample_beta returns uniform variate mapped
        # Actually, the formula uses U(0,1) and inverse CDF.
        # So:
        u_theta = random.uniform(0, 1)
        u_phi = random.uniform(0, 1)
        
        # Inverse CDF of Beta(alpha_t, beta_t)
        theta = beta_ppf(u_theta, alpha_t, beta_t) * range_theta[1] + range_theta[0]
        phi = beta_ppf(u_phi, alpha_t, beta_t) * range_phi[1] + range_phi[0]
        
    # Boundary handling
    sampled_pos = get_camera_position(theta, phi)
    if is_in_boundary(sampled_pos):
        nearest_pos = find_nearest(sampled_pos, pre_rendered_positions)
        return nearest_pos
    else:
        return sampled_pos
```

**Parameters**
*   $\alpha_0, \beta_0$: Initial shape parameters. Typical values are not explicitly given in the sources, but they must be $>1$ to ensure the decay reaches 1 [S1, unofficial].
*   $u_0$: Scaling factor for time. Implicitly $1/T$ if $t$ is integer iteration count [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Precision:** If $\alpha_0$ or $\beta_0$ are close to 1, the uniform point $t_u$ is reached very early, reducing the benefit of progressive sampling.
*   **IB Region Definition:** The "in-boundary" region must be well-defined to avoid selecting incorrect nearest neighbors [S1, unofficial].
*   **Degenerate Input:** If $\alpha_0 = 1$ and $\beta_0 = 1$, the distribution is uniform from the start, and no progressive sampling occurs.

## Subtopics

### Moving Beta Distribution
A variant where $\alpha$ and $\beta$ gradually swap to each other's initial values, shifting focus from front to back differently than standard PGVS [S1, unofficial].

### Discrete Accumulation Distribution
Used in conjunction with PGVS for optimizing camera placement, though specific details are not provided in the sources [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Sampling is $O(1)$ per view, assuming inverse CDF lookup or approximation is constant time. The boundary check is $O(N)$ or $O(\log N)$ depending on the spatial index used for $\mathcal{P}$ [S1, unofficial].
*   **Memory Complexity:** $O(N)$ to store pre-rendered positions $\mathcal{P}$ [S1, unofficial].
*   **Strengths:** Reduces computational burden by reusing pre-rendered views in the IB region; ensures comprehensive coverage of the object by progressively shifting sampling focus [S1, unofficial].
*   **Limitations:** Requires pre-rendered positions for the IB region; performance depends on the quality of the IB/OB boundary definition [S1, unofficial].
*   **When to Use:** When reconstructing 3D objects from text or image prompts where initial views are biased (e.g., frontal) and comprehensive coverage is needed [S1, unofficial].

## Background
PGVS was introduced in the DITTO-NeRF paper, which focuses on diffusion-based iterative text-to-omni-directional 3D model generation [S1, unofficial]. The method addresses the issue of extraneous points and inconsistent reconstruction by optimizing camera placement using a probabilistic gradient vector sampling approach [S1, unofficial].

## Sources
[S1] DITTO-NeRF: Diffusion-based Iterative Text To Omni-directional 3D Model, https://arxiv.org/abs/2304.02827v1, fetched 2026-10-26 (unofficial)
