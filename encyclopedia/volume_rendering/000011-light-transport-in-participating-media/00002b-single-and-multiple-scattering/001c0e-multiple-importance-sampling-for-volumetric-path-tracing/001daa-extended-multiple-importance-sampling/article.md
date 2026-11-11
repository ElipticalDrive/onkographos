# Extended Multiple Importance Sampling

Also called: MIS for different dimensional spaces

## Overview
Extended Multiple Importance Sampling (Extended MIS) is a framework for combining estimators of integrals defined on spaces of different dimensions, such as standard path sampling and extended-space density estimators (e.g., photon mapping or beam tracing) [S1]. It solves the problem of variance reduction in volumetric path tracing when mixing unbiased path contributions with biased density estimates that operate on higher-dimensional spaces [S1]. Unlike standard MIS, which requires all strategies to sample from the same domain, Extended MIS derives a balance heuristic that accounts for the additional integration over the kernel support inherent in extended-space techniques [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** $n$ sampling techniques; $u$ regular techniques ($1 \leq i \leq u$) sampling from domain $\mathcal{D}_x$; $n-u$ extended techniques ($u < i \leq n$) sampling from $\mathcal{D}_x \times \mathcal{D}_{y_i}$; $n_i$ samples per technique $i$ [S1].
*   **Outputs:** Combined estimator value $F^C$ and weights $\tilde{w}_i$ [S1].

**Data Structures:**
*   **Technique Index:** Integer $i \in [1, n]$ [S1].
*   **Sample Index:** Integer $j \in [1, n_i]$ [S1].
*   **Regular Sample:** $X_{i,j} \in \mathcal{D}_x$ [S1].
*   **Extended Sample:** $(X_{i,j}, Y_{i,j}) \in \mathcal{D}_x \times \mathcal{D}_{y_i}$ [S1].
*   **Contribution Function:** $f_i(x)$ for regular; $f_i(x, y_i)$ for extended [S1].
*   **PDF:** $p_i(x)$ for regular; $p_i(x, y_i)$ for extended [S1].
*   **Kernel:** $K_i(x, y_i)$ for extended techniques [S1].
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
    For regular techniques ($1 \leq i \leq u$), $Y_{i,j}$ is undefined and $f_i, p_i$ depend only on $X_{i,j}$ [S1]. For extended techniques ($u < i \leq n$), the term uses the extended space formulation [S1].

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
    For extended techniques, $\kappa_i(x)$ includes an integral over the kernel support $\mathcal{D}_{y_i}$ to account for the variance contribution of MC sampling over that domain [S1].

3.  **Approximate $\kappa_i(x)$ for Extended Techniques:**
    The exact integral in $\kappa_i(x)$ is impractical to evaluate [S1]. Assume $h_i(x, y_i)$ and $p_i(x, y_i)$ are constant within the kernel support $K_i$ [S1]. Let $y_i^*$ be a representative sample (e.g., the actual photon location) [S1]. The approximation is [S1]:
    $$
    \kappa_i(x) \approx \frac{h_i^2(x, y_i^*)}{p_i(x, y_i^*)} \int_{\mathcal{D}_{y_i}} K_i^2(x, y_i) dy_i
    $$
    Substitute $h_i(x, y_i^*) = f_i(x, y_i^*) / K_i(x, y_i^*)$ [S1].

4.  **Simplify Weights for Implementation:**
    Assume $h_i(x, y_i^*) \approx f(x)$, where $f(x)$ is the true integrand value at $x$ [S1]. This assumption causes the $h_i$ and $f$ terms to cancel out in the weight ratio [S1]. Define a practical weight term $\tilde{w}_i'(x, y_i^*)$ [S1]:
    $$
    \tilde{w}_i'(x, y_i^*) = \frac{n_i p_i'(x, y_i^*)}{\sum_{k=1}^n n_k p_k'(x, y_k^*)}
    $$
    The effective term $p_i'(x, y_i^*)$ is defined as [S1]:
    $$
    p_i'(x, y_i^*) = \begin{cases} 
    p_i(x) & \text{if } 1 \leq i \leq u \\
    \frac{p_i(x, y_i^*)}{\int_{\mathcal{D}_{y_i}} K_i^2(x, y_i) dy_i} & \text{if } u < i \leq n
    \end{cases}
    $$
    In practice, for extended techniques, $y_i^*$ is set to the actual realization of the random variable (e.g., photon location) [S1]. For regular techniques, $y_i^*$ is irrelevant as $p_i'$ depends only on $x$ [S1].

5.  **Final Estimator Evaluation:**
    For each sample $j$ of technique $i$, compute the weight $\tilde{w}_i'$ using the appropriate $p_i'$ and the sample location [S1]. Multiply the raw contribution of the sample by this weight and sum all weighted contributions [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extended_mis_combine(x, techniques, samples_per_technique):
    """
    Combines contributions from regular and extended sampling techniques using Extended MIS.
    
    Args:
        x: Point in D_x where estimators are evaluated.
        techniques: List of technique objects. Each has:
            - is_extended: bool
            - pdf(x, y=None): returns p_i(x) or p_i(x, y)
            - kernel_integral_sq(x): returns integral of K_i^2 over D_yi for extended techniques.
            - contribution(x, y=None): returns f_i(x) or f_i(x, y).
        samples_per_technique: List of lists. samples_per_technique[i] contains n_i samples.
                 For regular techniques (i < u), sample is a point x.
                 For extended techniques (i >= u), sample is a tuple (x, y).
                 
    Returns:
        float: The combined estimator value F^C.
    """
    n = len(techniques)
    n_i_list = [len(s) for s in samples_per_technique]
    
    # Step 1: Determine representative y_i* for each technique
    rep_y = []
    for i in range(n):
        if len(samples_per_technique[i]) > 0:
            if techniques[i].is_extended:
                rep_y.append(samples_per_technique[i][0][1])
            else:
                rep_y.append(None)
        else:
            rep_y.append(None)
            
    # Step 2: Compute p_i' for each technique
    p_prime = []
    for i in range(n):
        if rep_y[i] is None:
            p_prime.append(techniques[i].pdf(x))
        else:
            int_k_sq = techniques[i].kernel_integral_sq(x)
            if int_k_sq == 0.0:
                p_prime.append(0.0)
            else:
                p_prime.append(techniques[i].pdf(x, rep_y[i]) / int_k_sq)
                
    # Step 3: Compute denominator D = sum_{k=1}^n n_k * p_k'
    denominator = sum(n_i_list[k] * p_prime[k] for k in range(n))
    
    if denominator == 0.0:
        return 0.0
        
    # Step 4: Compute weights and sum contributions
    total = 0.0
    for i in range(n):
        n_i = n_i_list[i]
        if n_i == 0:
            continue
            
        weight_i = (n_i * p_prime[i]) / denominator
        
        for sample in samples_per_technique[i]:
            if techniques[i].is_extended:
                _, y = sample
                f_val = techniques[i].contribution(x, y)
                p_val = techniques[i].pdf(x, y)
            else:
                f_val = techniques[i].contribution(x)
                p_val = techniques[i].pdf(x)
                
            if p_val == 0.0:
                continue
                
            raw_estimator = f_val / p_val
            total += (1.0 / n_i) * weight_i * raw_estimator
            
    return total
```

**Parameters:**
*   $n$: Total number of sampling techniques [S1].
*   $u$: Number of regular sampling techniques ($1 \leq i \leq u$) [S1].
*   $n-u$: Number of extended space sampling techniques ($u < i \leq n$) [S1].
*   $n_i$: Number of samples for technique $i$ [S1].

**Edge Cases and Pitfalls:**
*   **Kernel Integral Zero:** If $\int K_i^2 = 0$, the weight calculation divides by zero. Guard against this [S1].
*   **PDF Zero:** If $p_i(x, y_i) = 0$, the raw estimator is undefined. Guard against division by zero [S1].
*   **Approximation Validity:** The approximation $h_i(x, y_i^*) \approx f(x)$ assumes the contribution function and PDF are constant within the kernel support [S1].
*   **Dimension Mismatch:** Standard MIS cannot be applied directly because extended techniques integrate over $\mathcal{D}_{y_i}$, changing the effective dimension of the estimator [S1].

## Subtopics

### [Multi-View Path Tracing with MWIS](1834-multi-view-path-tracing-with-mwis.md)
Uses Multiple Weighted Importance Sampling to combine views. It sets itself apart by weighting samples based on view-dependent coverage rather than dimensionality, and is chosen when rendering multi-camera or light-field scenes.

### [Weighted Importance Sampling](1849-weighted-importance-sampling.md)
General WIS method for combining estimators with known importance weights. It sets itself apart by not requiring a balance heuristic derivation, and is chosen when weights are analytically known or precomputed.

### [Direct Lighting with MIS in VolPathIntegrator](1877-direct-lighting-with-mis-in-volpathintegrator.md)
Applies MIS to direct lighting in volumetric path tracing. It sets itself apart by focusing specifically on the phase function vs. next-event estimation trade-off, and is chosen for efficient direct illumination in participating media.

## Operating characteristics
*   **Time Complexity:** Linear in the number of samples $n_i$ and techniques $n$. The approximation avoids the expensive integration over $\mathcal{D}_{y_i}$ during runtime by precomputing or caching $\int K_i^2$ [S1].
*   **Strengths:** Allows combining unbiased path tracing with biased density estimators (photon/beam tracing) without reformulating all estimators to the same space [S1]. Provides provably good weights [S1].
*   **When to Use:** When mixing path tracing with photon mapping or beam tracing in participating media [S1].
*   **When Not to Use:** When all techniques operate on the same space (use standard MIS) [S1].

## Background
Extended Multiple Importance Sampling was introduced in the paper "Unifying points, beams, and paths in volumetric light transport simulation" [S1]. It extends the MIS framework to handle estimators on spaces of different dimensions, specifically addressing the integration over kernel supports in photon and beam density estimators [S1]. Prior work [Hachisuka et al. 2012; Georgiev et al. 2012] reformulated estimators to the same space; Extended MIS avoids this by extending the heuristic directly [S1].

## Sources
[S1] Unifying points, beams, and paths in volumetric light transport simulation, https://doi.org/10.1145/2601097.2601219, fetched 2026-10-30
