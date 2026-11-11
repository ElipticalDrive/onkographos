# Lagrangian Per-Bucket Feature Representation

Also called: Gaussian Mixture Feature, Lagrangian Hashing Feature

## Overview
This method defines the feature retrieval mechanism for the deepest levels of a hierarchical hash grid in Lagrangian Hashing [S1]. Instead of storing static feature vectors in hash buckets, it computes features as a weighted sum of isotropic Gaussians centered at learnable means [S1]. This allows for continuous, position-dependent feature retrieval, distinguishing it from the static bucket lookups of InstantNGP [S1].

## How it works
**Inputs and Outputs:**
- Input: Query position $\mathbf{x}$ (dimension $D$, e.g., 2D for image fitting, 3D for NeRF) [S1].
- Output: Feature vector $\mathbf{F}(\mathbf{x})$ of dimension $F$ [S1].

**Data Structures:**
The representation is used at levels $l \ge L - \tilde{L}$ of a hierarchical hash grid with $L$ total levels [S1]. Each bucket at these levels stores $K$ Gaussian components [S1]. Each component $k$ consists of: a learnable mean vector $\boldsymbol{\mu}_k \in \mathbb{R}^D$ [S1], a fixed standard deviation $\sigma_k$ (scalar) [S1], and a learnable feature vector $\mathbf{f}_k \in \mathbb{R}^F$ [S1].

**Numbered Steps:**
1. Determine the hash bucket index $v$ for query $\mathbf{x}$ at level $l$ [S1].
2. Retrieve the $K$ Gaussian components stored in bucket $v$ [S1].
3. For each component $k \in \{1, \dots, K\}$:
   a. Compute the Gaussian probability density function (PDF) value $\mathcal{N}_k(\mathbf{x})$ [S1].
   b. Multiply the PDF value by the component's feature vector $\mathbf{f}_k$ [S1].
4. Sum the results from all $K$ components to produce $\mathbf{F}(\mathbf{x})$ [S1].

**Formulas:**
The feature vector $\mathbf{F}(\mathbf{x})$ is computed as:
$$
\mathbf{F}(\mathbf{x}) = \sum_{k=1}^{K} \mathcal{N}_k(\mathbf{x}) \cdot \mathbf{f}_k
$$
where the Gaussian PDF $\mathcal{N}_k(\mathbf{x})$ is:
$$
\mathcal{N}_k(\mathbf{x}) = \frac{1}{(2\pi)^{1/2} \sigma_k} \exp\left(-\frac{\|\mathbf{x} - \boldsymbol{\mu}_k\|_2^2}{2\sigma_k^2}\right)
$$
Here, $\|\cdot\|_2$ denotes the Euclidean norm, $\boldsymbol{\mu}_k$ is the mean, and $\sigma_k$ is the standard deviation of the $k$-th Gaussian [S1].

**Parameters:**
- $K$: Number of Gaussian components per bucket. Typical value: $K=4$ [S1].
- $F$: Feature dimension size. Typical value: $F=2$ [S1].
- $L$: Total number of levels of detail (LoDs). Typical value: $L=16$ [S1].
- $\tilde{L}$: Number of Lagrangian levels (depth threshold). Not given in the sources.
- $B$: Size of the hash table (number of buckets). Typical values: $2^{17}$ or $2^{18}$ [S1].

**Edge Cases and Pitfalls:**
- **Vanishing Gradients:** If the mean $\boldsymbol{\mu}_k$ is too far from the query point $\mathbf{x}$ relative to $\sigma_k$, the gradient of the Gaussian with respect to $\boldsymbol{\mu}_k$ becomes negligible during back-propagation, interfering with optimization [S1].
- **Boundary Conditions:** The method assumes the query point $\mathbf{x}$ is within the domain covered by the Gaussians; points far from all means will yield near-zero features [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def zeros(n):
    """Returns a list of n zeros."""
    return [0.0] * n

def lagrangian_per_bucket_feature(x, bucket_components, K, F):
    """
    Computes the feature vector for a query point x using Lagrangian per-bucket features.
    
    Args:
        x: Query position, list of floats, length D
        bucket_components: List of K tuples, each containing (mu_k, sigma_k, f_k)
                           mu_k: list of floats, length D
                           sigma_k: float
                           f_k: list of floats, length F
        K: Number of Gaussian components
        F: Feature dimension
        
    Returns:
        Feature vector, list of floats, length F
    """
    feature_sum = zeros(F)
    
    for k in range(K):
        mu_k, sigma_k, f_k = bucket_components[k]
        
        # Compute squared Euclidean distance
        diff = [x_i - mu_i for x_i, mu_i in zip(x, mu_k)]
        dist_sq = sum(d * d for d in diff)
        
        # Compute Gaussian PDF value matching the source formula
        gaussian_weight = (1.0 / (math.sqrt(2.0 * math.pi) * sigma_k)) * math.exp(-dist_sq / (2.0 * sigma_k**2))
        
        # Add weighted feature to sum
        feature_sum = [feature_sum[i] + gaussian_weight * f_k[i] for i in range(F)]
        
    return feature_sum
```

## Operating characteristics
- **Time Complexity:** $O(K \cdot D)$ per query, where $K$ is the number of Gaussians and $D$ is the input dimension [G].
- **Memory Complexity:** Each bucket stores $K \times (D + 1 + F)$ parameters [G]. For $K=4, D=3, F=2$, this requires 24 floats per bucket, compared to $F=2$ for Eulerian [G].
- **Configuration:** The method uses $B=2^{17}$ or $2^{18}$ buckets and an MLP decoder with one hidden layer of 64 neurons [S1].
- **Training:** It is trained with an L2 reconstruction loss for 2D fitting and a guidance loss to prevent vanishing gradients during optimization [S1].
- **Evaluation:** Performance was measured on the Synthetic blender and Tanks & Temples datasets [S1].

## Background
The representation was introduced in "Lagrangian Hashing for Compressed Neural Field Representations" [S1]. It draws inspiration from the Expectation-Maximization (EM) algorithm for Gaussian Mixture Models to address vanishing gradients by guiding Gaussians towards surfaces during training [S1]. The implementation used the nerfacc framework for NeRF and Kaolin-wisp for 2D image fitting [S1].

## Sources
[S1] Lagrangian Hashing for Compressed Neural Field Representations, https://doi.org/10.1007/978-3-031-73383-3_11, fetched 2026-10-26
