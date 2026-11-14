# Near-Surface Point Sampling

Also called: Near-surface perturbation, Gaussian surface perturbation

## Overview
Near-Surface Point Sampling generates training data points in the immediate vicinity of a mesh surface by perturbing existing surface samples with Gaussian noise. It solves the problem of insufficient volumetric coverage near the object boundary during neural implicit surface reconstruction, ensuring the network learns the signed distance field (SDF) gradient accurately without requiring expensive ray marching for every sample. It sets itself apart from [Surfel Parameterization from Mesh Sampling](4501-surfel-parameterization-from-mesh-sampling.md) by not requiring explicit surface parameterization or normal-based offsetting, and from [SDF-guided Sampling](7324-sdf-guided-sampling.md) by relying on simple stochastic perturbation rather than iterative SDF evaluation.

## How it works
**Inputs:**
- A set of uniform surface samples $S_{surf}$ on a mesh.
- A perturbation standard deviation $\sigma$.

**Outputs:**
- A set of near-surface samples $S_{near}$.

**Data Structures:**
- $S_{surf}$: List of 3D points $(x, y, z)$.
- $S_{near}$: List of 3D points $(x', y', z')$.

**Steps:**
1. Initialize an empty list $S_{near}$.
2. For each point $p \in S_{surf}$:
   a. Generate a random 3D vector $\mathbf{n} = (n_x, n_y, n_z)$ where each component is drawn from a standard normal distribution $\mathcal{N}(0, 1)$.
   b. Scale the vector by $\sigma$: $\mathbf{v} = \sigma \cdot \mathbf{n}$.
   c. Compute the perturbed point: $p' = p + \mathbf{v}$.
   d. Append $p'$ to $S_{near}$.

**Parameters:**
- $\sigma$: Standard deviation of the Gaussian noise. Typical value: $0.01$ [S1].

**Edge Cases and Pitfalls:**
- The noise is isotropic, meaning points may fall on either side of the surface. This is desirable for learning the signed distance field sign.
- If $\sigma$ is too large, samples may fall too far from the surface, reducing the effectiveness of the near-surface constraint.
- If $\sigma$ is too small, the samples may be numerically indistinguishable from the surface points, providing no volumetric gradient information.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def near_surface_point_sampling(surface_samples, sigma=0.01):
    """
    Generates near-surface points by perturbing surface samples with Gaussian noise.
    
    Args:
        surface_samples: numpy array of shape (N, 3) containing surface points.
        sigma: Standard deviation of the Gaussian noise.
        
    Returns:
        numpy array of shape (N, 3) containing near-surface points.
    """
    # Generate random noise from standard normal distribution
    noise = np.random.normal(0, 1, size=surface_samples.shape)
    
    # Scale noise by sigma
    perturbation = sigma * noise
    
    # Add perturbation to surface samples
    near_surface_samples = surface_samples + perturbation
    
    return near_surface_samples
```

## Subtopics

### [Distance Field Foot Point Generation](6946-distance-field-foot-point-generation.md)
A method to generate surface points (foot points) from a clamped, signed distance field volume (DFV) by projecting grid voxels onto the isosurface along the gradient direction. It differs from general scalar field foot point estimation by relying on the specific property that the gradient of a distance field is unit length, allowing for a single-step projection without iterative refinement. It builds on [Near-Surface Point Sampling](2910-near-surface-point-sampling.md) [G].

### [Primary Edge Sampling](7168-primary-edge-sampling.md)
Primary Edge Sampling generates sample points on the primary boundary curve $S$ of an implicit surface, which is defined as the intersection of the surface function $\phi$ and the orientation function $\psi$. It solves the problem of sampling visibility discontinuities (silhouettes) for implicit surfaces, which lack a finite set of edges like polygonal meshes. It sets itself apart from [Distance Field Foot Point Generation](6946-distance-field-foot-point-generation.md) by explicitly targeting the geometric condition where the surface normal is perpendicular to the view vector, rather than projecting points onto the surface. It differs from [Surface and Volume Training Position Sampling](9996-surface-and-volume-training-position-sampling.md) by focusing on boundary integrals for differentiable rendering rather than general training position generation.

### [Surface and Volume Training Position Sampling](9996-surface-and-volume-training-position-sampling.md)
A method to distribute training positions for neural implicit surface reconstruction by combining uniform volume samples, uniform surface samples, and perturbed surface samples to ensure coverage of the object and its immediate vicinity. It sets itself apart from siblings like [Distance Field Foot Point Generation](6946-distance-field-foot-point-generation.md) by explicitly mixing volumetric background data with surface-localized data, and from [Primary Edge Sampling](7168-primary-edge-sampling.md) by using area-weighted triangle selection rather than edge-centric heuristics.

### [Spot Noise (Motion Blurred Particles near Contour Surfaces)](13068-spot-noise-motion-blurred-particles-near-contour-surfaces.md)
Spot noise visualizes 3D velocity fields in a thickened region around a contour surface by rendering motion-blurred ellipsoidal particles, allowing for interactive rotation and animation using standard hardware graphics pipelines. Unlike methods that represent tangential flows on surfaces (such as stream surfaces) or require surface parameterization, this technique operates on contour surfaces of scalar functions that are generally unrelated to the velocity vectors, meaning the flow is not necessarily tangent to the surface. It builds on [Near-Surface Point Sampling](2910-near-surface-point-sampling.md) by using distance-field estimation to restrict particle placement to a specific band around the isosurface.

### [Surface Sampling for NeRF Rendering](14054-surface-sampling-for-nerf-rendering.md)
Generates random points on the surface of a 3D object (specifically on triangles) to be used as input for a neural network reflectance model.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of surface samples, as each sample undergoes a constant-time noise generation and addition.
- **Memory Complexity:** $O(N)$ to store the resulting near-surface points.
- **Measured Performance:** In the source, this method is used to generate 500,000 points per epoch for training on a NVIDIA Tesla V100 GPU [S1]. The perturbation step is negligible compared to the neural network forward pass.
- **Strengths:** Simple to implement, computationally cheap, provides isotropic coverage around the surface.
- **Limitations:** Does not guarantee points stay within a specific distance threshold (though $\sigma$ controls the spread), may generate points far from the surface if $\sigma$ is large.
- **When to use:** When training neural implicit surfaces and a mix of surface and near-surface points is required.
- **When not to use:** When precise control over the distance from the surface is required, or when anisotropic perturbation is needed.

## Background
Originated in the context of Neural Geometric Level of Detail (NGLOD) for real-time rendering with implicit 3D shapes [S1]. The method was introduced to provide a simple and effective way to sample points near the surface for training neural networks that predict signed distance fields. It builds on the idea that perturbing surface points with Gaussian noise provides a good approximation of the volumetric distribution near the surface without the computational cost of ray marching or complex geometric queries.

## Sources
[S1] Neural Geometric Level of Detail: Real-time Rendering with Implicit 3D Shapes, https://doi.org/10.1109/cvpr46437.2021.01120, fetched 2026-10-26
