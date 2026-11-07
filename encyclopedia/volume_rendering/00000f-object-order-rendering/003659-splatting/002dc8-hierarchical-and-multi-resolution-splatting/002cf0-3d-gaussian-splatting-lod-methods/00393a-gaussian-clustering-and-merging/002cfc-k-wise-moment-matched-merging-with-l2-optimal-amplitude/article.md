# K-wise moment-matched merging with L2-optimal amplitude

Also called: cost-aware merging, substitutive LOD merging

## Overview
K-wise moment-matched merging with L2-optimal amplitude is a level-of-detail (LOD) technique that partitions a set of $N$ 3D Gaussian splats into disjoint groups of size $K$ (or variable sizes) and replaces each group with a single representative splat [S1, unofficial]. It solves the problem of reducing splat count while minimizing the integrated squared error (L2) between the original and merged representations [S1, unofficial]. It differs from mass-ranked culling by retaining spatial coverage through moment matching rather than discarding low-weight splats, and from simple centroid merging by optimizing the amplitude $\alpha'$ to minimize the L2 error rather than summing amplitudes [S1, unofficial].

## How it works
The method builds on the shared principles of Gaussian clustering and merging [8436-gaussian-clustering-and-merging.md].

**Inputs and Outputs:**
- **Input:** A set of $N$ splats $G_{total} = \{ (\mathbf{c}_i, \mathbf{\Sigma}_i, \alpha_i) \}_{i=1}^N$.
- **Output:** A reduced set of $M$ splats, where $M \ll N$.

**Data Structures:**
Each splat $i$ is defined by:
- Position $\mathbf{c}_i \in \mathbb{R}^3$.
- Covariance matrix $\mathbf{\Sigma}_i \in \mathbb{R}^{3 \times 3}$ (symmetric positive definite).
- Amplitude $\alpha_i \in \mathbb{R}$ (opacity/intensity weight).

**Partitioning:**
Splat groups are formed using one of three partitioning algorithms [S1, unofficial]:
1.  **Greedy Hierarchical Merging:** Iteratively merges the pair of splats (or clusters) with the lowest merging cost until the target count is reached [S1, unofficial]. This is analogous to the Runnalls algorithm [S1, unofficial].
2.  **Cost-Increment Relocation:** A more efficient alternative that matches greedy merging performance within 5.5% at compression factors of four and above [S1, unofficial].
3.  **Joint Relaxation:** A method that relaxes the discrete assignment problem to find optimal groupings [S1, unofficial].

**Merging Step:**
For a group $G$, the merged parameters $(\mathbf{c}', \mathbf{\Sigma}', \alpha')$ are computed as follows:

1.  **Weighted Moments:**
    Compute the total weight and first/second order moments:
    $$ W = \sum_{i \in G} \alpha_i $$
    $$ \mathbf{c}' = \frac{1}{W} \sum_{i \in G} \alpha_i \mathbf{c}_i $$
    $$ \mathbf{M}' = \sum_{i \in G} \alpha_i (\mathbf{c}_i \mathbf{c}_i^T + \mathbf{\Sigma}_i) $$

2.  **Covariance:**
    Derive the merged covariance matrix:
    $$ \mathbf{\Sigma}' = \mathbf{M}' - W \mathbf{c}' \mathbf{c}'^T $$
    $\mathbf{\Sigma}'$ must be symmetrized. If it is not positive definite, regularization (e.g., adding $\epsilon \mathbf{I}$) is required [G].

3.  **Amplitude Optimization:**
    The amplitude $\alpha'$ is optimized to minimize the L2 integrated squared error between the original group and the single representative splat. For the amplitude-optimized moment-matched template, the relative merging error $\epsilon$ has a geometric interpretation related to the angle $\alpha$ between the splats in the function space, where $\epsilon = \sin^2 \alpha$ [S1, unofficial]. In the general K-wise case, $\alpha'$ is solved analytically or numerically to minimize $\int \| \sum_{i \in G} f_i(\mathbf{x}) - \alpha' f'(\mathbf{x}) \|^2 d\mathbf{x}$, where $f$ are the Gaussian functions [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def merge_group(splats):
    # splats: list of tuples (c, Sigma, alpha)
    
    # 1. Compute Weighted Moments
    W = sum(s.alpha for s in splats)
    if W == 0:
        return None # Handle empty or zero-weight group
    
    c_prime = sum(s.alpha * s.c for s in splats) / W
    M_prime = sum(s.alpha * (np.outer(s.c, s.c) + s.Sigma) for s in splats)
    
    # 2. Compute Covariance
    Sigma_prime = M_prime - W * np.outer(c_prime, c_prime)
    Sigma_prime = 0.5 * (Sigma_prime + Sigma_prime.T) # Symmetrize
    
    # Check positive definiteness and regularize if needed [G]
    eigvals = np.linalg.eigvalsh(Sigma_prime)
    if np.any(eigvals <= 0):
        Sigma_prime += (abs(np.min(eigvals)) + 1e-6) * np.eye(3)
        
    # 3. Optimize Amplitude for L2 Error
    # The optimal alpha' minimizes the L2 error.
    # For two splats, this relates to the geometric angle alpha where error = sin^2(alpha).
    # For K splats, alpha' is derived from the projection of the sum of splats onto the template shape.
    # Assuming the template shape is fixed by c' and Sigma', alpha' is:
    # alpha' = (Sum of inner products) / (Norm of template squared)
    # However, the source specifies L2-optimal amplitude without explicit closed form for K>2 in the main text.
    # We use the property that for moment-matched merging, the amplitude is often set to W 
    # if shapes are identical, but here shapes differ.
    # Based on [S1, unofficial], the amplitude is optimized. A common L2-optimal scalar 
    # for approximating a sum of functions f_i by alpha * f' is:
    # alpha' = <Sum f_i, f'> / <f', f'>
    # This requires integrating the product of Gaussians.
    
    # Simplified implementation for L2-optimal amplitude [G]:
    # This integral is complex. In practice, for substitutive LOD, alpha' is often 
    # solved via the specific cost metric. The source mentions "L2-optimal amplitude".
    # We assume a standard L2 projection.
    
    # Note: Exact closed-form for K-wise L2-optimal amplitude is not explicitly detailed 
    # in the provided notes beyond the 2-splat geometric interpretation. 
    # We default to the weighted sum if optimization is not explicitly coded, 
    # but the method name implies optimization.
    # Given the constraint to implement from text, and the lack of explicit K-wise formula 
    # in the notes, we note the optimization goal.
    
    # For the purpose of this implementation guide, if the specific K-wise L2 formula 
    # is not provided, the amplitude is often initialized to W and refined.
    # However, the source [S1, unofficial] highlights "L2-optimal".
    # We will use the weighted sum as a baseline if the specific integral is not derivable 
    # from the provided text, but mark it as a gap.
    
    # Actually, the source [S1, unofficial] mentions "Analytical Merging... Closed-form L2...".
    # This implies a closed form exists. For K splats, it is the solution to the linear system 
    # derived from the L2 norm.
    
    # Let's assume the standard L2 projection coefficient:
    # alpha' = (Sum_i alpha_i * overlap(i, prime)) / overlap(prime, prime)
    # where overlap(a,b) is the integral of the product of two Gaussians.
    
    overlap_prime_prime = integral_gaussian_product(Sigma_prime, c_prime, Sigma_prime, c_prime)
    numerator = 0
    for s in splats:
        overlap_s_prime = integral_gaussian_product(s.Sigma, s.c, Sigma_prime, c_prime)
        numerator += s.alpha * overlap_s_prime
        
    alpha_prime = numerator / overlap_prime_prime
    
    return (c_prime, Sigma_prime, alpha_prime)

def integral_gaussian_product(Sigma1, c1, Sigma2, c2):
    # Integral of N(x; c1, Sigma1) * N(x; c2, Sigma2) over R^3
    # Result is N(c1; c2, Sigma1+Sigma2) * (2*pi)^(3/2) * det(Sigma1+Sigma2)^(-1/2) ?
    # Standard formula:
    # \int N(x; m1, S1) N(x; m2, S2) dx = N(m1; m2, S1+S2)
    # = (2*pi)^(-3/2) det(S1+S2)^(-1/2) exp(-0.5 (m1-m2)^T (S1+S2)^-1 (m1-m2))
    # Note: The Gaussian normalization constants are included in the splat definition usually.
    # If alpha is the peak value, the integral is alpha * (2*pi)^(3/2) det(Sigma)^(1/2).
    # The product integral is:
    # (2*pi)^(-3) det(S1)^(-1/2) det(S2)^(-1/2) \int exp(...)
    # = (2*pi)^(-3/2) det(S1+S2)^(-1/2) exp(-0.5 (c1-c2)^T (S1+S2)^-1 (c1-c2))
    
    S_sum = Sigma1 + Sigma2
    det_S_sum = np.linalg.det(S_sum)
    diff = c1 - c2
    inv_S_sum = np.linalg.inv(S_sum)
    exponent = -0.5 * diff.T @ inv_S_sum @ diff
    
    return (2 * np.pi)**(-1.5) * det_S_sum**(-0.5) * np.exp(exponent)
```

## Subtopics

### Analytical Merging of Gaussian Splats
A closed-form framework for merging two Gaussian splats using L2 integrated squared-error, providing a geometric interpretation of the relative merging error [S1, unofficial]. The relative merging error is $\epsilon = \sin^2 \alpha$, where $\alpha$ is the geometric angle parameter in the error derivation [S1, unofficial]. It is implemented in Luxar for merging Gaussian splats during fitting or LOD reduction [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the partitioning algorithm. Greedy hierarchical merging is $O(N^2)$ or $O(N \log N)$ with a heap. Cost-increment relocation is faster, matching greedy performance within 5.5% at compression factors $\ge 4$ [S1, unofficial].
- **Memory Complexity:** $O(N)$ for the input and $O(M)$ for the output, where $M$ is the target splat count.
- **Performance:** On full fits, cost-aware merging outperforms mass-ranked culling regardless of amplitude refit [S1, unofficial]. On sparse subsamples, culling wins in the median [S1, unofficial].
- **Strengths:** Provides a faithful surrogate for global error once spatial partition separates bins (single trials deviate by tens of percent at moderate separation) [S1, unofficial].
- **Limitations:** Requires careful handling of positive definiteness for covariance matrices.

## Background
This method originates from the Luxar project, which provides Gaussian splatting for microscopy and scalable interactive web visualization of multidimensional scientific data [S1, unofficial]. It is described in Supplementary Document 8 of the Luxar documentation, which details the cost-metric design space for substitutive LOD problems [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
