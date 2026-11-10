# K-wise moment-matched merging with L2-optimal amplitude

Also called: cost-aware merging, substitutive LOD merging

## Overview
K-wise moment-matched merging with L2-optimal amplitude is a level-of-detail (LOD) technique that reduces a cluster of $K$ Gaussian splats into a single representative splat by matching statistical moments and optimizing the resulting amplitude to minimize the integrated squared error [S1, unofficial]. It distinguishes itself from simple arithmetic averaging or mass-ranked culling by explicitly solving for the optimal amplitude coefficient via L2 optimization, ensuring the merged splat best approximates the visual energy of the original group [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A cluster $C$ of $K$ Gaussians $\{G_i\}_{i=1}^K$, where each $G_i$ has position $\mathbf{t}_i$, covariance $\mathbf{\Sigma}_i$, opacity $\alpha_i$, and amplitude/color coefficients $c_i$.
*   **Output:** A single merged Gaussian $G_m$ with parameters $\mathbf{t}_m, \mathbf{\Sigma}_m, \alpha_m, c_m$.

**Data Structures:**
The method operates on the standard Gaussian representation defined in [Gaussian clustering and merging](8436-gaussian-clustering-and-merging.md). No additional bit-packing is specified in the core merging algorithm, though practical implementations may use quantized encodings (e.g., uint16 centers, uint8 Cholesky factors) as described in [S1, unofficial].

**Steps:**
1.  **Partitioning:** Group the $N$ input Gaussians into clusters. The sources describe three partitioning algorithms:
    *   Greedy hierarchical merging (à la Runnalls).
    *   Cost-increment relocation.
    *   Joint relaxation [S1, unofficial].
2.  **Moment Matching (Position and Shape):** Compute the first and second moments of the cluster to determine the merged position $\mathbf{t}_m$ and covariance $\mathbf{\Sigma}_m$. Using weights $w_i$ (typically opacity $\alpha_i$ or volume-normalized weights):
    $$ W = \sum_{i \in C} w_i $$
    $$ \mathbf{t}_{m} = \frac{1}{W} \sum_{i \in C} w_i \mathbf{t}_i $$
    $$ \mathbf{\Sigma}_{m} = \frac{1}{W} \sum_{i \in C} w_i (\mathbf{\Sigma}_i + \mathbf{t}_i \mathbf{t}_i^T) - \mathbf{t}_{m} \mathbf{t}_{m}^T $$
    [S1, unofficial]
3.  **L2-Optimal Amplitude:** Optimize the amplitude coefficient $c_m$ (and potentially opacity $\alpha_m$) to minimize the integrated squared error between the merged Gaussian and the sum of the original Gaussians. For the specific case of merging two Gaussians, the relative merging error $\epsilon$ for the amplitude-optimized moment-matched template is given by:
    $$ \epsilon = \sin^2 \alpha $$
    where $\alpha$ is a geometric angle parameter derived from the relative positions and covariances of the splats [S1, unofficial]. For $K > 2$, the amplitude is optimized to minimize the global L2 error of the cluster approximation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def kwise_moment_matched_merge(cluster: List[Gaussian]) -> Gaussian:
    """
    Merges a cluster of Gaussians into one using moment matching and L2 amplitude optimization.
    """
    # Step 1: Compute weights and total weight
    # w_i is typically alpha_i or a volume-normalized weight
    weights = [g.alpha for g in cluster]
    W = sum(weights)
    
    if W == 0:
        return cluster[0] # Fallback for empty or zero-opacity cluster

    # Step 2: Compute merged position (First Moment)
    t_m = np.zeros(3)
    for g, w in zip(cluster, weights):
        t_m += w * g.position
    t_m /= W

    # Step 3: Compute merged covariance (Second Moment)
    # Sigma_m = (1/W) * sum(w_i * (Sigma_i + t_i * t_i^T)) - t_m * t_m^T
    Sigma_m = np.zeros((3, 3))
    for g, w in zip(cluster, weights):
        # Outer product t_i * t_i^T
        t_t_T = np.outer(g.position, g.position)
        Sigma_m += w * (g.covariance + t_t_T)
    Sigma_m /= W
    Sigma_m -= np.outer(t_m, t_m)

    # Step 4: Compute merged amplitude/color via L2 optimization
    # The exact closed-form for K > 2 is not provided in the sources.
    # For K=2, the error is sin^2(alpha).
    # General approach: Solve for c_m that minimizes || sum(G_i) - G_m ||^2_L2
    # This typically involves solving a linear system or using a gradient step
    # to find the optimal amplitude coefficient c_m and opacity alpha_m.
    # [G] Placeholder for specific K-wise L2 solver implementation details.
    # In practice, this may be approximated by weighted averaging of amplitudes
    # or a numerical optimization step.
    
    # Simplified amplitude estimation (common heuristic if closed-form is complex):
    # c_m = sum(w_i * c_i) / W
    # However, the source emphasizes L2-optimal amplitude.
    # We assume a solver function exists for the L2 minimization.
    c_m = solve_l2_amplitude(cluster, t_m, Sigma_m)
    alpha_m = compute_optimal_alpha(cluster, t_m, Sigma_m, c_m)

    return Gaussian(position=t_m, covariance=Sigma_m, alpha=alpha_m, amplitude=c_m)

def solve_l2_amplitude(cluster, t_m, Sigma_m):
    """
    Solves for the amplitude c_m that minimizes the integrated squared error.
    This function encapsulates the L2 optimization step.
    """
    # [G] Implementation detail: The L2 error integral between Gaussians
    # can be computed analytically. The optimization finds c_m such that
    # the derivative of the error with respect to c_m is zero.
    # For two Gaussians, this relates to the geometric angle alpha.
    # For K Gaussians, it is a weighted combination.
    # A robust implementation would use a numerical optimizer or
    # derive the specific linear system for the K-wise case.
    pass
```

**Parameters:**
*   $K$: The number of splats to merge. Not given in the sources.
*   Weighting scheme: Opacity $\alpha_i$ or volume-normalized. Not given in the sources.

**Edge Cases:**
*   **Zero Opacity:** If the sum of weights $W$ is zero, the position is undefined. The pseudocode handles this by returning the first Gaussian or a default [G].
*   **Degenerate Covariance:** If $\mathbf{\Sigma}_m$ becomes singular or non-positive definite, it may need regularization [G].

## Subtopics

### Analytical Merging of Gaussian Splats
A closed-form framework for merging two Gaussian splats using L2 integrated squared-error, providing a geometric interpretation of the relative merging error [S1, unofficial]. The relative merging error $\epsilon$ for the amplitude-optimized moment-matched template is $\epsilon = \sin^2 \alpha$, where $\alpha$ is the geometric angle parameter [S1, unofficial]. It is implemented in Luxar for merging Gaussian splats during fitting or LOD reduction [S1, unofficial].

## Operating characteristics
*   **Performance:** Cost-increment relocation matches greedy merging performance within 5.5% at compression factors of four and above [S1, unofficial].
*   **Quality:** On full fits, cost-aware merging outperforms mass-ranked culling regardless of amplitude refit [S1, unofficial]. On sparse subsamples, culling wins in the median [S1, unofficial].
*   **Complexity:** The moment matching is $O(K)$ per cluster. The L2 amplitude optimization adds computational cost, potentially requiring iterative solving or closed-form derivation for $K>2$ [S1, unofficial].
*   **Strengths:** Provides a faithful surrogate for global error once spatial partition separates bins; single trials deviate by tens of percent at moderate separation [S1, unofficial].
*   **Limitations:** Performance degrades on sparse subsamples compared to culling methods [S1, unofficial].

## Background
This method is part of the substitutive levels of detail for Gaussian splats, exploring the cost-metric design space (L2, Cauchy–Schwarz, gKL, W2, MMD, L∞) [S1, unofficial]. It is implemented in Luxar, a tool for Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data [S1, unofficial]. The analytical merging error for two splats is detailed in Supplementary Document 1 of the Luxar project [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
