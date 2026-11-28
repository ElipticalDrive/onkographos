# NV Rendering Formula (Trilerp)
Also called: Trilerp, Volume Ray Tracing with Trilinear Interpolation
## Overview
The NV rendering formula is a direct volume ray casting method that resamples volume data at evenly spaced locations along rays and composites the results using trilinear interpolation [S1]. It is implemented in modern frameworks (e.g., svox2) as `volume_render_nvol` and its variants, providing forward, backward, and fused execution modes [S2, unofficial].
## How it works
### Inputs and Outputs
Based on the function signatures in [S2, unofficial], the forward pass takes `SparseGridSpec` (volume grid metadata), `RaysSpec` (ray origins and directions), and `RenderOptions` (step size, sample count), and returns a rendered `Tensor`. The backward and fused variants additionally accept upstream gradient tensors and scalar parameters, returning `GridOutputGrads` [S2, unofficial]. Exact internal layouts are implementation-specific to the sparse storage format [S2, unofficial].
### Data Structures
*   **SparseGridSpec**: Metadata for the sparse voxel grid. Exact internal layout is implementation-specific to the sparse storage format (e.g., hash table or octree), but logically represents a 3D field $V(x,y,z)$ [S2, unofficial].
*   **RaysSpec**: Contains arrays of ray origins $\mathbf{o}_i$ and direction vectors $\mathbf{d}_i$ for each pixel $i$ [S2, unofficial].
*   **RenderOptions**: Includes step size $\Delta s$, number of samples $N$, and clipping distances $t_{min}, t_{max}$ [S2, unofficial].
### Algorithm Steps
1.  **Ray Marching (Forward):** For each ray $i$, generate sample positions $\mathbf{p}_k = \mathbf{o}_i + t_k \mathbf{d}_i$ for $k = 0 \dots N-1$, where $t_k$ are evenly spaced depths [S1].
2.  **Trilinear Sampling:** For each sample position $\mathbf{p}_k$, perform trilinear interpolation on the volume grid $V$ to obtain color $C_k$ and opacity $\alpha_k$ [S1].
3.  **Compositing:** Composite the samples using standard front-to-back volume integration:
    $$C_{out} = \sum_{k=0}^{N-1} C_k \alpha_k \prod_{j=0}^{k-1} (1 - \alpha_j)$$
    $$\alpha_{out} = 1 - \prod_{k=0}^{N-1} (1 - \alpha_k)$$
4.  **Backward Pass:** Given the output gradient $\frac{\partial L}{\partial C_{out}}$ and $\frac{\partial L}{\partial \alpha_{out}}$, compute gradients for each sample $k$ using the chain rule through the compositing equation, propagate gradients through the trilinear interpolation kernel to compute gradients with respect to the voxel values, and accumulate gradients into `GridOutputGrads` [S2, unofficial].
5.  **Fused Pass:** Combines forward and backward operations in a single kernel launch to reduce memory bandwidth overhead, typically used during training [S2, unofficial].
### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_render_nvol_forward(grid, rays, options):
    """Forward pass: Renders image from sparse grid using trilinear interpolation."""
    image = {}
    for ray in rays:
        origin, direction = ray.origin, ray.direction
        depths = [options.t_min + k * options.step for k in range(options.num_samples)]
        acc_color = [0.0, 0.0, 0.0]
        transmittance = 1.0
        samples = []
        for t in depths:
            pos = origin + t * direction
            color, alpha = trilinear_interpolate(grid, pos)
            weight = alpha * transmittance
            acc_color = [acc_color[i] + color[i] * weight for i in range(3)]
            transmittance *= (1.0 - alpha)
            samples.append((pos, color, alpha, weight, transmittance))
            if transmittance < 1e-4:
                break
        image[ray.pixel_index] = acc_color
    return image

def volume_render_nvol_backward(grid, rays, options, grad_output, grid_grads):
    """Backward pass: Computes gradients w.r.t. grid values."""
    for ray in rays:
        origin, direction = ray.origin, ray.direction
        depths = [options.t_min + k * options.step for k in range(options.num_samples)]
        samples = []
        transmittance = 1.0
        for t in depths:
            pos = origin + t * direction
            color, alpha = trilinear_interpolate(grid, pos)
            weight = alpha * transmittance
            transmittance *= (1.0 - alpha)
            samples.append((pos, color, alpha, weight, transmittance))
            if transmittance < 1e-4:
                break
        grad_c = grad_output[ray.pixel_index]
        grad_T = [0.0, 0.0, 0.0]
        for k in range(len(samples) - 1, -1, -1):
            pos, c_k, a_k, w_k, t_k = samples[k]
            t_next = t_k * (1.0 - a_k)
            grad_c_k = [grad_c[i] * a_k * t_k for i in range(3)]
            grad_alpha_k = sum((c_k[i] * t_k - t_next) * grad_c[i] for i in range(3))
            accumulate_grid_gradient(grid_grads, pos, grad_c_k, grad_alpha_k)
            grad_T = [grad_c[i] * c_k[i] * a_k + grad_T[i] * (1.0 - a_k) for i in range(3)]
    return grid_grads

def trilinear_interpolate(grid, pos):
    """Standard trilinear interpolation returning color and opacity [G]."""
    x, y, z = pos
    ix, iy, iz = floor(x), floor(y), floor(z)
    fx, fy, fz = x - ix, y - iy, z - iz
    # Fetch 8 surrounding voxels (assumes grid supports direct indexing)
    v000 = grid[ix, iy, iz]; v100 = grid[ix+1, iy, iz]
    v010 = grid[ix, iy+1, iz]; v110 = grid[ix+1, iy+1, iz]
    v001 = grid[ix, iy, iz+1]; v101 = grid[ix+1, iy, iz+1]
    v011 = grid[ix, iy+1, iz+1]; v111 = grid[ix+1, iy+1, iz+1]
    # Interpolate along x
    c00 = v000 * (1-fx) + v100 * fx
    c01 = v010 * (1-fx) + v110 * fx
    c10 = v001 * (1-fx) + v101 * fx
    c11 = v011 * (1-fx) + v111 * fx
    # Interpolate along y
    c0 = c00 * (1-fy) + c01 * fy
    c1 = c10 * (1-fy) + c11 * fy
    # Interpolate along z
    color = c0 * (1-fz) + c1 * fz
    return color

def accumulate_grid_gradient(grid_grads, pos, grad_color, grad_alpha):
    """Accumulate interpolated gradients into sparse grid storage [G]."""
    x, y, z = pos
    ix, iy, iz = floor(x), floor(y), floor(z)
    fx, fy, fz = x - ix, y - iy, z - iz
    # Distribute gradients to 8 surrounding voxels based on trilinear weights
    weights = [(1-fx)*(1-fy)*(1-fz), fx*(1-fy)*(1-fz), (1-fx)*fy*(1-fz), fx*fy*(1-fz),
               (1-fx)*(1-fy)*fz, fx*(1-fy)*fz, (1-fx)*fy*fz, fx*fy*fz]
    for i in range(8):
        grid_grads[ix + (i&1), iy + ((i>>1)&1), iz + (i>>2)] += grad_color * weights[i] + grad_alpha * weights[i]
```
### Parameters
*   `num_samples`: Number of steps per ray. Typical values not given in the sources [S1][S2, unofficial].
*   `step`: Distance between samples. Typical values not given in the sources [S1][S2, unofficial].
*   `t_min`, `t_max`: Near and far clipping planes. Typical values not given in the sources [S2, unofficial].
### Edge Cases and Pitfalls
Not given in the sources.
## Subtopics

### Optimized Trilinear Reconstruction
Uses SIMD extensions (MMX, SSE, 3DNow!) to accelerate memory copying and interpolation [2437-optimized-trilinear-reconstruction.md]. Choose when hardware supports these extensions and performance is critical [2437-optimized-trilinear-reconstruction.md].

### Bilinear Interpolation
2D interpolation [1234-bilinear-interpolation.md]. Choose for texture maps or 2D slices, not 3D volumes [1234-bilinear-interpolation.md].

## Operating characteristics
*   **Time Complexity**: Not given in the sources [S1][S2, unofficial].
*   **Memory Complexity**: Not given in the sources [S1][S2, unofficial].
*   **Performance**: Measured performance metrics not given in the sources [S1][S2, unofficial].
*   **Strengths**: Trilinear interpolation is simple and differentiable [G].
*   **Limitations**: Trilinear interpolation causes blurring compared to higher-order schemes [G]. Sparse grid lookup overhead depends on the storage implementation [G].
*   **When to Use**: Not given in the sources [S1][S2, unofficial].
## Background
The base algorithm of ray casting with trilinear interpolation was described in early volume rendering literature by Nieh and Levoy [S1]. The specific "NV" naming and fused differentiable implementations appear in modern deep learning frameworks like svox2 [S2, unofficial].
## Sources
[S1] Volume rendering on scalable shared-memory MIMD architectures, https://doi.org/10.1145/147130.147141, fetched 2026-10-25
[S2] sxyu/svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
