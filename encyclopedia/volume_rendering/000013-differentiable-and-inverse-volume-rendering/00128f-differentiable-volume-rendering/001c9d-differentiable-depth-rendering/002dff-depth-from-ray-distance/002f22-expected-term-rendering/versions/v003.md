# Expected Term Rendering
Also called: depth rendering via expected termination

## Overview
Expected Term Rendering computes the expected termination distance (depth) for each ray traversing a volume, returning a tensor of size $Q$ (number of rays) [S1, unofficial][S2, unofficial]. It differs from standard depth rendering by calculating the weighted mean of sample depths along the ray, where weights are derived from the volume's transmittance and absorption properties [G]. This method builds on [Depth from Ray Distance](5665-depth-from-ray-distance.md) by specifically implementing the expected value strategy for depth estimation [S2, unofficial].

## How it works
**Inputs:**
- `rays`: A tensor containing ray origins and directions, with shape `(Q, 3)` for origins and `(Q, 3)` for directions, where $Q$ is the number of rays [S1, unofficial][S2, unofficial].
- `grid`: The volume representation containing density ($\sigma$) and other attributes [S1, unofficial][S2, unofficial].
- `opt`: Rendering configuration options, including sampling bounds and step counts [S1, unofficial][S2, unofficial].

**Outputs:**
- `results`: A tensor of dtype and device matching `rays.origins`, with size `{Q}` (or `rays.origins.size(0)`), containing the expected depth for each ray [S1, unofficial][S2, unofficial].

**Data Structures:**
- `SingleRaySpec`: A structure holding the origin, direction, and bounds for a single ray [S2, unofficial].
- `CUDA_N_BLOCKS_NEEDED(Q, TRACE_RAY_CUDA_THREADS)`: A macro or function calculating the number of CUDA blocks required for $Q$ rays, given a thread block size [S1, unofficial][S2, unofficial].

**Steps:**
1. **Initialization**: Create a result tensor `results` with shape `(rays.origins.size(0),)` and dtype `float32` on the same device as `rays.origins`. Set `requires_grad` to `False` [S2, unofficial].
2. **Ray Specification**: For each ray $i$, construct a `SingleRaySpec` object. This involves determining the near and far bounds of the volume intersection for the ray [S2, unofficial].
3. **Kernel Launch**: Calculate the number of blocks $B$ using `CUDA_N_BLOCKS_NEEDED(Q, TRACE_RAY_CUDA_THREADS)`, where $Q$ is the total number of rays. Launch the CUDA kernel `device::render_ray_expected_term_kernel` with $B$ blocks. Pass the grid, rays, options, and the data pointer of `results` as arguments [S1, unofficial][S2, unofficial].
4. **Kernel Execution (Per Ray)**: Inside the kernel, for each ray:
    a. Trace the ray through the volume, sampling density $\sigma$ at discrete points along the ray segment defined by the bounds [G].
    b. Compute the transmittance $T(t)$ at each sample point $t$:
       $$ T(t) = \exp\left(-\int_{t_{near}}^{t} \sigma(s) ds\right) $$
       In discrete form, with sample distances $\Delta t_j$:
       $$ T_j = \exp\left(-\sum_{k=0}^{j-1} \sigma_k \Delta t_k\right) $$
    c. Compute the weight $w_j$ for each sample $j$ as the probability of termination at that sample:
       $$ w_j = T_j \cdot (1 - \exp(-\sigma_j \Delta t_j)) $$
    d. Compute the expected depth $D$ as the weighted mean of the sample depths $t_j$:
       $$ D = \sum_{j} w_j \cdot t_j $$
    e. Store $D$ in the corresponding position in `results` [G].

**Parameters:**
- `TRACE_RAY_CUDA_THREADS`: The number of threads per CUDA block for the expected term rendering kernel [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision**: The exponential decay in transmittance can lead to underflow for large depths or high densities. Use log-space computations if necessary [G].
- **Boundaries**: Incorrect near/far bounds can lead to sampling outside the volume or missing relevant density. Ensure bounds are tightly fitted to the volume geometry [G].
- **Degenerate Input**: Rays with zero length or invalid directions should be handled or filtered out before kernel launch [G].

## Subtopics

### [Expected Term Ray Tracing](14910-expected-term-ray-tracing.md)
Computes the expected distance to the first opaque surface (or significant density threshold) along a ray passing through a sparse voxel grid. It differs from standard volume rendering by outputting a scalar expected depth value rather than a color or opacity-weighted sum, making it suitable for uncertainty estimation or specific depth-based loss functions. It builds on [Expected Term Rendering](5952-expected-term-rendering.md) by providing the specific CUDA kernel implementation for this metric.

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: Not given in the sources.
- **Strengths**: Not given in the sources.
- **Limitations**: Not given in the sources.
- **When to use**: Not given in the sources.
- **When not to use**: Not given in the sources.

## Background
Expected Term Rendering is implemented in the context of neural volume rendering frameworks, specifically noted in the `nerf-factory` and `svox2` repositories [S1, unofficial][S2, unofficial]. It computes the expected termination distance (depth) along each ray, typically used for depth map rendering [S2, unofficial]. The method is part of the broader family of differentiable volume rendering techniques [G].

## Sources
[S1] kakaobrain/nerf-factory, https://github.com/kakaobrain/nerf-factory, fetched 2026-10-26 (unofficial)
[S2] sxyu/svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
