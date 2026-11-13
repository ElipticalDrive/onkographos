# K-wise moment-matched merging with L2-optimal amplitude

Also called: cost-aware merging, substitutive LOD merging

## Overview
K-wise moment-matched merging with L2-optimal amplitude is a level-of-detail (LOD) technique that reduces a cluster of $K$ Gaussian splats into a single representative splat by matching statistical moments and optimizing the resulting amplitude to minimize the integrated squared error [S1, unofficial]. It distinguishes itself from simple arithmetic averaging or mass-ranked culling by explicitly solving for the optimal amplitude coefficient via L2 optimization, ensuring the merged splat best approximates the visual energy of the original group [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A cluster $C$ of $K$ Gaussians $\{G_i\}_{i=1}^K$, where each $G_i$ has position $\mathbf{t}_i$, covariance $\mathbf{\Sigma}_i$, opacity $\alpha_i$, and amplitude/color coefficients $c_i$ [S1, unofficial].
*   **Output:** A single merged Gaussian $G_m$ with parameters $\mathbf{t}_m, \mathbf{\Sigma}_m, \alpha_m, c_m$ [S1, unofficial].

**Data Structures:**
The method operates on the standard Gaussian representation defined in [Gaussian clustering and merging](8436-gaussian-clustering-and-merging.md) [G]. No additional bit-packing is specified in the core merging algorithm, though practical implementations may use quantized encodings (e.g., uint16 centers, uint8 Cholesky factors) as described in [S1, unofficial].

**Steps:**
1.  **Partitioning:** Group the $N$ input Gaussians into clusters using one of three algorithms: greedy hierarchical merging (à la Runnalls), cost-increment relocation, or joint relaxation [S1, unofficial].
2.  **Moment Matching (Position and Shape):** Compute the first and second moments of the cluster to determine the merged position $\mathbf{t}_m$ and covariance $\mathbf{\Sigma}_m$ [S1, unofficial]. Using weights $w_i$ (typically opacity $\alpha_i$ or volume-normalized weights) [G]:
    $$ W = \sum_{i \in C} w_i $$
    $$ \mathbf{t}_{m} = \frac{1}{W} \sum_{i \in C} w_i \mathbf{t}_i $$
    $$ \mathbf{\Sigma}_{m} = \frac{1}{W} \sum_{i \in C} w_i (\mathbf{\Sigma}_i + \mathbf{t}_i \mathbf{t}_i^T) - \mathbf{t}_{m} \mathbf{t}_{m}^T $$
3.  **L2-Optimal Amplitude:** Optimize the amplitude coefficient $c_m$ (and potentially opacity $\alpha_m$) to minimize the integrated squared error between the merged Gaussian and the sum of the original Gaussians [S1, unofficial]. For the specific case of merging two Gaussians, the relative merging error $\epsilon$ for the amplitude-optimized moment-matched template is given by:
    $$ \epsilon = \sin^2 \alpha $$
    where $\alpha$ is a geometric angle parameter derived from the relative positions and covariances of the splats [S1, unofficial]. The amplitude is optimized to minimize the global L2 error of the cluster approximation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Gaussian:
    def __init__(self, position, covariance, alpha, amplitude):
        self.position = position
        self.covariance = covariance
        self.alpha = alpha
        self.amplitude = amplitude

def outer_product(v1, v2):
    return [[v1[i] * v2[j] for j in range(len(v2))] for i in range(len(v1))]

def kwise_moment_matched_merge(cluster):
    if not cluster:
        return None

    weights = [g.alpha for g in cluster]
    W = sum(weights)

    if W == 0:
        return Gaussian(cluster[0].position, cluster[0].covariance, 0.0, 0.0)

    t_m = [0.0, 0.0, 0.0]
    for g, w in zip(cluster, weights):
        for k in range(3):
            t_m[k] += w * g.position[k]
    for k in range(3):
        t_m[k] /= W

    Sigma_m = [[0.0, 0.0, 0.0], [0.0, 0.0, 0.0], [0.0, 0.0, 0.0]]
    for g, w in zip(cluster, weights):
        t_t_T = outer_product(g.position, g.position)
        for r in range(3):
            for c in range(3):
                Sigma_m[r][c] += w * (g.covariance[r][c] + t_t_T[r][c])
    for r in range(3):
        for c in range(3):
            Sigma_m[r][c] /= W
    t_m_t_m_T = outer_product(t_m, t_m)
    for r in range(3):
        for c in range(3):
            Sigma_m[r][c] -= t_m_t_m_T[r][c]

    c_m = 0.0
    alpha_m = 0.0
    for g, w in zip(cluster, weights):
        c_m += w * g.amplitude
        alpha_m += w * g.alpha
    c_m /= W
    alpha_m /= W

    # [G] Closed-form L2 solver for K>2 not provided in sources; 
    # weighted average used as standard approximation for amplitude/opacity.
    return Gaussian(position=t_m, covariance=Sigma_m, alpha=alpha_m, amplitude=c_m)
```

**Parameters:**
*   $K$: The number of splats to merge. Not given in the sources.
*   Weighting scheme: Opacity $\alpha_i$ or volume-normalized. Not given in the sources.

**Edge Cases:**
*   **Zero Opacity:** If the sum of weights $W$ is zero, the position is undefined. The pseudocode handles this by returning a zero-opacity Gaussian [G].
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
