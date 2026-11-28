# Expected Term Rendering

Also called: depth rendering via expected termination

## Overview
Expected Term Rendering computes the expected termination distance (depth) for each ray traversing a volume, providing a differentiable depth map output [S1, S2, unofficial]. It differs from standard depth rendering by calculating the weighted mean of sample depths along the ray, where weights are derived from the volume's transmittance and absorption properties, rather than simply picking the first opaque surface or the mean of all samples [S2, unofficial]. This method builds on [Depth from Ray Distance](5665-depth-from-ray-distance.md) by specifically implementing the "expected" strategy for depth estimation.

## How it works
**Inputs:**
- `rays`: A tensor of ray origins and directions, typically with shape `(N, 3)` for origins and `(N, 3)` for directions, where $N$ is the number of rays.
- `grid`: The volume representation (e.g., voxel grid or neural field parameters) containing density ($\sigma$) and other attributes.
- `opt`: Optimization or rendering configuration options, including sampling bounds and step counts.

**Outputs:**
- `results`: A tensor of dtype `float32` and device matching `rays.origins`, with size `(N,)`, containing the expected depth for each ray. Gradients are disabled during the forward pass (`requires_grad=false`) [S1, unofficial].

**Data Structures:**
- `SingleRaySpec`: A structure holding the origin, direction, and bounds for a single ray.
- `CUDA_N_BLOCKS_NEEDED(Q, TRACE_RAY_CUDA_THREADS)`: A macro or function calculating the number of CUDA blocks required for $Q$ rays, given a thread block size.

**Steps:**
1.  **Initialization**: Create a result tensor `results` with shape `(rays.origins.size(0),)` and dtype `float32` on the same device as `rays.origins`. Set `requires_grad` to `False` [S1, unofficial].
2.  **Ray Specification**: For each ray $i$, construct a `SingleRaySpec` object. This involves determining the near and far bounds of the volume intersection for the ray [S1, unofficial].
3.  **Kernel Launch**: Calculate the number of blocks $B$ using `CUDA_N_BLOCKS_NEEDED(Q, TRACE_RAY_CUDA_THREADS)`, where $Q$ is the total number of rays. Launch the CUDA kernel `device::render_ray_expected_term_kernel` with $B$ blocks. Pass the grid, rays, options, and the data pointer of `results` as arguments [S1, unofficial].
4.  **Kernel Execution (Per Ray)**: Inside the kernel, for each ray:
    a.  Trace the ray through the volume, sampling density $\sigma$ at discrete points along the ray segment defined by the bounds.
    b.  Compute the transmittance $T(t)$ at each sample point $t$:
        $$ T(t) = \exp\left(-\int_{t_{near}}^{t} \sigma(s) ds\right) $$
        In discrete form, with sample distances $\Delta t_j$:
        $$ T_j = \exp\left(-\sum_{k=0}^{j-1} \sigma_k \Delta t_k\right) $$
    c.  Compute the weight $w_j$ for each sample $j$ as the probability of termination at that sample:
        $$ w_j = T_j \cdot (1 - \exp(-\sigma_j \Delta t_j)) $$
    d.  Compute the expected depth $D$ as the weighted mean of the sample depths $t_j$:
        $$ D = \sum_{j} w_j \cdot t_j $$
    e.  Store $D$ in the corresponding position in `results` [S2, unofficial].

**Parameters:**
- `method`: The strategy for computing depth. Set to `'expected'` to use the weighted mean depth calculation [S2, unofficial].
- `TRACE_RAY_CUDA_THREADS`: The number of threads per CUDA block. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision**: The exponential decay in transmittance can lead to underflow for large depths or high densities. Use log-space computations if necessary [G].
- **Boundaries**: Incorrect near/far bounds can lead to sampling outside the volume or missing relevant density. Ensure bounds are tightly fitted to the volume geometry [G].
- **Degenerate Input**: Rays with zero length or invalid directions should be handled or filtered out before kernel launch [G].

## Operating characteristics
- **Time Complexity**: $O(N \cdot S)$, where $N$ is the number of rays and $S$ is the number of samples per ray.
- **Memory Complexity**: $O(N)$ for the output tensor, plus memory for the volume representation.
- **Performance**: Performance depends heavily on the volume density and the efficiency of the CUDA kernel implementation. Parallelization across rays is straightforward [S1, unofficial].
- **Strengths**: Provides a smooth, differentiable depth estimate that accounts for the entire volume structure, not just the first surface. Useful for training neural radiance fields where depth supervision is needed [S2, unofficial].
- **Limitations**: Computationally more expensive than simple depth picking. Sensitive to noise in density estimates, which can affect the weighted mean [G].
- **When to use**: When a differentiable depth map is required for training or optimization, such as in neural volume rendering [S1, S2, unofficial].
- **When not to use**: When real-time performance is critical and a simple depth approximation suffices.

## Background
Expected Term Rendering is implemented in the context of neural volume rendering frameworks, specifically noted in the `svox2` and `ganerf` repositories [S1, S2, unofficial]. It is used to generate depth maps that can be used as supervision signals for training neural radiance fields, leveraging the differentiability of the volume rendering integral [S1, S2, unofficial]. The method is part of the broader family of differentiable volume rendering techniques.

## Sources
[S1] svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
[S2] ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
