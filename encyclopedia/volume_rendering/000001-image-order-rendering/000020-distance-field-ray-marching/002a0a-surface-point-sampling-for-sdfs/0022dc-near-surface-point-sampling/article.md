# Near-Surface Point Sampling

Also called: Near-surface perturbation, Gaussian surface perturbation

## Overview
Near-Surface Point Sampling generates training data points in the immediate vicinity of a mesh surface by perturbing existing surface samples with Gaussian noise. It complements uniform and pure surface sampling to provide the neural network with information about the local signed distance field (SDF) gradient and surface continuity, which is critical for learning smooth implicit representations [S1]. It differs from [Distance Field Foot Point Generation](6946-distance-field-foot-point-generation.md), which projects grid voxels to the surface, and [Primary Edge Sampling](7168-primary-edge-sampling.md), which targets geometric singularities; instead, it focuses on the continuous surface manifold itself [S1].

## How it works
**Inputs:**
- A set of $N$ surface sample points $P = \{p_1, \dots, p_N\}$, where each $p_i \in \mathbb{R}^3$ lies on the mesh surface.
- A noise standard deviation $\sigma$.

**Outputs:**
- A set of perturbed points $P' = \{p'_1, \dots, p'_N\}$, where each $p'_i$ is a near-surface sample.

**Data Structures:**
- Points are stored as 3D vectors (float32). No additional metadata is required for the sampling step itself.

**Steps:**
1. For each surface point $p_i \in P$:
   a. Generate a random 3D vector $n_i \in \mathbb{R}^3$ where each component is drawn from a standard normal distribution $\mathcal{N}(0, 1)$.
   b. Scale the noise vector by $\sigma$: $v_i = \sigma \cdot n_i$.
   c. Compute the perturbed point: $p'_i = p_i + v_i$.
2. Return the set $P'$.

**Parameters:**
- $\sigma$: Standard deviation of the Gaussian noise. The standard value used in the source is $\sigma = 0.01$ [S1].

**Edge Cases and Pitfalls:**
- **Precision:** Ensure $\sigma$ is consistent with the scale of the mesh. If the mesh is not normalized to a unit bounding box, $\sigma=0.01$ may be too large or too small.
- **Boundary:** The method does not clamp points to the surface or the bounding volume. Points may fall outside the region of interest, which is acceptable as the SDF evaluation will handle them (returning large distances) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def near_surface_point_sampling(surface_points: np.ndarray, sigma: float = 0.01) -> np.ndarray:
    """
    Perturb surface points with Gaussian noise.
    
    Args:
        surface_points: Array of shape (N, 3) containing 3D coordinates of surface points.
        sigma: Standard deviation of the Gaussian noise.
        
    Returns:
        Array of shape (N, 3) containing perturbed near-surface points.
    """
    # Generate random noise from standard normal distribution
    noise = np.random.normal(loc=0.0, scale=1.0, size=surface_points.shape)
    
    # Scale noise by sigma
    scaled_noise = noise * sigma
    
    # Add noise to surface points
    perturbed_points = surface_points + scaled_noise
    
    return perturbed_points
```

## Subtopics

### [Distance Field Foot Point Generation](6946-distance-field-foot-point-generation.md)
Generates points by projecting grid voxels onto the isosurface. Use when you need samples derived from the volume grid structure rather than mesh vertices.

### [Primary Edge Sampling](7168-primary-edge-sampling.md)
Samples points where the surface normal is perpendicular to the view vector. Use when prioritizing geometric features and edges over the smooth surface manifold.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of surface samples. The operation is a simple vector addition.
- **Memory Complexity:** $O(N)$ to store the perturbed points.
- **Strengths:** Simple to implement; provides dense coverage near the surface without requiring complex geometric queries.
- **Limitations:** Does not account for surface curvature; points are distributed isotropically in space, not along the surface normal. This may lead to inefficient sampling if the SDF varies rapidly in specific directions.
- **When to use:** As part of a mixed sampling strategy (e.g., 2:2:1 split with surface and uniform samples) to improve SDF learning stability [S1].

## Background
The method was introduced in the context of Neural Geometric Level of Detail (N-GLOD) for real-time rendering with implicit 3D shapes [S1]. It is used during the training phase of the neural network to help the model learn the signed distance function accurately near the surface. The approach builds on the idea that perturbing surface points helps the network generalize the distance field in the immediate neighborhood, reducing artifacts and improving convergence [S1].

## Sources
[S1] Neural Geometric Level of Detail: Real-time Rendering with Implicit 3D Shapes, https://doi.org/10.1109/cvpr46437.2021.01120, fetched 2026-10-26
