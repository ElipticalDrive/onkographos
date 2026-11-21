# Fast Cache Estimator

Also called: Categorical sampling estimator

## Overview
The Fast Cache Estimator is an unbiased Monte Carlo estimator for the incoming radiance term in the volume rendering integral, designed to reduce computational cost by decoupling density queries from appearance evaluations [S1]. It reduces the number of expensive physically-based appearance evaluations from $N$ to $K \ll N$ samples drawn from a categorical distribution based on render weights, while still requiring density queries for all $N$ points to determine those weights [S1]. Unlike the [Volume Rendering Integral](8682-volume-rendering-integral.md) which typically sums over all sample points, this method approximates the integral using a subset of points, trading off the number of appearance evaluations for a slight increase in variance that is mitigated by importance sampling [S1].

## How it works
**Inputs and Outputs:**
- **Input:** Ray origin $\mathbf{x}_{origin}$, ray direction $\mathbf{d}_{out}$, number of density query points $N$, number of appearance evaluation points $K$.
- **Output:** Estimated incoming radiance $\hat{L}_i(\mathbf{x}_{origin}, \mathbf{d}_{out})$.

**Data Structures:**
- **Render Weights:** An array $w$ of size $N$, where $w_j$ is the render weight for the $j$-th sample point along the ray. These weights are derived from the volume density queries at positions $\mathbf{x}_{pos}(t_j)$.
- **Sample Positions:** An array of positions $\mathbf{x}_{pos}(t_1), \dots, \mathbf{x}_{pos}(t_N)$ corresponding to the $N$ density query locations.

**Steps:**
1. **Query Density:** For each of the $N$ sample points along the ray, query the volume density to compute the render weights $w_1, \dots, w_N$. Note that this step is required even though appearance is not evaluated at all points [S1].
2. **Normalize Weights:** Ensure the weights $w_1, \dots, w_N$ form a valid probability distribution (i.e., $\sum_{j=1}^N w_j = 1$). If they do not sum to 1, normalize them: $w_j \leftarrow w_j / \sum_{k=1}^N w_k$ [S1].
3. **Sample Indices:** Draw $K$ indices $j_1, \dots, j_K$ independently from the categorical distribution defined by the weights $w_1, \dots, w_N$ [S1].
4. **Evaluate Appearance:** For each sampled index $j_k$, evaluate the outgoing radiance $L_o(\mathbf{x}_{pos}(t_{j_k}), \mathbf{d}_{out})$ at the corresponding position [S1].
5. **Compute Estimator:** Calculate the average of the evaluated outgoing radiances:
   $$ \hat{L}_i(\mathbf{x}_{origin}, \mathbf{d}_{out}) = \frac{1}{K} \sum_{k=1}^{K} L_o(\mathbf{x}_{pos}(t_{j_k}), \mathbf{d}_{out}) $$
   [S1]

**Parameters:**
- **$K$:** The number of appearance evaluations per ray. Typical value: $K=1$ is sufficient if the render weight distribution has a single peak [S1]. In practice, $K=1$ provides the best results in terms of PSNR for synthetic datasets [S1].

**Pitfalls:**
- **Density Query Overhead:** The method requires querying volume density for all $N$ points to determine the weights, even though appearance is only evaluated at $K$ points [S1]. This can be a bottleneck if density queries are expensive.
- **Variance:** If the render weight distribution is multimodal or flat, $K=1$ may result in high variance. In such cases, increasing $K$ may be necessary, though it increases computational cost [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fast_cache_estimator(ray_origin, ray_direction, N, K, query_density, evaluate_appearance):
    """
    Estimates incoming radiance using the Fast Cache Estimator.
    
    Args:
        ray_origin: Origin of the ray.
        ray_direction: Direction of the ray.
        N: Number of density query points.
        K: Number of appearance evaluation points (K << N).
        query_density: Function that returns render weight for a position.
        evaluate_appearance: Function that returns outgoing radiance for a position and direction.
        
    Returns:
        Estimated incoming radiance.
    """
    # Step 1: Query density for all N points to get weights
    weights = []
    positions = []
    for i in range(N):
        # Assume positions are sampled along the ray, e.g., uniformly or via stratified sampling
        # The exact sampling strategy for positions is not specified in the source,
        # but they are fixed for the ray.
        t = i / N  # Placeholder for position sampling; actual implementation depends on ray marching
        pos = ray_origin + t * ray_direction
        positions.append(pos)
        w = query_density(pos)
        weights.append(w)
    
    # Step 2: Normalize weights
    total_weight = sum(weights)
    if total_weight == 0:
        return 0.0  # Handle degenerate case where no density is encountered
    normalized_weights = [w / total_weight for w in weights]
    
    # Step 3: Sample K indices from categorical distribution
    sampled_indices = categorical_sample(normalized_weights, K)
    
    # Step 4 & 5: Evaluate appearance and compute estimator
    total_radiance = 0.0
    for idx in sampled_indices:
        pos = positions[idx]
        radiance = evaluate_appearance(pos, ray_direction)
        total_radiance += radiance
        
    return total_radiance / K

def categorical_sample(weights, K):
    """
    Samples K indices from a categorical distribution defined by weights.
    """
    import random
    indices = []
    for _ in range(K):
        r = random.random()
        cumulative = 0.0
        for i, w in enumerate(weights):
            cumulative += w
            if r <= cumulative:
                indices.append(i)
                break
    return indices
```

## Subtopics

### Unbiasedness Proof for Categorical Sampling Estimator
This section provides the proof that the Fast Cache Estimator is an unbiased estimator for the volume rendering quadrature [S1]. The proof expands the expectation of the estimator and shows that it equals the sum of render weights multiplied by outgoing radiance [S1].

**Steps:**
1. Expand the expectation of the estimator: $\mathbb{E}[\hat{L}_i(\mathbf{x}_{origin}, \mathbf{d}_{out})] = \mathbb{E}[\frac{1}{K} \sum_{k=1}^{K} L_o(\mathbf{x}_{pos}(t_{j_k}), \mathbf{d}_{out})]$ [S1].
2. Substitute the categorical expectation, where $j_k \sim \operatorname{Cat}(w_1, \ldots, w_N)$: $\frac{1}{K} \sum_{k=1}^{K} \sum_{j=1}^{N} w_j L_o(\mathbf{x}_{pos}(t_j), \mathbf{d}_{out})$ [S1].
3. Simplify the expression to the final sum: $\sum_{j=1}^{N} w_j L_o(\mathbf{x}_{pos}(t_j), \mathbf{d}_{out})$, which is the quadrature value [S1].

**Formula:**
$$ \mathbb{E}[\hat{L}_i(\mathbf{x}_{origin}, \mathbf{d}_{out})] = \sum_{j=1}^{N} w_j L_o(\mathbf{x}_{pos}(t_j), \mathbf{d}_{out}) $$
[S1]

**Parameters:**
- $w_1, \ldots, w_N$: Render weights for the ray $(\mathbf{o}, \boldsymbol{\omega}_o)$ [S1].

### Unbiasedness Proof for Fast Cache Estimator
This section provides the proof that the Fast Cache Estimator is an unbiased estimator of the rendering equation, provided that the incoming illumination from the NeRF cache is correct [S1]. The proof leverages the unbiasedness of the estimator for the incoming radiance term [S1].

**Steps:**
1. Expand the expectation using the unbiasedness of the incoming radiance estimator: $\int_{\Omega} f(\mathbf{x}_{pos}^t, \mathbf{d}_{in}, \mathbf{d}_{out}) \hat{L}_i^{\text{NeRF}}(\mathbf{x}, \mathbf{d}_{in}) (\mathbf{n} \cdot \mathbf{d}_{in}) d\mathbf{d}_{in} + \int_{\Omega} f(\mathbf{x}_{pos}^t, \mathbf{d}_{in}, \mathbf{d}_{out}) (\hat{L}_i^{\text{NeRF}} - \hat{L}_i^{\text{fast}})(\mathbf{x}, \mathbf{d}_{in}) (\mathbf{n} \cdot \mathbf{d}_{in}) d\mathbf{d}_{in}$ [S1].
2. Simplify to the final integral: $\int_{\Omega} f(\mathbf{x}_{pos}^t, \mathbf{d}_{in}, \mathbf{d}_{out}) \hat{L}_i^{\text{NeRF}}(\mathbf{x}, \mathbf{d}_{in}) (\mathbf{n} \cdot \mathbf{d}_{in}) d\mathbf{d}_{in}$ [S1].

**Formula:**
$$ \mathbb{E}[\hat{L}_o] = \mathbb{E}[\hat{L}_o^{\text{fast}}] + \mathbb{E}[\Delta \hat{L}_o] $$
[S1]

## Operating characteristics
- **Time Complexity:** The method requires $N$ density queries and $K$ appearance evaluations per ray. Since $K \ll N$, the computational cost is dominated by the density queries. The total number of secondary ray samples is reduced from $N \cdot M$ to $K \cdot M$, where $M$ is the number of rays [S1].
- **Memory Complexity:** Requires storing $N$ render weights and $N$ positions for each ray.
- **Measured Performance:** On the TensoIR-synthetic dataset, $K=1$ provides the best NVS PSNR (34.915) and Albedo PSNR (30.345) with the lowest MAE (3.355) compared to $K=2, 4, 8$ [S1].
- **Strengths:** Significantly reduces the number of expensive appearance evaluations. Unbiased estimator.
- **Limitations:** Still requires density queries for all $N$ points. Variance can be high if the weight distribution is not peaked [S1].
- **When to use:** When appearance evaluation is significantly more expensive than density queries, and the render weight distribution is peaked.
- **When not to use:** When density queries are also expensive, or when the weight distribution is multimodal and $K=1$ leads to high variance.

## Background
The Fast Cache Estimator was introduced in "Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering" by B. Attal et al. [S1]. It is part of a broader effort to accelerate volume rendering and inverse rendering by reducing the computational cost of Monte Carlo integration. The method builds on the idea of using importance sampling to reduce the number of samples needed for convergence [S1].

## Sources
[S1] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
