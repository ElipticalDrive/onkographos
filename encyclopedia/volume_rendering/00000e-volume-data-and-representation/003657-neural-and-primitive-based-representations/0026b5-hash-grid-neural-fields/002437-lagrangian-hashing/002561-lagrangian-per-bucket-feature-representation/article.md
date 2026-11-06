# Lagrangian Per-Bucket Feature Representation

Also called: Gaussian Mixture Feature, Lagrangian Hashing Feature

## Overview
This method defines the feature retrieval mechanism for the deepest levels of a hierarchical hash grid in Lagrangian Hashing. Instead of storing static feature vectors in hash buckets (Eulerian approach), it computes features as a weighted sum of isotropic Gaussians centered at learnable means. This allows for continuous, position-dependent feature retrieval, distinguishing it from the static bucket lookups of InstantNGP [S1].

## How it works
**Inputs and Outputs:**
- Input: Query position $\mathbf{x}$ (dimension $D$, e.g., 2D for image fitting, 3D for NeRF).
- Output: Feature vector $\mathbf{F}(\mathbf{x})$ of dimension $F$.

**Data Structures:**
The representation is used at levels $l \ge L - \tilde{L}$ of a hierarchical hash grid with $L$ total levels. Each bucket at these levels stores $K$ Gaussian components. Each component $k$ consists of:
1.  Learnable mean vector $\boldsymbol{\mu}_k \in \mathbb{R}^D$.
2.  Fixed standard deviation $\sigma_k$ (scalar).
3.  Learnable feature vector $\mathbf{f}_k \in \mathbb{R}^F$.

**Parameters:**
- $K$: Number of Gaussian components per bucket. Typical value: $K=4$ [S1].
- $F$: Feature dimension size. Typical value: $F=2$ [S1].
- $L$: Total number of levels of detail (LoDs). Typical value: $L=16$ [S1].
- $\tilde{L}$: Number of Lagrangian levels (depth threshold). Not given in the sources.
- $\sigma_k$: Standard deviation. The source notes it is fixed, but training may optionally optimize it [S1].

**Algorithm:**
1.  Determine the hash bucket index $v$ for query $\mathbf{x}$ at level $l$.
2.  Retrieve the $K$ Gaussian components stored in bucket $v$.
3.  For each component $k \in \{1, \dots, K\}$:
    a. Compute the Gaussian probability density function (PDF) value $\mathcal{N}_k(\mathbf{x})$.
    b. Multiply the PDF value by the component's feature vector $\mathbf{f}_k$.
4.  Sum the results from all $K$ components to produce $\mathbf{F}(\mathbf{x})$.

**Formula:**
The feature vector $\mathbf{F}(\mathbf{x})$ is computed as:
$$
\mathbf{F}(\mathbf{x}) = \sum_{k=1}^{K} \mathcal{N}_k(\mathbf{x}) \cdot \mathbf{f}_k
$$
where the Gaussian PDF $\mathcal{N}_k(\mathbf{x})$ is:
$$
\mathcal{N}_k(\mathbf{x}) = \frac{1}{(2\pi)^{1/2} \sigma_k} \exp\left(-\frac{\|\mathbf{x} - \boldsymbol{\mu}_k\|_2^2}{2\sigma_k^2}\right)
$$
Here, $\|\cdot\|_2$ denotes the Euclidean norm, $\boldsymbol{\mu}_k$ is the mean, and $\sigma_k$ is the standard deviation of the $k$-th Gaussian [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lagrangian_per_bucket_feature(x, bucket_components, K, F):
    """
    Computes the feature vector for a query point x using Lagrangian per-bucket features.
    
    Args:
        x: Query position, shape (D,)
        bucket_components: List of K tuples, each containing (mu_k, sigma_k, f_k)
                           mu_k: shape (D,)
                           sigma_k: scalar
                           f_k: shape (F,)
        K: Number of Gaussian components
        F: Feature dimension
        
    Returns:
        Feature vector, shape (F,)
    """
    feature_sum = zeros(F)
    
    for k in range(K):
        mu_k, sigma_k, f_k = bucket_components[k]
        
        # Compute squared Euclidean distance
        diff = x - mu_k
        dist_sq = dot(diff, diff)
        
        # Compute Gaussian PDF value
        # Note: (2*pi)**0.5 is sqrt(2*pi)
        gaussian_weight = (1.0 / (sqrt(2.0 * pi) * sigma_k)) * exp(-dist_sq / (2.0 * sigma_k**2))
        
        # Add weighted feature to sum
        feature_sum += gaussian_weight * f_k
        
    return feature_sum
```

**Edge Cases and Pitfalls:**
- **Vanishing Gradients:** If the mean $\boldsymbol{\mu}_k$ is too far from the query point $\mathbf{x}$ relative to $\sigma_k$, the gradient of the Gaussian with respect to $\boldsymbol{\mu}_k$ becomes negligible during back-propagation, interfering with optimization [S1]. This is analogous to the "dead neuron" problem in Gaussian Mixture Models [S1].
- **Boundary Conditions:** The method assumes the query point $\mathbf{x}$ is within the domain covered by the Gaussians. Points far from all means will yield near-zero features [S1].

## Subtopics

### [Lagrangian Hashing](3242-lagrangian-hashing.md)
The parent method that combines this representation with Eulerian hash grids. Use this when memory efficiency and high-frequency detail are required in novel view synthesis.

### [InstantNGP](3241-instantngp.md)
The baseline Eulerian method. Use this when simpler implementation is preferred and continuous feature interpolation is not critical.

## Operating characteristics
- **Time Complexity:** $O(K \cdot D)$ per query, where $K$ is the number of Gaussians and $D$ is the input dimension. This is slightly higher than the $O(1)$ lookup of static Eulerian features but remains efficient for small $K$ [S1].
- **Memory Complexity:** Each bucket stores $K \times (D + 1 + F)$ parameters. For $K=4, D=3, F=2$, this is $4 \times 6 = 24$ floats per bucket, compared to $F=2$ for Eulerian. However, the Lagrangian representation can achieve similar quality with fewer buckets or levels due to its continuous nature [S1].
- **Performance:** On gigapixel image fitting tasks, this method achieves comparable PSNR to InstantNGP with fewer parameters when using $B=2^{17}$ or $2^{18}$ buckets [S1].
- **Strengths:** Continuous feature retrieval allows for smoother gradients and better handling of high-frequency details without aliasing artifacts common in discrete grids [S1].
- **Limitations:** Requires careful initialization and guidance (e.g., guidance loss) to prevent vanishing gradients during training [S1].

## Background
The Lagrangian Per-Bucket Feature Representation was introduced in "Lagrangian Hashing for Compressed Neural Field Representations" [S1]. It draws inspiration from the Expectation-Maximization (EM) algorithm for Gaussian Mixture Models, specifically addressing the issue of vanishing gradients by guiding Gaussians towards surfaces during training [S1]. The method was implemented using the nerfacc framework for NeRF and Kaolin-wisp for 2D image fitting [S1].

## Sources
[S1] Lagrangian Hashing for Compressed Neural Field Representations, https://doi.org/10.1007/978-3-031-73383-3_11, fetched 2026-10-26
