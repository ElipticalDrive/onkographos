# NeRF Volumetric Rendering
Also called: NeRF Alpha Compositing, NeRF Rendering Equation

## Overview
NeRF Volumetric Rendering is the differentiable integration step that synthesizes a 2D image from a 3D radiance field by accumulating color and opacity along camera rays. It solves the problem of rendering continuous volumetric representations into discrete pixel values by approximating the line integral of the rendering equation. It is distinguished from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by its specific reliance on the alpha-compositing formulation derived from the original NeRF paper, which serves as the primary gradient source for optimizing the underlying neural network parameters.

## How it works
**Inputs and Outputs**
- **Input**: A set of $N_p$ samples along a ray $r$. For each sample $i$, the input includes the 3D position $\mathbf{x}_i$, the density $\sigma_i$ (or volume density), the color $\mathbf{c}_i$ (typically RGB), and the distance from the camera $t_i$.
- **Output**: The final pixel color $\hat{\mathbf{C}}(r)$ and the expected depth $D(r)$.

**Data Structures**
The method operates on arrays of size $N_p$:
- $\sigma$: Array of densities, shape $(N_p, 1)$ or scalar per sample.
- $\mathbf{c}$: Array of colors, shape $(N_p, 3)$.
- $t$: Array of depths/distances, shape $(N_p, 1)$.
- $\delta t$: Array of distances between consecutive samples, shape $(N_p, 1)$, where $\delta t_i = t_{i+1} - t_i$ (with $\delta t_{N_p} = 0$ or handled via boundary conditions).

**Formulas**
The rendering equation computes the accumulated transmittance $T_i$ and the alpha weight $\alpha_i$ for each sample:

$$
\alpha_i = 1 - \exp(-\sigma_i \delta t_i)
$$

$$
T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta t_j\right)
$$

The final color and depth are computed as weighted sums:

$$
\hat{\mathbf{C}}(r) = \sum_{i=1}^{N_p} T_i \alpha_i \mathbf{c}_i
$$

$$
D(r) = \sum_{i=1}^{N_p} T_i \alpha_i t_i
$$

Where:
- $T_i$ is the probability that the ray passes through all previous samples $1 \dots i-1$ without being absorbed.
- $\alpha_i$ is the probability that the ray is absorbed at sample $i$.
- $t_i$ is the distance of the $i$-th sample from the camera origin.
- $\delta t_i$ is the distance between sample $i$ and $i+1$.

**Parameters**
- $N_p$: Number of samples per ray. Typical values range from 48 to 256 for training, depending on the scene complexity and computational budget [S1].

**Edge Cases and Pitfalls**
- **Precision**: When $\sigma_i \delta t_i$ is very large, $\exp(-\sigma_i \delta t_i)$ underflows to 0, making $\alpha_i = 1$. This is correct for opaque surfaces but can cause gradient issues if not handled carefully.
- **Empty Space**: In regions with $\sigma_i \approx 0$, $T_i \approx 1$ and $\alpha_i \approx 0$, contributing nothing to the sum. This is efficient but requires sufficient sampling density near object boundaries to capture gradients.
- **Boundary Conditions**: The last sample's $\delta t$ is often undefined or zero. In practice, $\delta t_{N_p}$ is set to 0, making $\alpha_{N_p} = 0$, effectively ignoring the last sample's contribution to absorption but allowing its color to be included if $T_{N_p}$ is non-zero (though typically the last sample is just a terminator).

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_volumetric_rendering(sigma, colors, depths, delta_t):
    """
    Render a ray using NeRF volumetric rendering.
    
    Args:
        sigma: Array of densities, shape (N_p,)
        colors: Array of RGB colors, shape (N_p, 3)
        depths: Array of distances from camera, shape (N_p,)
        delta_t: Array of distances between samples, shape (N_p,)
        
    Returns:
        color: Final pixel color, shape (3,)
        depth: Expected depth, scalar
    """
    N_p = len(sigma)
    
    # Compute alpha: probability of absorption at each sample
    # alpha_i = 1 - exp(-sigma_i * delta_t_i)
    alpha = 1.0 - torch.exp(-sigma * delta_t)
    
    # Compute transmittance: probability of passing through all previous samples
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_t_j)
    # We compute the cumulative sum of sigma * delta_t
    accumulated_sigma_delta_t = torch.cumsum(sigma * delta_t, dim=0)
    
    # T_i corresponds to exp(-accumulated_sigma_delta_t_{i-1})
    # For i=0, T_0 = 1 (exp(0))
    # We shift the accumulated sum to the right by 1, filling the first element with 0
    # T = exp(-shifted_accumulated)
    
    # Create a tensor for shifted accumulation
    # accumulated_sigma_delta_t[:-1] gives sum for j=0 to i-1 for i > 0
    # For i=0, the sum is 0.
    
    # Efficiently compute T:
    # T[0] = 1.0
    # T[i] = exp(-sum_{j=0}^{i-1} sigma[j] * delta_t[j])
    
    # Using cumsum, the sum up to i-1 is accumulated_sigma_delta_t[i-1]
    # We can compute T by taking exp(-cumsum) and shifting
    
    cum_sigma_delta_t = torch.cumsum(sigma * delta_t, dim=0)
    # Shift right by 1, pad with 0 at the beginning
    # T_i = exp(-cum_sigma_delta_t_{i-1})
    # For i=0, cum_sigma_delta_t_{-1} is 0.
    
    # Create T array
    T = torch.zeros(N_p)
    T[0] = 1.0
    if N_p > 1:
        T[1:] = torch.exp(-cum_sigma_delta_t[:-1])
        
    # Compute weights: w_i = T_i * alpha_i
    weights = T * alpha
    
    # Final color: sum of w_i * c_i
    color = torch.sum(weights.unsqueeze(1) * colors, dim=0)
    
    # Final depth: sum of w_i * t_i
    depth = torch.sum(weights * depths)
    
    return color, depth
```

## Subtopics

### [Lightplane Renderer](16859-lightplane-renderer.md)
A memory-efficient renderer for neural 3D fields that reduces memory usage by four orders of magnitude compared to standard autograd renderers. It solves the bottleneck of storing intermediate values for backpropagation by fusing operations over casted rays instead of 3D points.

## Operating characteristics
- **Time Complexity**: $O(N_p)$ per ray, where $N_p$ is the number of samples. The total time is proportional to the number of rays (pixels) times $N_p$.
- **Memory Complexity**: $O(N_p)$ per ray for storing intermediate values (alpha, transmittance) during the forward pass. For backpropagation, gradients must be stored for all samples, leading to $O(N_p)$ memory per ray. This can be a significant bottleneck for high-resolution images and large $N_p$ [S1].
- **Measured Performance**: Standard NeRF rendering is computationally expensive due to the large number of samples required for smooth results. Performance scales linearly with $N_p$ and image resolution.
- **Strengths**: Differentiable, allowing end-to-end optimization of the radiance field. Simple to implement and understand.
- **Limitations**: High memory usage for backpropagation. Requires many samples to avoid aliasing and artifacts.
- **When to use**: When training a NeRF model and differentiable rendering is required.
- **When not to use**: When memory is constrained and $N_p$ is large; consider [Lightplane Renderer](16859-lightplane-renderer.md) or other memory-efficient alternatives.

## Background
The NeRF Volumetric Rendering method was introduced in the original NeRF paper by Mildenhall et al. in 2020. It is based on the classical volume rendering equation from computer graphics. The method has become the standard for rendering neural radiance fields. It is widely used in 3D reconstruction, novel view synthesis, and 3D object detection tasks that rely on volumetric representations [S1].

## Sources
[S1] NeRF-Det: Learning Geometry-Aware Volumetric Representation for Multi-View 3D Object Detection, https://doi.org/10.1109/iccv51070.2023.02131, fetched 2026-10-30
