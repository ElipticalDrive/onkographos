# Extended Multiple Importance Sampling

Also called: MIS for different dimensional spaces

## Overview
Extended Multiple Importance Sampling (Extended MIS) is a framework for combining estimators of integrals defined on spaces of different dimensions, such as standard path sampling and extended-space density estimators (e.g., photon mapping or beam tracing) [S1]. It solves the problem of variance reduction in volumetric path tracing when mixing unbiased path contributions with biased density estimates that operate on higher-dimensional spaces [S1]. Unlike standard MIS, which requires all strategies to sample from the same domain, Extended MIS derives a balance heuristic that accounts for the additional integration over the kernel support inherent in extended-space techniques [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $n$ sampling techniques; $u$ regular techniques ($1 \leq i \leq u$) sampling from domain $\mathcal{D}_x$; $n-u$ extended techniques ($u < i \leq n$) sampling from $\mathcal{D}_x \times \mathcal{D}_{y_i}$; $n_i$ samples per technique $i$.
*   **Outputs:** Combined estimator value $F^C$ and weights $\tilde{w}_i$.

**Data Structures:**
*   **Technique Index:** Integer $i \in [1, n]$.
*   **Sample Index:** Integer $j \in [1, n_i]$.
*   **Regular Sample:** $X_{i,j} \in \mathcal{D}_x$.
*   **Extended Sample:** $(X_{i,j}, Y_{i,j}) \in \mathcal{D}_x \times \mathcal{D}_{y_i}$.
*   **Contribution Function:** $f_i(x)$ for regular; $f_i(x, y_i)$ for extended.
*   **PDF:** $p_i(x)$ for regular; $p_i(x, y_i)$ for extended.
*   **Kernel:** $K_i(x, y_i)$ for extended techniques.
*   **Helper Function:** $h_i(x, y_i) = f_i(x, y_i) / K_i(x, y_i)$ [S1].

**Numbered Steps:**

1.  **Define the Extended Combined Estimator:**
    Compute the combined estimator $F^C$ as the sum of contributions from all techniques and samples [S1]:
    $$
    F^C = \sum_{i=1}^n \frac{1}{n_i} \sum_{j=1}^{n_i} F_{i,j}^C
    $$
    where the term $F_{i,j}^C$ is defined as [S1]:
    $$
    F_{i,j}^C = w_i(X_{i,j}) \frac{f_i(X_{i,j}, Y_{i,j})}{p_i(X_{i,j}, Y_{i,j})}
    $$
    For regular techniques ($1 \leq i \leq u$), $Y_{i,j}$ is undefined/ignored, and $f_i, p_i$ depend only on $X_{i,j}$. For extended techniques ($u < i \leq n$), the term uses the extended space formulation [S1].

2.  **Compute Theoretical Weights (Extended Balance Heuristic):**
    Determine the optimal weight $\tilde{w}_i(x)$ for a point $x$ by minimizing variance [S1]:
    $$
    \tilde{w}_i(x) = \frac{n_i / \kappa_i(x)}{\sum_{k=1}^n n_k / \kappa_k(x)}
    $$
    where $\kappa_i(x)$ is defined as [S1]:
    $$
    \kappa_i(x) = \begin{cases} 
    \frac{f_i^2(x)}{p_i(x)} & \text{if } 1 \leq i \leq u \\
    \int_{\mathcal{D}_{y_i}} \frac{f_i^2(x, y_i)}{p_i(x, y_i)} dy_i & \text{if } u < i \leq n
    \end{cases}
    $$
    For extended techniques, $\kappa_i(x)$ includes an integral over the kernel support $\mathcal{D}_{y_i}$ [S1].

3.  **Approximate $\kappa_i(x)$ for Extended Techniques:**
    The integral in $\kappa_i(x)$ is impractical to evaluate exactly. Assume $h_i(x, y_i)$ and $p_i(x, y_i)$ are constant within the kernel support $K_i$ [S1]. Let $y_i^*$ be a representative sample (e.g., the actual photon location) [S1]. The approximation is [S1]:
    $$
    \kappa_i(x) \approx \frac{h_i^2(x, y_i^*)}{p_i(x, y_i^*)} \int_{\mathcal{D}_{y_i}} K_i^2(x, y_i) dy_i
    $$
    Substitute $h_i(x, y_i^*) = f_i(x, y_i^*) / K_i(x, y_i^*)$ [S1].

4.  **Simplify Weights for Implementation:**
    Assume $h_i(x, y_i^*) \approx f(x)$, where $f(x)$ is the true integrand value at $x$ [S1]. This assumption causes the $h_i$ and $f$ terms to cancel out in the weight ratio [S1]. Define a practical weight term $\tilde{w}_i'(x, y_i^*)$ [S1]:
    $$
    \tilde{w}_i'(x, y_i^*) = \frac{n_i p_i'(x, y_i^*)}{\sum_{k=1}^n n_k p_k'(x, y_k^*)}
    $$
    where the effective PDF $p_i'$ is defined as [S1]:
    $$
    p_i'(x, y_i^*) = \begin{cases} 
    p_i(x) & \text{if } 1 \leq i \leq u \\
    \frac{p_i(x, y_i^*)}{\int_{\mathcal{D}_{y_i}} K_i^2(x, y_i) dy_i} & \text{if } u < i \leq n
    \end{cases}
    $$
    In practice, for extended techniques, $y_i^*$ is set to the actual realization of the random variable (e.g., photon location) [S1]. For regular techniques, $y_i^*$ corresponds to the kernel center or is irrelevant as $p_i'$ depends only on $x$ [S1].

5.  **Final Estimator Evaluation:**
    For each sample $j$ of technique $i$, compute the weight $\tilde{w}_i'$ using the appropriate $p_i'$ and the sample location. Multiply the raw contribution of the sample by this weight and sum all weighted contributions [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extended_mis_combine(samples, techniques, n_samples_per_technique):
    """
    Combines contributions from regular and extended sampling techniques using Extended MIS.
    
    Args:
        samples: List of lists. samples[i] contains n_samples_per_technique[i] samples.
                 For regular techniques (i < u), sample is a point x.
                 For extended techniques (i >= u), sample is a tuple (x, y).
        techniques: List of technique objects. Each has:
            - is_extended: bool
            - pdf(x, y=None): returns p_i(x) or p_i(x, y)
            - kernel_integral_sq(x): returns integral of K_i^2 over D_yi for extended techniques.
            - contribution(x, y=None): returns f_i(x) or f_i(x, y).
        n_samples_per_technique: List of integers n_i.
        
    Returns:
        float: The combined estimator value F^C.
    """
    u = len([t for t in techniques if not t.is_extended])
    n = len(techniques)
    total_weighted_sum = 0.0
    
    # Precompute denominator terms for the weight heuristic
    # We need to compute sum_{k=1}^n n_k * p_k'(x, y_k*)
    # However, the weight depends on the specific sample's location.
    # The standard MIS application applies weights per sample.
    # The formula for weight w_i' depends on x and y_i*.
    # We must compute the denominator for each sample individually or group by x if applicable.
    # In volumetric path tracing, x is often the same for multiple techniques (e.g. scattering event).
    # Here we assume we process samples for a specific x.
    
    # To implement strictly from the formula:
    # For each sample j of technique i:
    #   1. Determine x and y*
    #   2. Compute p_i'
    #   3. Compute weight w_i' = (n_i * p_i') / sum_k(n_k * p_k')
    #   4. Add w_i' * (f_i / p_i) to sum
    
    # Note: The source defines F^C as sum over i,j. The weight w_i is applied to the term f/p.
    # The term f/p is the raw estimator. The weight normalizes it.
    
    for i in range(n):
        n_i = n_samples_per_technique[i]
        technique = techniques[i]
        is_extended = technique.is_extended
        
        # Collect all samples for this technique to compute the denominator sum?
        # No, the weight w_i'(x, y_i*) depends on the specific sample's y_i*.
        # The denominator sum_{k} n_k p_k'(x, y_k*) also depends on the specific y_k* of each technique k.
        # This implies the weights are computed per-sample, but the denominator aggregates 
        # the "representative" y_k* for each technique k.
        
        # In practice, for a given x, each technique provides one or more samples.
        # The "y_k*" for technique k is the sample used to evaluate the weight.
        
        # Let's assume we are evaluating the contribution of a specific set of samples 
        # for a single point x.
        
        # Step 1: Compute the denominator D = sum_{k=1}^n n_k * p_k'(x, y_k*)
        # We need a representative y_k* for each technique k.
        # For regular techniques, y_k* is not used, p_k' = p_k(x).
        # For extended techniques, y_k* is the sample location y.
        
        # If we have multiple samples per technique, which y_k* to use?
        # The source says "y_i* can be set to an arbitrary y_i within the kernel support".
        # In practice, use the actual realization. If multiple samples, we might average 
        # or pick one. The source implies one representative per technique for the weight calculation.
        
        # Let's assume samples[i] is a list of samples. We pick the first one as y_i* for weight calculation?
        # Or we compute weights for each sample individually?
        # The formula w_i'(x, y_i*) suggests the weight depends on y_i*.
        # If we have multiple samples, do we use the same weight for all?
        # The source says "y_i* appears... because we have pulled the terms".
        # It seems the weight is constant for all samples of technique i for a given x,
        # determined by the representative y_i*.
        
        # Let's pick the first sample as the representative y_i* for weight calculation.
        if len(samples[i]) == 0:
            continue
            
        rep_y_i = None
        if is_extended:
            rep_y_i = samples[i][0][1] # (x, y)
            
        # Compute denominator
        denominator = 0.0
        for k in range(n):
            n_k = n_samples_per_technique[k]
            tech_k = techniques[k]
            is_ext_k = tech_k.is_extended
            
            if is_ext_k:
                # Need rep_y_k for technique k
                if len(samples[k]) == 0:
                    p_k_prime = 0.0
                else:
                    rep_y_k = samples[k][0][1]
                    # p_k' = p_k(x, y_k*) / integral K_k^2
                    p_k_val = tech_k.pdf(samples[i][0][0], rep_y_k) # x is same for all techniques at this event
                    int_k_sq = tech_k.kernel_integral_sq(samples[i][0][0])
                    if int_k_sq == 0:
                        p_k_prime = 0.0
                    else:
                        p_k_prime = p_k_val / int_k_sq
            else:
                # Regular technique
                # p_k' = p_k(x)
                p_k_prime = tech_k.pdf(samples[i][0][0])
                
            denominator += n_k * p_k_prime
            
        if denominator == 0:
            continue
            
        # Step 2: Compute weight for technique i
        if is_extended:
            # p_i' = p_i(x, y_i*) / integral K_i^2
            p_i_val = technique.pdf(samples[i][0][0], rep_y_i)
            int_i_sq = technique.kernel_integral_sq(samples[i][0][0])
            if int_i_sq == 0:
                p_i_prime = 0.0
            else:
                p_i_prime = p_i_val / int_i_sq
        else:
            p_i_prime = technique.pdf(samples[i][0][0])
            
        weight_i = (n_i * p_i_prime) / denominator
        
        # Step 3: Sum contributions
        for sample in samples[i]:
            if is_extended:
                x, y = sample
                f_val = technique.contribution(x, y)
                p_val = technique.pdf(x, y)
            else:
                x = sample
                f_val = technique.contribution(x)
                p_val = technique.pdf(x)
                
            if p_val == 0:
                continue
                
            raw_estimator = f_val / p_val
            total_weighted_sum += weight_i * raw_estimator
            
    return total_weighted_sum
```

**Parameters:**
*   $n$: Total number of sampling techniques.
*   $u$: Number of regular sampling techniques ($1 \leq i \leq u$).
*   $n-u$: Number of extended space sampling techniques ($u < i \leq n$).
*   $n_i$: Number of samples for technique $i$.

**Edge Cases and Pitfalls:**
*   **Kernel Integral Zero:** If $\int K_i^2 = 0$, the weight calculation divides by zero. Guard against this [S1].
*   **PDF Zero:** If $p_i(x, y_i) = 0$, the raw estimator is undefined. Guard against division by zero [S1].
*   **Approximation Validity:** The approximation $h_i(x, y_i^*) \approx f(x)$ assumes the contribution function and PDF are constant within the kernel support. This may fail for highly varying functions or large kernels [S1].
*   **Dimension Mismatch:** Standard MIS cannot be applied directly because extended techniques integrate over $\mathcal{D}_{y_i}$, changing the effective dimension of the estimator [S1].

## Subtopics

### [Multi-View Path Tracing with MWIS](1834-multi-view-path-tracing-with-mwis.md)
Uses Multiple Weighted Importance Sampling for combining views.

### [Weighted Importance Sampling](1849-weighted-importance-sampling.md)
General WIS method.

### [Direct Lighting with MIS in VolPathIntegrator](1877-direct-lighting-with-mis-in-volpathintegrator.md)
Applies MIS to direct lighting in volumetric path tracing.

## Operating characteristics
*   **Time Complexity:** Linear in the number of samples $n_i$ and techniques $n$. The approximation avoids the expensive integration over $\mathcal{D}_{y_i}$ during runtime by precomputing or caching $\int K_i^2$ [S1].
*   **Memory Complexity:** Depends on storing samples and technique data. No significant additional memory for the heuristic itself [S1].
*   **Strengths:** Allows combining unbiased path tracing with biased density estimators (photon/beam tracing) without reformulating all estimators to the same space [S1]. Provides provably good weights [S1].
*   **Limitations:** The approximation of $\kappa_i(x)$ may introduce bias or suboptimal weights if the kernel support is large or the function varies rapidly [S1].
*   **When to Use:** When mixing path tracing with photon mapping or beam tracing in participating media [S1].
*   **When Not to Use:** When all techniques operate on the same space (use standard MIS) or when high accuracy of weights is critical and the approximation assumptions are violated [S1].

## Background
Extended Multiple Importance Sampling was introduced in the paper "Unifying points, beams, and paths in volumetric light transport simulation" [S1]. It extends the MIS framework to handle estimators on spaces of different dimensions, specifically addressing the integration over kernel supports in photon and beam density estimators [S1]. Prior work [Hachisuka et al. 2012; Georgiev et al. 2012] reformulated estimators to the same space; Extended MIS avoids this by extending the heuristic directly [S1].

## Sources
[S1] Unifying points, beams, and paths in volumetric light transport simulation, https://doi.org/10.1145/2601097.2601219, fetched 2026-10-30
