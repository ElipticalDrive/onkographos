# CUDA Ray Marching (IDDR-NGP Backend)
## Overview
A GPU-accelerated ray marching kernel that samples points along rays using a hashed density grid, supporting adaptive step sizes and perturbation [S1, unofficial]. It builds on [Sparse Ray Marching (Training)](3890-sparse-ray-marching-training.md) [G].
## How it works
**Inputs and Outputs:**
The kernel takes ray origins $\mathbf{rays\_o} \in \mathbb{R}^{N \times 3}$, ray directions $\mathbf{rays\_d} \in \mathbb{R}^{N \times 3}$, near planes $\mathbf{near} \in \mathbb{R}^{N}$, and far planes $\mathbf{far} \in \mathbb{R}^{N}$ [S1, unofficial]. It outputs sampled positions $\mathbf{xyzs} \in \mathbb{R}^{N \times n\_step \times 3}$, directions $\mathbf{dirs} \in \mathbb{R}^{N \times n\_step \times 3}$, and deltas $\mathbf{deltas} \in \mathbb{R}^{N \times n\_step \times 2}$, where the first delta channel stores RGB contribution weights and the second stores depth contributions [S1, unofficial].

**Data Structures:**
The hashed density grid is stored in a `density_bitfield` tensor of type `uint8` [S1, unofficial]. The grid resolution and spatial extent are defined by the `resolution` parameter and a bounding box defined by `bound` [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:** For each ray $i \in [0, N)$, initialize current position $\mathbf{p} = \mathbf{rays\_o}[i]$, direction $\mathbf{d} = \mathbf{rays\_d}[i]$, near distance $t_{near} = \mathbf{near}[i]$, far distance $t_{far} = \mathbf{far}[i]$, accumulated transmittance $T = 1.0$, and step counter $k = 0$ [G].
2. **Ray Bounds:** Clip $t_{near}$ and $t_{far}$ to the scene bounding box defined by `bound` [G].
3. **Marching Loop:** While $k < n\_step$ and $T > \epsilon$ (where $\epsilon$ is a small threshold, e.g., $10^{-4}$) [G]:
   a. **Sampling:** Compute sample position $\mathbf{x} = \mathbf{p} + t \cdot \mathbf{d}$, where $t$ is the accumulated distance along the ray [G]. Apply perturbation to $\mathbf{x}$ if enabled [S1, unofficial].
   b. **Density Lookup:** Query the hashed density grid at $\mathbf{x}$ using `density_bitfield` to obtain density $\sigma$ [S1, unofficial].
   c. **Adaptive Step:** Calculate step size $\Delta t$ based on `density_thresh` and grid resolution [S1, unofficial]. [G: Standard adaptive step sizing].
   d. **Accumulation:** Compute opacity $\alpha = 1 - \exp(-\sigma \cdot \Delta t)$ [G]. Update transmittance $T \leftarrow T \cdot (1 - \alpha)$ [G]. Store $\mathbf{x}$ in $\mathbf{xyzs}[i, k]$ and $\mathbf{d}$ in $\mathbf{dirs}[i, k]$ [G]. Store $\Delta t$ in $\mathbf{deltas}[i, k, 0]$ and $\Delta t$ in $\mathbf{deltas}[i, k, 1]$ [S1, unofficial].
   e. **Advance:** $t \leftarrow t + \Delta t$, $k \leftarrow k + 1$ [G].
4. **Termination:** Stop when $k \ge n\_step$ or $T \le \epsilon$ [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def march_rays(rays_o, rays_d, near, far, density_bitfield, bound, density_thresh, n_step, epsilon=1e-4):
    N = rays_o.shape[0]
    xyzs = torch.zeros(N, n_step, 3)
    dirs = torch.zeros(N, n_step, 3)
    deltas = torch.zeros(N, n_step, 2)
    
    for i in range(N):
        p = rays_o[i].clone()
        d = rays_d[i] / torch.norm(rays_d[i])
        t_near = near[i]
        t_far = far[i]
        # Clip to bounding box [G]
        t_near = max(t_near, -bound)
        t_far = min(t_far, bound)
        
        T = 1.0
        t = 0.0
        k = 0
        
        while k < n_step and T > epsilon:
            # Sampling with optional perturbation [S1, unofficial]
            x = p + t * d
            # Perturbation logic omitted: not specified in sources [G]
            
            # Density lookup [S1, unofficial]
            sigma = query_hashed_grid(density_bitfield, x, bound)
            
            # Adaptive step size [S1, unofficial]
            delta_t = compute_adaptive_step(sigma, density_thresh)
            
            # Accumulation [G]
            alpha = 1.0 - torch.exp(-sigma * delta_t)
            T *= (1.0 - alpha)
            
            xyzs[i, k] = x
            dirs[i, k] = d
            deltas[i, k, 0] = delta_t
            deltas[i, k, 1] = delta_t
            
            t += delta_t
            k += 1
            
    return xyzs, dirs, deltas
```

**Parameters:**
- `density_bitfield`: `uint8` tensor for hashed density [S1, unofficial].
- `bound`: Bounding box size [S1, unofficial].
- `density_thresh`: Density threshold for step sizing [S1, unofficial].
- `min_near`: Minimum near plane distance [S1, unofficial].
- `bg_radius`: Background radius [S1, unofficial].
- Typical values: Not given in the sources.

**Edge Cases:**
- **Precision:** Floating-point precision in density lookup and transmittance accumulation can cause artifacts [G].
- **Boundaries:** Rays outside the `bound` should be clipped or discarded [G].
- **Degenerate Input:** Zero-length rays or invalid directions should be skipped or handled with a default value [G].
## Subtopics

### [Ray Tracing Kernel](5968-ray-tracing-kernel.md)
The Ray Tracing Kernel is a CUDA kernel implementation for forward and backward ray tracing through a sparse grid volume, handling spherical harmonic (SH) evaluation and ray bounds. It distinguishes itself by organizing computation per-warp for SH evaluation and using shared memory for ray specification caching to minimize global memory traffic. Choose this variant when differentiable gradient computation and warp-level SH evaluation are required.

## Operating characteristics
**Time and Memory Complexity:**
Not given in the sources.
**Measured Performance:**
Not given in the sources.
**Scaling:**
The method is designed for GPU acceleration, leveraging CUDA kernels for parallel processing [S1, unofficial].
**Strengths and Limitations:**
Strengths include adaptive step sizes and perturbation support, which can improve rendering quality and training stability [S1, unofficial]. Limitations include the lack of detailed specification for the hashing function and adaptive step size formula, which may hinder implementation [S1, unofficial].
**When to Use:**
Use this method for differentiable volume rendering with sparse hashed grids on CUDA-compatible hardware [S1, unofficial].
## Background
Origin: The method originates from the NeRF-Factory repository [S1, unofficial]. It builds on [Sparse Ray Marching (Training)](3890-sparse-ray-marching-training.md) [G].
## Sources
[S1] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
