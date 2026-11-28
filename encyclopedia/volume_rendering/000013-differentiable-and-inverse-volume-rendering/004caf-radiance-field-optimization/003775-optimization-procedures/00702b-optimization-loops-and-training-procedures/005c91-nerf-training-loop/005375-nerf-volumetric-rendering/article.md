# NeRF Volumetric Rendering
Also called: NeRF Alpha Compositing, NeRF Rendering Equation

## Overview
NeRF Volumetric Rendering synthesizes final pixel color and expected depth from sampled radiance field properties along camera rays [S1]. It models scene geometry using an opacity field that reflects the probability of an object’s presence, accumulating color and transmittance via alpha compositing [S1]. It differs from sibling methods by explicitly computing the volume rendering equation to produce differentiable color and depth outputs for training supervision [S1].

## How it works
**Inputs and Outputs**
- **Input**: A set of $N_p$ samples along a ray $r$. For each sample $i$, the input includes the 3D position $\mathbf{x}_i$, the density $\sigma_i$, the color $\mathbf{c}_i$ (typically RGB), and the distance from the camera $t_i$ [S1].
- **Output**: The final pixel color $\hat{\mathbf{C}}(r)$ and the expected depth $D(r)$ [S1].

**Data Structures**
The method operates on arrays of size $N_p$:
- $\sigma$: Array of densities, shape $(N_p, 1)$ or scalar per sample [S1].
- $\mathbf{c}$: Array of colors, shape $(N_p, 3)$ [S1].
- $t$: Array of depths/distances, shape $(N_p, 1)$ [S1].
- $\delta t$: Array of distances between consecutive samples, shape $(N_p, 1)$ [S1].

**Numbered Steps**
1. Compute opacity weights $\alpha_i = 1 - \exp(-\sigma_i \delta t_i)$ for each sample [S1].
2. Compute accumulated transmittance $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta t_j)$ starting with $T_1 = 1$ [S1].
3. Accumulate final color $\hat{\mathbf{C}}(r) = \sum_{i=1}^{N_p} T_i \alpha_i \mathbf{c}_i$ and depth $D(r) = \sum_{i=1}^{N_p} T_i \alpha_i t_i$ [S1].

**Formulas**
The rendering equation computes the accumulated transmittance $T_i$ and the alpha weight $\alpha_i$ for each sample:
$$ \alpha_i = 1 - \exp(-\sigma_i \delta t_i) $$
$$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta t_j\right) $$
The final color and depth are computed as weighted sums:
$$ \hat{\mathbf{C}}(r) = \sum_{i=1}^{N_p} T_i \alpha_i \mathbf{c}_i $$
$$ D(r) = \sum_{i=1}^{N_p} T_i \alpha_i t_i $$
Where:
- $T_i$ is the accumulated transmittance at sample $i$ [S1].
- $\alpha_i$ is the opacity weight at sample $i$ [S1].
- $t_i$ is the distance of the $i$-th sample from the camera [S1].
- $\delta t$ is the distance between sampled points [S1].

**Parameters**
- $N_p$: Number of samples per ray. Not given in the sources.

**Edge Cases and Pitfalls**
Not given in the sources.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def nerf_volumetric_rendering(sigma, colors, depths, delta_t):
    """
    Render a ray using NeRF volumetric rendering.
    Maps 0-based Python indices to 1-based formula indices (i_formula = i_python + 1).
    """
    N_p = len(sigma)
    
    # Step 1: Compute alpha_i = 1 - exp(-sigma_i * delta_t_i)
    alpha = [1.0 - math.exp(-sigma[i] * delta_t[i]) for i in range(N_p)]
    
    # Step 2: Compute T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_t_j)
    # T_1 (i=0 in Python) = 1.0
    T = [0.0] * N_p
    T[0] = 1.0
    cum_sum = 0.0
    for i in range(1, N_p):
        cum_sum += sigma[i-1] * delta_t[i-1]
        T[i] = math.exp(-cum_sum)
        
    # Step 3: Accumulate final color and depth using weights w_i = T_i * alpha_i
    color = [0.0, 0.0, 0.0]
    depth = 0.0
    for i in range(N_p):
        w = T[i] * alpha[i]
        for c in range(3):
            color[c] += w * colors[i][c]
        depth += w * depths[i]
        
    return color, depth
```

## Subtopics

### [Lightplane Renderer](16859-lightplane-renderer.md)
A memory-efficient differentiable component for neural 3D fields that reduces memory usage by four orders of magnitude compared to standard autograd renderers. It solves the bottleneck of storing intermediate values for backpropagation by fusing operations over casted rays instead of 3D points. It sets itself apart from siblings by updating rendered pixels and transmittance on-the-fly rather than storing intermediate tensors. Choose it when memory constraints limit standard autograd rendering.

### [PL-NeRF Volumetric Rendering](23171-pl-nerf-volumetric-rendering.md)
PL-NeRF is a NeRF implementation that replaces the constant opacity approximation used in vanilla NeRF with piecewise linear opacity interpolation and precise importance sampling. It solves the quadrature instability inherent in the constant approximation, allowing for more accurate integration of arbitrarily small opacity values. It sets itself apart from siblings like [Lightplane Renderer](16859-lightplane-renderer.md) by focusing on the analytical integration of the volume rendering equation within the standard MLP-based radiance field framework rather than using a differentiable ray-marching or plane-based rasterization approach. Choose it when quadrature instability from constant opacity approximations degrades training accuracy.

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] NeRF-Det: Learning Geometry-Aware Volumetric Representation for Multi-View 3D Object Detection, https://doi.org/10.1109/iccv51070.2023.02131, fetched 2026-10-30
