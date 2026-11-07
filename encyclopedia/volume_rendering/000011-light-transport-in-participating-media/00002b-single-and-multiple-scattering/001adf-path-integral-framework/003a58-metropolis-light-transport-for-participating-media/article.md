# Metropolis Light Transport for Participating Media

Also called: MLT for Participating Media, Metropolis Light Transport, Metropolis Sampling for Light Transport

## Overview
Metropolis Light Transport (MLT) for participating media is a rendering algorithm that uses Metropolis-Hastings sampling to generate light transport paths directly according to the measurement contribution function, rather than using importance sampling which only considers factors of it [S2]. It builds on the [Path Integral Framework](873-path-integral-framework.md) by treating the space of light paths as a high-dimensional domain where the target distribution is proportional to the contribution of each path to the final image. Unlike [Surrounding Path Contribution (WKB Expansion)](1300-surrounding-path-contribution-wkb-expansion.md) which relies on perturbative expansions, or [Null-Scattering Path Integral Framework](3070-null-scattering-path-integral-framework.md) which uses null-collision absorption to handle inhomogeneous media, MLT explores path space via a Markov chain, allowing it to efficiently sample complex, high-order scattering events and highly anisotropic media that cause standard path tracing to converge slowly [S1, S2].

## How it works
**Inputs:**
- Scene description (geometry, material properties, participating media extinction/scattering coefficients).
- Camera parameters.
- A base path tracer (or any suitable rendering algorithm) capable of generating initial path samples and estimating the normalization constant $b$ [S2].
- A mutation operator (perturbation strategy) for the path space [S2].

**Outputs:**
- An unbiased estimate of the pixel values (radiance) for the image.

**Data Structures:**
- **Path State ($\bar{x}$):** Represents a complete light transport path. In the context of the path integral framework, this is a point in the high-dimensional path space $\mathcal{P}$ [S2].
- **Contribution Function ($f(\bar{x})$):** The measurement contribution function, representing the unnormalized weight of path $\bar{x}$ [S2].
- **Normalization Constant ($b$):** The integral of the contribution function over all paths, $b = \int_{\mathcal{P}} f(\bar{y}) d\mu(\bar{y})$ [S2].
- **Tentative Transition Function ($T(\bar{x} \rightarrow \bar{y})$):** The probability density of proposing state $\bar{y}$ given current state $\bar{x}$ [S2].

**Parameters:**
- **$N$:** Number of samples in the Metropolis chain.
- **$b_{est}$:** Estimated value of the normalization constant $b$. Typically estimated using a low number of samples ($10^4$–$10^5$) from a standard rendering algorithm [S2].
- **Mutation Interval:** Frequency at which independent paths are generated to ensure ergodicity.

**Steps:**

1.  **Initialization:**
    Use the base path tracer to generate a set of approximation samples. Select one sample $\bar{x}_0$ as the initial state for the Metropolis phase. The selection probability for a sample $\bar{x}$ is proportional to $f(\bar{x})/p(\bar{x})$, where $p(\bar{x})$ is the probability density of generating $\bar{x}$ with the base tracer [S2]. This ensures the initial state is distributed according to the stationary density, avoiding start-up bias without discarding samples [S2].

2.  **Estimate Normalization Constant:**
    Estimate $b$ using the base path tracer with a low number of samples (typically $10^4$–$10^5$). This cost is considered negligible compared to the main MLT phase [S2].

3.  **Mutation Loop:**
    For each step $i$ from 1 to $N$:
    a.  **Choose Mutation Type:** Randomly select a mutation operator from the available set. The mutation must be chosen such that it can exploit coherence in the scene and maintain low correlation between samples [S2].
    b.  **Generate Proposal:** Generate a tentative new state $\bar{y}$ from the current state $\bar{x}_{i-1}$ using the tentative transition function $T(\bar{x}_{i-1} \rightarrow \bar{y})$ [S2].
        *   *Note on Adaptive Mutation:* A common strategy (Kelemen et al.) perturbs the primary sample point corresponding to the current path by a small exponentially distributed amount. This perturbation is symmetric, simplifying the acceptance probability [S2].
    c.  **Calculate Acceptance Probability:** Compute the acceptance probability $a(\bar{x}_{i-1} \rightarrow \bar{y})$:
        $$
        a(\bar{x}_{i-1} \rightarrow \bar{y}) := \min \left\{ 1, \frac{f(\bar{y})T(\bar{y} \rightarrow \bar{x}_{i-1})}{f(\bar{x}_{i-1})T(\bar{x}_{i-1} \rightarrow \bar{y})} \right\}
        $$
        If the perturbation is symmetric (i.e., $T(\bar{x} \rightarrow \bar{y}) = T(\bar{y} \rightarrow \bar{x})$), this simplifies to:
        $$
        a(\bar{x} \rightarrow \bar{y}) = \min \left\{ 1, \frac{f(\bar{y})}{f(\bar{x})} \right\}
        $$
        [S2].
    d.  **Accept or Reject:** Generate a uniform random number $u \in [0, 1]$. If $u < a(\bar{x}_{i-1} \rightarrow \bar{y})$, accept the proposal: $\bar{x}_i = \bar{y}$. Otherwise, reject it: $\bar{x}_i = \bar{x}_{i-1}$ [S2].
    e.  **Accumulate Contribution:** Add the contribution of $\bar{x}_i$ to the pixel buffers. The estimator for pixel $j$ is:
        $$
        F_{j,N} = \frac{1}{N} \sum_{i=1}^N h_j(\bar{X}_i) b
        $$
        where $h_j(\bar{x})$ is the pixel filter function for pixel $j$, and $f(\bar{x})$ is the remainder of the contribution function common to every pixel [S2].
    f.  **Ergodicity Check:** At random intervals, generate a fresh independent path using the base path tracer (feeding a fresh set of uniform random numbers) and use it as the new current state $\bar{x}_i$. This prevents the Markov chain from getting stuck in isolated areas of the path space [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def metropolis_light_transport(scene, camera, num_mlt_steps, base_tracer, mutation_operators):
    """
    Implements Metropolis Light Transport for participating media.
    
    Args:
        scene: Scene description (geometry, media, etc.)
        camera: Camera parameters
        num_mlt_steps: Number of Metropolis chain steps per pixel (or global)
        base_tracer: Function that generates initial paths and estimates b
        mutation_operators: List of mutation functions [mutate_path(path, rng)]
        
    Returns:
        image_estimate: Rendered image buffer
    """
    
    # Step 1: Estimate normalization constant b
    # Use base_tracer with low sample count (e.g., 10^4 - 10^5)
    b_estimate = base_tracer.estimate_normalization(scene, camera, num_samples=10000)
    
    # Step 2: Initialization
    # Generate initial path samples using base_tracer
    # Select initial state x_0 with probability proportional to f(x)/p(x)
    # This avoids start-up bias
    initial_paths = base_tracer.generate_paths(scene, camera, num_samples=1000)
    x_current = select_initial_state(initial_paths, base_tracer.density)
    
    # Initialize image buffer
    image_estimate = zeros_like(camera.resolution)
    
    # Step 3: Mutation Loop
    for i in range(num_mlt_steps):
        # Choose mutation type randomly
        mutation_op = random.choice(mutation_operators)
        
        # Generate tentative sample y from x_current
        y_tentative = mutation_op(x_current, rng=random_number_generator())
        
        # Calculate acceptance probability
        # f(x) is the contribution function value for path x
        f_current = scene.evaluate_contribution(x_current)
        f_tentative = scene.evaluate_contribution(y_tentative)
        
        # T(x->y) is the transition probability
        # For symmetric perturbations (e.g., exponential perturbation of primary sample),
        # T(x->y) = T(y->x), so they cancel out.
        # If non-symmetric, compute T(x->y) and T(y->x) explicitly.
        t_current_to_tentative = mutation_op.transition_probability(x_current, y_tentative)
        t_tentative_to_current = mutation_op.transition_probability(y_tentative, x_current)
        
        if t_tentative_to_current > 0 and t_current_to_tentative > 0:
            acceptance_ratio = (f_tentative * t_tentative_to_current) / (f_current * t_current_to_tentative)
        else:
            acceptance_ratio = 0.0
            
        acceptance_prob = min(1.0, acceptance_ratio)
        
        # Accept or reject
        u = random.uniform(0, 1)
        if u < acceptance_prob:
            x_current = y_tentative
        # else: x_current remains unchanged
        
        # Accumulate contribution
        # Split f(x) into h_j(x) * f_common(x)
        # F_j = (1/N) * sum(h_j(X_i) * b)
        for pixel_j in camera.pixels:
            h_j = pixel_j.filter_function(x_current)
            image_estimate[pixel_j] += h_j * b_estimate
            
        # Ergodicity: Generate fresh independent path at random intervals
        # This prevents getting stuck in isolated areas
        if random.random() < ergodicity_probability: # e.g., 1 in 100 steps
            fresh_path = base_tracer.generate_single_path(scene, camera, rng=random_number_generator())
            x_current = fresh_path
            
    # Normalize by number of steps
    image_estimate /= num_mlt_steps
    
    return image_estimate
```

**Edge Cases and Pitfalls:**
- **Ergodicity:** The mutation operator alone cannot ensure ergodicity. If the chain gets stuck in a local region of the path space (e.g., a specific scattering event), it will fail to explore other valid paths. Independent paths must be generated at random intervals by the underlying path tracer to guarantee the algorithm explores the entire space [S2].
- **Symmetric Perturbations:** If the mutation operator is not symmetric (i.e., $T(\bar{x} \rightarrow \bar{y}) \neq T(\bar{y} \rightarrow \bar{x})$), the acceptance probability must include the ratio of transition probabilities. Failure to do so introduces bias [S2].
- **Normalization Constant:** The accuracy of the final estimate depends on the accuracy of $b$. While $10^4$–$10^5$ samples are usually sufficient, highly complex scenes may require more samples for a stable estimate [S2].

## Subtopics

### Adaptive Mutation
A specific mutation strategy proposed by Kelemen et al. that perturbs the primary sample point corresponding to the current path by a small exponentially distributed amount. This is symmetric, simplifying the acceptance probability. It is chosen for its ability to exploit coherence in the scene [S2].

### Ergodicity Restoration
The technique of periodically injecting fresh, independent paths generated by a base path tracer. This is essential for MLT to avoid getting stuck in isolated areas of the path space [S2].

## Operating characteristics
- **Time Complexity:** The cost per Metropolis step is dominated by the evaluation of the contribution function $f(\bar{x})$ and the mutation operation. The total time is proportional to $N \times (\text{cost of } f(\bar{x}) + \text{cost of mutation})$. The initialization cost is negligible if $b$ is estimated with $10^4$–$10^5$ samples [S2].
- **Memory Complexity:** Low, as only the current path state and a few auxiliary variables need to be stored. The base tracer's memory usage is separate and typically low for the initialization phase [S2].
- **Measured Performance:** MLT is particularly effective for scenes with high-order scattering or highly anisotropic participating media, where standard path tracing converges slowly [S1]. It can achieve better convergence rates in these scenarios by exploring the path space more efficiently [S1, S2].
- **Scaling:** Performance scales linearly with the number of Metropolis steps $N$. The quality of the estimate improves as $N$ increases, but the rate of convergence depends on the correlation between samples, which is influenced by the mutation strategy [S2].
- **Strengths:**
    - Can sample complex, high-order scattering events efficiently.
    - Exploits coherence in the scene through local perturbations.
    - Unbiased estimator.
- **Limitations:**
    - Requires a good mutation strategy to be effective.
    - Can get stuck in local regions if ergodicity is not maintained.
    - Convergence can be slow if the correlation between samples is high.
- **When to Use:** Use MLT for participating media when the scene contains complex scattering effects, high-order scattering, or highly anisotropic media that are difficult to sample with standard path tracing [S1, S2].
- **When Not to Use:** Avoid MLT for simple scenes with low-order scattering or isotropic media, where standard path tracing or other methods may be more efficient. Also avoid if a good mutation strategy is not available or if ergodicity cannot be guaranteed [S2].

## Background
Metropolis Light Transport (MLT) was first presented by Veach and Guibas in 1997 [VG97] and strongly modified by Kelemen et al. in 2002 [KSKAC02]. The algorithm uses the Metropolis-Hastings sampling method [MRR+53] to sample the path space. Pauly et al. proposed the MLT approach specifically for participating media, addressing the difficulties of sampling light transport in media with unidirectional path tracing [S1]. The method has been further optimized in [Vea97], although these optimizations are not detailed in the provided sources [S2]. MLT is used in global illumination rendering, particularly for scenes with complex light transport phenomena such as caustics, subsurface scattering, and participating media [S1, S2].

## Sources
[S1] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
[S2] Unbiased Global Illumination with Participating Media, http://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/ugiwpm.pdf, fetched 2026-10-30
