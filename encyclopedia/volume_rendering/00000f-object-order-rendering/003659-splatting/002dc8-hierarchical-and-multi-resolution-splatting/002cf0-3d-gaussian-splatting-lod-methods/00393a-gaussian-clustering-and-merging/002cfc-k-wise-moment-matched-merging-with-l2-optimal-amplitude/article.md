# K-wise moment-matched merging with L2-optimal amplitude

Also called: cost-aware merging, substitutive LOD merging

## Overview
K-wise moment-matched merging with L2-optimal amplitude is a level-of-detail (LOD) technique that reduces the number of 3D Gaussian splats by grouping $K$ splats into a single representative splat. It achieves this by matching the statistical moments of the input splats and optimizing the resulting amplitude to minimize the L2 error [S1, unofficial]. Unlike [MIP-Splatting](6063-mip-splatting.md), which handles frequency aliasing, or [CityGaussian / VastGaussian / DoGaussian](6143-citygaussian-vastgaussian-dogaussian.md), which use spatial clustering for pruning, this method focuses on substitutive merging where the merged splat replaces the group entirely [S1, unofficial].

## How it works
**Inputs:** A set of $N$ 3D Gaussian splats, where each splat $i$ is defined by position $\mathbf{c}_i$, covariance matrix $\mathbf{\Sigma}_i$, and amplitude $\alpha_i$.
**Outputs:** A reduced set of splats, where groups of $K$ splats are replaced by a single merged splat.

**Data Structures:**
Each splat $i$ stores:
- $\mathbf{c}_i \in \mathbb{R}^3$: Center position.
- $\mathbf{\Sigma}_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix (symmetric positive definite).
- $\alpha_i \in \mathbb{R}$: Amplitude (opacity/intensity weight).

**Partitioning:**
The splats are partitioned into disjoint groups of size $K$ using one of three algorithms [S1, unofficial]:
1. **Greedy hierarchical:** Similar to Runnalls' algorithm, iteratively merging the pair with the lowest cost.
2. **Cost-increment relocation:** Iteratively moves splats to minimize the global merging cost.
3. **Joint relaxation:** Optimizes all splat assignments simultaneously.

**Merging Step (for a group of $K$ splats):**
For a group of splats $G = \{1, \dots, K\}$, the merged splat parameters $(\mathbf{c}', \mathbf{\Sigma}', \alpha')$ are computed as follows:

1. **Compute Weighted Moments:**
   Calculate the total weight $W$ and the first and second moments of the group [S1, unofficial]:
   $$ W = \sum_{i \in G} \alpha_i $$
   $$ \mathbf{c}' = \frac{1}{W} \sum_{i \in G} \alpha_i \mathbf{c}_i $$
   $$ \mathbf{M}' = \sum_{i \in G} \alpha_i (\mathbf{c}_i \mathbf{c}_i^T + \mathbf{\Sigma}_i) $$

2. **Compute Covariance:**
   The merged covariance matrix is derived from the second moment and the outer product of the mean [S1, unofficial]:
   $$ \mathbf{\Sigma}' = \mathbf{M}' - W \mathbf{c}' \mathbf{c}'^T $$
   *Note:* In practice, $\mathbf{\Sigma}'$ must be symmetrized and checked for positive definiteness. If not positive definite, regularization (e.g., adding $\epsilon \mathbf{I}$) is required [G].

3. **Optimize Amplitude:**
   The amplitude $\alpha'$ is optimized to minimize the L2 integrated squared error between the original splats and the merged splat. For the amplitude-optimized moment-matched template, the relative merging error $\epsilon$ is given by [S1, unofficial]:
   $$ \epsilon = \sin^2 \alpha $$
   where $\alpha$ is a geometric angle parameter related to the alignment of the splats in the feature space. The optimal $\alpha'$ is chosen to minimize the global L2 loss [S1, unofficial].

**Parameters:**
- $K$: Group size. Typical values are powers of 2 (e.g., 2, 4, 8) for LOD levels [S1, unofficial].
- Partitioning Algorithm: Chosen based on performance vs. speed trade-offs.

**Edge Cases and Pitfalls:**
- **Degenerate Covariance:** If the merged covariance $\mathbf{\Sigma}'$ is singular or not positive definite, the splat cannot be rendered correctly. Regularization is needed [G].
- **Precision:** Floating-point errors in moment calculation can accumulate. Use double precision for intermediate calculations if possible [G].
- **Boundary Conditions:** Splats near the boundary of the partitioning space may have asymmetric neighbors, affecting the cost metric [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def merge_splats(group):
    """
    Merges a group of K splats into one.
    group: list of dicts with keys 'c', 'Sigma', 'alpha'
    """
    W = 0.0
    c_prime = np.zeros(3)
    M_prime = np.zeros((3, 3))
    
    for splat in group:
        alpha = splat['alpha']
        c = splat['c']
        Sigma = splat['Sigma']
        
        W += alpha
        c_prime += alpha * c
        M_prime += alpha * (np.outer(c, c) + Sigma)
        
    if W == 0:
        return None # Empty group
        
    c_prime /= W
    Sigma_prime = M_prime - W * np.outer(c_prime, c_prime)
    
    # Regularization for numerical stability [G]
    Sigma_prime = 0.5 * (Sigma_prime + Sigma_prime.T)
    eigenvalues = np.linalg.eigvalsh(Sigma_prime)
    if np.any(eigenvalues <= 0):
        Sigma_prime += (abs(np.min(eigenvalues)) + 1e-6) * np.eye(3)
        
    # Optimize amplitude [S1, unofficial]
    # The optimal alpha' minimizes L2 error. 
    # For simplicity, we use the total weight as a baseline, 
    # but the source indicates an L2-optimal amplitude is derived 
    # from the geometric angle parameter alpha.
    # Exact closed-form for alpha' is not fully detailed in the snippet, 
    # but it is derived from the relative merging error epsilon = sin^2(alpha).
    # We assume alpha' is computed via an optimization step or closed-form 
    # based on the error metric.
    alpha_prime = compute_optimal_amplitude(group, c_prime, Sigma_prime)
    
    return {'c': c_prime, 'Sigma': Sigma_prime, 'alpha': alpha_prime}

def compute_optimal_amplitude(group, c_prime, Sigma_prime):
    """
    Computes the L2-optimal amplitude for the merged splat.
    This is a placeholder for the specific optimization routine 
    described in the source, which minimizes the L2 error.
    """
    # The source mentions epsilon = sin^2(alpha) for the relative error.
    # The optimal alpha' is likely derived from minimizing the global L2 loss.
    # Without the full pseudocode from the supplementary document, 
    # we assume a standard L2 projection or optimization.
    # [G] Placeholder for the specific amplitude optimization logic.
    # In practice, this might involve solving a linear system or using 
    # a closed-form solution if the error metric is quadratic.
    return sum(s['alpha'] for s in group) # Fallback: sum of amplitudes
```

## Subtopics

### Analytical Merging of Gaussian Splats
A closed-form framework for merging two Gaussian splats using L2 integrated squared-error, providing a geometric interpretation of the relative merging error [S1, unofficial]. The relative merging error is $\epsilon = \sin^2 \alpha$ for the amplitude-optimised moment-matched template, where $\alpha$ is the geometric angle parameter [S1, unofficial]. It is implemented in Luxar for merging Gaussian splats during fitting or LOD reduction [S1, unofficial].

## Operating characteristics
**Time Complexity:** Depends on the partitioning algorithm. Greedy hierarchical is $O(N \log N)$ or $O(N^2)$ depending on implementation. Cost-increment relocation is iterative and may be slower but yields better results [S1, unofficial].
**Memory Complexity:** $O(N)$ for storing splats. Merging reduces the number of splats by a factor of $K$.
**Performance:** Cost-increment relocation matches greedy merging performance within 5.5% at compression factors of four and above [S1, unofficial]. On full fits, cost-aware merging outperforms mass-ranked culling regardless of amplitude refit; on sparse subsamples, culling wins in the median [S1, unofficial].
**Strengths:** Provides a principled way to reduce splat count while minimizing L2 error. Closed-form solutions for small groups (e.g., 2 splats) are efficient.
**Limitations:** Partitioning can be computationally expensive for large $N$. The method assumes that the merged splat is a valid approximation, which may not hold for highly heterogeneous groups.

## Background
This method originates from the Luxar project, which focuses on Gaussian splatting for microscopy and scalable interactive web visualization of multidimensional scientific data [S1, unofficial]. It was developed to address the need for efficient LOD management in large-scale 3D Gaussian splatting scenes, particularly in scientific visualization contexts where data fidelity is critical [S1, unofficial]. The technique is detailed in the supplementary documents of the Luxar paper, specifically "Substitutive levels of detail for Gaussian splats" [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
