# Soft Voronoi Decomposition

Also called: Differentiable Voronoi Diagram, SoftVoronoi

## Overview
Soft Voronoi Decomposition is a differentiable spatial partitioning method that decomposes a scene into $N$ regions (Voronoi cells) defined by a set of $N$ sites $\phi \in \mathbb{R}^{N \times 3}$ [S1]. It solves the problem of creating a smooth, differentiable approximation of a hard Voronoi diagram, allowing gradients to flow to the site positions $\phi$ during training while maintaining spatial coherence [S1]. Unlike MLP-based decomposition methods which use softmax activations and suffer from random memory access patterns during integration, Soft Voronoi provides cache-friendly memory access and accelerates inference by ensuring that for any query point $x$, only one rendering head needs to be evaluated in the limit [S1].

## How it works
**Inputs:**
- A set of $N$ Voronoi sites $\phi = [\phi_1, \dots, \phi_N]$, where each $\phi_n \in \mathbb{R}^3$. The sites are typically scaled to the range $[-1, 1]$ to prevent optimizer magnitude invariance issues [S1].
- A temperature parameter $\beta \in \mathbb{R}^+$, which controls the softness of the approximation [S1].

**Outputs:**
- A set of weights $w_{\phi}(x) = [w_{\phi_1}(x), \dots, w_{\phi_N}(x)]$ for a query point $x \in \mathbb{R}^3$, where $\sum_{n=1}^N w_{\phi_n}(x) = 1$ [S1].

**Data Structures:**
- The sites $\phi$ are stored as a dense array of shape $(N, 3)$.
- The weights are computed on-the-fly for each query point $x$.

**Parameters:**
- $\beta$: Temperature parameter. Typical value: $10^9$ [S1].
- Schedule: $\beta$ is not constant; it is scheduled over the training process [S1].

**Steps:**
1. **Initialize $\beta$**: Start training with a sufficiently low value of $\beta$ such that $w_i(x) \approx w_j(x)$ for all $i, j$ and all $x$ [S1].
2. **Schedule $\beta$**: Exponentially increase $\beta$ during training until it reaches a high value (e.g., $10^9$) [S1]. This ensures that as training progresses, the decomposition becomes indistinguishable from a hard Voronoi diagram, where $w_i(x) \approx 1$ or $w_i(x) \approx 0$ for all $i$ [S1].
3. **Compute Weights**: For a query point $x$, compute the weight for each site $n$ using the formula below [S1].
4. **Select Head**: In inference, identify the site $n^*$ with the maximum weight $w_{\phi_{n^*}}(x)$. Evaluate only the rendering head corresponding to $n^*$ for point $x$ [S1].

**Formula:**
The weight $w_{\phi_n}(x)$ for site $n$ at query point $x$ is defined as [S1]:

$$
w_{\phi_n}(x) = \frac{\exp(-\beta \|x - \phi_n\|^2)}{\sum_{j=1}^{N} \exp(-\beta \|x - \phi_j\|^2)}
$$

where $\| \cdot \|$ denotes the Euclidean distance [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def soft_voronoi_weights(x, sites, beta):
    """
    Compute Soft Voronoi weights for a single query point x.
    
    Args:
        x: numpy array of shape (3,), query point.
        sites: numpy array of shape (N, 3), Voronoi sites.
        beta: float, temperature parameter.
        
    Returns:
        weights: numpy array of shape (N,), weights for each site.
    """
    # Calculate squared Euclidean distances from x to each site
    # sites shape: (N, 3), x shape: (3,)
    # diff shape: (N, 3)
    diff = sites - x
    
    # Squared L2 norm for each site
    # dist_sq shape: (N,)
    dist_sq = np.sum(diff ** 2, axis=1)
    
    # Compute unnormalized weights
    # exp(-beta * dist_sq)
    unnormalized_weights = np.exp(-beta * dist_sq)
    
    # Normalize weights to sum to 1
    sum_weights = np.sum(unnormalized_weights)
    
    # Avoid division by zero if all weights are extremely small (numerical underflow)
    if sum_weights == 0:
        return np.ones(len(sites)) / len(sites)
        
    weights = unnormalized_weights / sum_weights
    
    return weights

def soft_voronoi_inference(x, sites, beta):
    """
    Select the rendering head for query point x using Soft Voronoi.
    
    Args:
        x: numpy array of shape (3,), query point.
        sites: numpy array of shape (N, 3), Voronoi sites.
        beta: float, temperature parameter.
        
    Returns:
        best_site_index: int, index of the site with the highest weight.
    """
    weights = soft_voronoi_weights(x, sites, beta)
    best_site_index = np.argmax(weights)
    return best_site_index
```

**Edge Cases and Pitfalls:**
- If $\beta$ is too low, the decomposition is not spatially distinct, leading to poor scene separation [S1].
- If $\beta$ is too high too early in training, gradients may be zero or unstable because the weights become effectively one-hot, preventing gradient flow to the site positions $\phi$ [S1]. The exponential scheduling of $\beta$ resolves this [S1].
- The magnitude of the scene coordinates must be scaled (e.g., $\phi \in [-1, 1]^{N \times 3}$) to prevent training instability when using optimizers like Adam [S1].

## Subtopics

### [Adjustable Center Point for Voronoi Partitioning](9851-adjustable-center-point-for-voronoi-partitioning.md)
This method modifies the standard Voronoi partitioning within a barycentric simplex (triangle or tetrahedron) by shifting the central "Y-point" or centroid to arbitrary locations. It solves the problem of inaccurate material interface representation in cases where the uniform distribution assumption fails, such as T-intersections or specific material distributions. Unlike standard Soft Voronoi Decomposition which assumes symmetric sector subdivision, this approach allows the center point $c$ to be adjusted to better weigh materials about the junction, ensuring planar separating surfaces remain consistent.

## Operating characteristics
**Time Complexity:**
- Computing weights for a single point $x$ requires $O(N)$ operations to calculate distances and exponentials [S1].
- Inference is accelerated because only one rendering head needs to be evaluated per point, unlike MLP-based methods where all heads might be accessed [S1].

**Memory Complexity:**
- Storage of sites $\phi$ is $O(N)$ [S1].
- No additional large data structures are required for the decomposition itself [S1].

**Measured Performance:**
- Soft Voronoi decomposition allows for efficient compositing of rendering heads using the Painter’s Algorithm [S1].
- Compared to MLP-based decomposition, Soft Voronoi runtime cost grows much slower as the number of heads $N$ increases, due to cache-friendly memory access patterns and the ability to skip evaluating most heads [S1].

**Strengths and Limitations:**
- **Strengths:** Differentiable with respect to site positions $\phi$, spatially coherent, cache-friendly, compatible with Painter’s Algorithm [S1].
- **Limitations:** Requires careful scheduling of $\beta$ to balance differentiability and spatial distinctness [S1].

**When to Use:**
- Use when differentiable scene decomposition is required for training neural radiance fields or similar models, and when spatial coherence and efficient inference are priorities [S1].

**When Not to Use:**
- Avoid if the scene does not benefit from spatial partitioning or if the overhead of managing $N$ sites is prohibitive for the specific hardware [S1].

## Background
Soft Voronoi Decomposition was introduced in the context of Decomposed Radiance Fields (DeRF) [S1]. The method leverages the properties of Voronoi diagrams to create a balanced scene decomposition where all regions represent a similar amount of visual complexity [S1]. It was proposed as an alternative to naive decomposition strategies and MLP-based decompositions, which suffer from random memory access and inefficient scaling [S1]. The approach builds on the mathematical formulation of soft Voronoi diagrams using exponential weighting [S1].

## Sources
[S1] DeRF: Decomposed Radiance Fields, https://doi.org/10.1109/cvpr46437.2021.01393, fetched 2026-10-26
[S2] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
