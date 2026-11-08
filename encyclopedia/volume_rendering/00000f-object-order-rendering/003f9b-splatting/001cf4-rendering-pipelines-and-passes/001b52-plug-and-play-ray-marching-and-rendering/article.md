# Plug-and-Play Ray Marching and Rendering

Also called: Differentiable Volumetric Rendering Pipeline, nerfacc pipeline

## Overview
Plug-and-Play Ray Marching and Rendering is a modular pipeline interface that decouples the ray marching and volumetric accumulation logic from the specific radiance field representation, allowing efficient differentiable rendering to be integrated into any user-defined neural radiance field [S1, unofficial]. Unlike specialized, monolithic implementations such as Instant-NGP which rely on custom CUDA kernels, this method provides a Python API built on PyTorch, prioritizing ease of extension for dynamic and unbounded scenes over raw kernel-level optimization [S1, unofficial]. It solves the problem of boilerplate rendering code by standardizing the input/output contract for density and color queries, enabling acceleration of existing NeRF methods with minimal code changes [S1, unofficial].

## How it works
The method operates by defining two callback functions that interface with a user-defined radiance field, followed by calls to a core ray marching engine and a differentiable renderer.

**Inputs and Outputs**
*   **Inputs:** Ray origins $\mathbf{o}$ (Tensor, shape $N_{rays} \times 3$), ray directions $\mathbf{d}$ (Tensor, shape $N_{rays} \times 3$), and a radiance field object capable of querying density and RGB at 3D positions.
*   **Outputs:** Final image color (Tensor, shape $N_{rays} \times 3$), opacity (Tensor, shape $N_{rays} \times 1$), and depth (Tensor, shape $N_{rays} \times 1$).

**Data Structures**
The pipeline uses PyTorch Tensors for all data exchange. The ray marching step returns:
1.  `packed_info`: A packed integer tensor containing metadata about valid samples (e.g., ray indices and sample indices) to optimize memory access during rendering.
2.  `t_starts`: Tensor of start distances for each valid sample interval.
3.  `t_ends`: Tensor of end distances for each valid sample interval.

**Parameters**
*   `near_plane`: Starting distance for ray marching. Typical value: 0.2 [S1, unofficial].
*   `far_plane`: Ending distance for ray marching. Typical value: 1.0 [S1, unofficial].
*   `early_stop_eps`: Threshold for early stopping based on transmittance. Typical value: $1 \times 10^{-4}$ [S1, unofficial].
*   `alpha_thre`: Threshold for skipping samples with low opacity. Typical value: $1 \times 10^{-2}$ [S1, unofficial].

**Formulas**
The volumetric rendering equation accumulates color contributions:
$$ C(\mathbf{r}) = \sum_{i=1}^N T_i \alpha_i \mathbf{c}_i $$
where:
*   $\mathbf{c}_i$ is the RGB color at sample $i$.
*   $\alpha_i$ is the opacity of sample $i$, defined as $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$.
*   $\sigma_i$ is the volume density at sample $i$.
*   $\delta_i$ is the distance between sample $i$ and $i-1$ (i.e., $t_{ends,i} - t_{starts,i}$).
*   $T_i$ is the accumulated transmittance up to sample $i$, defined as $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ [S1, unofficial].

**Steps**
1.  **Define Density Query Function (`sigma_fn`):** Create a function that accepts `t_starts`, `t_ends`, and `ray_indices`. Compute the 3D positions of the sample midpoints using $\mathbf{p} = \mathbf{o}_{ray\_idx} + \mathbf{d}_{ray\_idx} \cdot \frac{t_{starts} + t_{ends}}{2}$. Query the radiance field for density $\sigma$ at these positions and return the tensor of shape $(N_{samples}, 1)$ [S1, unofficial].
2.  **Define Color and Density Query Function (`rgb_sigma_fn`):** Create a function with the same inputs. Compute positions identically to Step 1. Query the radiance field for both RGB color $\mathbf{c}$ and density $\sigma$. Return tensors of shape $(N_{samples}, 3)$ and $(N_{samples}, 1)$ respectively [S1, unofficial].
3.  **Execute Ray Marching:** Call the ray marching engine with ray origins, directions, `sigma_fn`, and plane parameters. The engine iterates along rays, querying `sigma_fn` to determine where to place samples. It prunes empty or occluded regions by skipping samples where transmittance is below `early_stop_eps` or opacity is below `alpha_thre`. It returns `packed_info`, `t_starts`, and `t_ends` for the valid samples [S1, unofficial].
4.  **Render:** Call the differentiable rendering function with `rgb_sigma_fn`, `packed_info`, `t_starts`, and `t_ends`. The renderer uses the packed info to efficiently gather colors and densities, computes $\alpha_i$ and $T_i$ using the formulas above, and accumulates the final color, opacity, and depth per ray [S1, unofficial].
5.  **Optimize:** Compute loss (e.g., MSE) between rendered color and ground truth, and backpropagate through the rendering and ray marching steps to update the radiance field parameters [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
from torch import Tensor
import nerfacc

# Assume radiance_field is a user-defined PyTorch module
# Assume rays_o (N_rays, 3) and rays_d (N_rays, 3) are provided

def sigma_fn(t_starts: Tensor, t_ends: Tensor, ray_indices: Tensor) -> Tensor:
    """Query density values from a user-defined radiance field."""
    # Gather ray origins and directions for the specific samples
    t_origins = rays_o[ray_indices]  # (n_samples, 3)
    t_dirs = rays_d[ray_indices]     # (n_samples, 3)
    
    # Compute sample positions at the midpoint of the interval
    positions = t_origins + t_dirs * (t_starts + t_ends) / 2.0
    
    # Query density from the radiance field
    sigmas = radiance_field.query_density(positions)
    return sigmas  # (n_samples, 1)

def rgb_sigma_fn(t_starts: Tensor, t_ends: Tensor, ray_indices: Tensor):
    """Query RGB and density values from a user-defined radiance field."""
    t_origins = rays_o[ray_indices]  # (n_samples, 3)
    t_dirs = rays_d[ray_indices]     # (n_samples, 3)
    
    # Compute sample positions
    positions = t_origins + t_dirs * (t_starts + t_ends) / 2.0
    
    # Query RGB and density from the radiance field
    # Note: Implementation depends on specific radiance field API
    rgbs, sigmas = radiance_field(positions, condition=t_dirs)
    return rgbs, sigmas  # (n_samples, 3), (n_samples, 1)

# 1. Efficient Raymarching: Skip empty and occluded space
packed_info, t_starts, t_ends = nerfacc.ray_marching(
    rays_o, 
    rays_d, 
    sigma_fn=sigma_fn, 
    near_plane=0.2, 
    far_plane=1.0, 
    early_stop_eps=1e-4, 
    alpha_thre=1e-2
)

# 2. Differentiable Volumetric Rendering
color, opacity, depth = nerfacc.rendering(
    rgb_sigma_fn, 
    packed_info, 
    t_starts, 
    t_ends
)

# 3. Optimization step (example)
# loss = F.mse_loss(color, color_gt)
# loss.backward()
# optimizer.step()
```

**Edge Cases and Pitfalls**
*   **Precision:** The transmittance calculation involves exponentials of negative sums. If densities are very high or step sizes large, underflow to zero can occur, correctly terminating the ray but potentially causing gradient issues if not handled carefully by the backend [S1, unofficial].
*   **Boundary Conditions:** The `near_plane` and `far_plane` parameters define the marching bounds. Samples outside these bounds are ignored. If the scene extends beyond `far_plane`, parts of the scene will be clipped [S1, unofficial].
*   **Empty Regions:** The pruning logic relies on `sigma_fn` returning accurate densities. If the radiance field returns non-zero density in empty space (e.g., due to noise or improper training), the ray marching will waste samples on occluded regions, reducing performance [S1, unofficial].

## Subtopics

### Dynamic Scene Rendering
Extends the base pipeline to handle time-varying radiance fields by including time as an input to the radiance field query functions. [S1, unofficial]

### Unbounded Scene Handling
Adapts the ray marching bounds and density queries to handle scenes without explicit bounding boxes, often using coordinate transformations or specific density biases. [S1, unofficial]

## Operating characteristics
*   **Time Complexity:** Depends on the radiance field query cost and the number of valid samples after pruning. The ray marching step is $O(N_{samples})$ where $N_{samples}$ is the number of samples retained after pruning empty/occluded regions. The rendering step is also $O(N_{samples})$ [S1, unofficial].
*   **Memory Complexity:** Proportional to the number of valid samples stored in `t_starts`, `t_ends`, and `packed_info`. The `packed_info` structure is optimized to minimize memory overhead compared to dense arrays [S1, unofficial].
*   **Measured Performance:** On a single NVIDIA TITAN RTX GPU, training a vanilla 8-layer MLP NeRF took 1 hour with this pipeline, compared to 1-2 days in the original paper. Instant-NGP training took 4.5 minutes in Python, comparable to its official CUDA implementation [S1, unofficial].
*   **Strengths:** High ease of use due to Python/PyTorch API; flexible integration with various radiance field architectures; automatic pruning of empty space improves efficiency in sparse scenes [S1, unofficial].
*   **Limitations:** Python overhead may make it slower than highly optimized C++/CUDA implementations for extremely large-scale or real-time applications where every millisecond counts; relies on PyTorch's autograd which may have higher memory overhead than custom CUDA kernels [S1, unofficial].
*   **When to use:** Ideal for research, prototyping, and training dynamic or unbounded NeRFs where flexibility and ease of implementation are prioritized over maximum raw inference speed.
*   **When not to use:** When deploying to hardware with strict latency constraints requiring custom CUDA kernels, or when the radiance field is static and highly optimized for a specific architecture [S1, unofficial].

## Background
The method was introduced as part of the NerfAcc toolbox, a general NeRF acceleration library. It builds on techniques from Instant-NGP but extends them to support dynamic and unbounded scenes. The library was released as a preprint in 2022, aiming to provide a user-friendly Python API that accelerates most NeRF methods without requiring low-level CUDA programming [S1, unofficial].

## Sources
[S1] NerfAcc: A General NeRF Acceleration Toolbox, https://doi.org/10.48550/arxiv.2210.04847, fetched 2026-10-26 (unofficial)
