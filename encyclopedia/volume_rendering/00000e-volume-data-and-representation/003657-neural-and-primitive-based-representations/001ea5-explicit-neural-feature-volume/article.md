# Explicit Neural Feature Volume

Also called: Explicit Neural Feature Tensor

## Overview
Explicit Neural Feature Volume stores neural features as a large, directly addressable tensor, serving as the primary data structure for neural adaptive tomography [S1]. It solves the problem of balancing reconstruction fidelity with memory efficiency in limited-angle tomography, where hybrid explicit-implicit models suffer from increased blur and training costs [S1]. It differs from hybrid approaches like ACORN by avoiding implicit compression of feature volumes, which otherwise worsens PSNR in the missing wedge direction [S1].

## How it works
**Inputs and Outputs:**
*   Input: A set of 2D projection images and corresponding camera poses.
*   Output: A 3D density volume reconstructed via volume rendering.

**Data Structures:**
*   **Feature Tensor:** An explicit 3D grid storing feature vectors [S1].
*   **Sparse Representation:** The tensor is organized hierarchically (e.g., octree) to enable empty space skipping and structural refinement (merging, splitting, and culling leaf nodes) [S1].

**Steps:**
1.  **Storage:** Store neural features explicitly as a large tensor, rather than compressing them into an implicit neural network [S1].
2.  **Decoding:** Pass sampled feature vectors through a decoder network to produce scalar density values [S1]. The decoder acts as a regularizer to improve reconstruction in difficult conditions [S1].
3.  **Rendering:** Compute accumulated color and opacity along rays using the decoded density values [G]. The opacity at each step follows $\alpha = 1 - e^{-\sigma \Delta t}$, where $\sigma$ is the decoded density and $\Delta t$ is the step size [G].
4.  **Refinement:** Periodically (every few epochs), perform octree optimization by merging, splitting, and culling leaf nodes to reduce reconstruction time via empty space skipping [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_ray(ray_origin, ray_direction, feature_grid, decoder_network, step_size, grid_min, grid_max):
    # March along ray within grid bounds
    t_min = max(0.0, (grid_min - ray_origin) / ray_direction)
    t_max = min(grid_max, (grid_max - ray_origin) / ray_direction)
    num_steps = int((t_max - t_min) / step_size)
    
    accumulated_color = 0.0
    T = 1.0  # Transmittance
    
    for i in range(num_steps):
        t = t_min + i * step_size
        pos = ray_origin + ray_direction * t
        
        # Sample feature vector from explicit grid (sparse octree traversal omitted for brevity) [G]
        feature_vec = feature_grid.sample(pos)
        
        # Decode feature vector to scalar density
        density = float(decoder_network(feature_vec))
        
        # Compute alpha from density
        alpha = 1.0 - math.exp(-density * step_size)
        
        # Retrieve color (typically stored in grid or decoded separately) [G]
        color = float(feature_vec[0])
        
        # Composite
        weight = T * alpha
        accumulated_color += weight * color
        T *= (1.0 - alpha)
        
        if T < 1e-5:
            break
            
    return accumulated_color
```

**Parameters:**
*   **Refinement Frequency:** Every few epochs [S1].

**Edge Cases and Pitfalls:**
*   **Missing Wedge Blur:** In limited-angle tomography, reconstructions are naturally blurred in the direction orthogonal to the missing viewing direction [S1].
*   **Hybrid Model Degradation:** Using a hybrid explicit-implicit model worsens PSNR in limited-angle scenarios because the implicit network's regularization encourages additional blur instead of repairing the missing wedge artifacts [S1].
*   **Memory Consumption:** Hybrid models have higher *intermediate* memory consumption during training because all feature volumes must be decoded to trace rays for a single view, despite potentially smaller final model size [S1].
*   **Training Time:** Hybrid models dramatically increase training times [S1].

## Subtopics

### [Density Activation](2421-density-activation.md)
Density Activation converts raw neural feature outputs into valid, non-negative density values suitable for volume rendering [G]. It ensures that density values remain physically meaningful and differentiable for gradient-based optimization [G]. It differs from the parent [Explicit Neural Feature Volume](1838-explicit-neural-feature-volume.md) by specifically handling the final non-linear mapping step from feature space to scalar density, whereas the parent focuses on the storage and retrieval of the feature tensor itself [G].

## Operating characteristics
*   **Time Complexity:** Training time is lower than hybrid explicit-implicit models because it avoids the overhead of decoding an implicit network for every ray [S1]. Structural refinement (octree operations) adds overhead but reduces reconstruction time via empty space skipping [S1].
*   **Memory Complexity:** Intermediate memory consumption is lower than hybrid models because features are stored explicitly and do not require decoding an implicit network for ray tracing [S1]. Final model size is larger than hybrid models, but this is a trade-off for better reconstruction quality [S1].
*   **Performance:** Achieves higher PSNR than hybrid explicit-implicit models, particularly in limited-angle tomography [S1].

## Background
The Explicit Neural Feature Volume method was introduced in the context of Neural Adaptive Tomography (NeAT) [S1]. It addresses limitations in hybrid explicit-implicit representations like ACORN, which were found to degrade performance in limited-angle tomography [S1]. The method emphasizes the benefits of explicit storage for feature tensors to maintain high-frequency details and avoid over-regularization [S1].

## Sources
[S1] NeAT: Neural Adaptive Tomography, https://doi.org/10.48550/arxiv.2202.02171, fetched 2026-10-26
