# Inference Ray Marching and Integration

Also called: Inference Ray Marching, Ray Marching for Inference, CUDA Ray Marching (Inference), Sparse Ray Marching (Inference), Ray Marching (CUDA Kernel)

## Overview
Inference Ray Marching and Integration is a CUDA-accelerated pipeline that generates 3D sample points, view directions, and step sizes (deltas) along rays for real-time neural volume rendering inference [S1, S2, S4, unofficial]. It solves the problem of efficiently traversing a pre-computed density grid (or hashed bitfield) to identify occupied regions while skipping empty space, serving as the core sampling engine for inference-time rendering [S2, S3, unofficial]. It differs from training-time marchers by prioritizing throughput and memory efficiency over gradient stability, often utilizing fixed step caps and strict early-termination logic [S2, S3, unofficial]. It builds on [Acceleration](18-acceleration.md) principles by using occupancy grids and mean density for step size estimation and early termination [S2, unofficial].

## How it works
The method operates in two main phases: **Ray Marching** (sampling) and **Integration** (compositing). The marching phase is typically implemented as a CUDA kernel (`_backend.march_rays`) that outputs tensors of coordinates, directions, and deltas. The integration phase evaluates the neural network at these points and composites the results.

### Inputs and Outputs
**Inputs:**
*   `rays_o`: Float array `[N, 3]`, ray origins for all rays [S2, S4, unofficial].
*   `rays_d`: Float array `[N, 3]`, ray directions for all rays [S2, S4, unofficial].
*   `near`: Float array `[N]`, near plane distances [S2, S4, unofficial].
*   `far`: Float array `[N]`, far plane distances [S2, S4, unofficial].
*   `density_bitfield`: Uint8 array `[CHHH // 8]`, hashed grid for density acceleration (Instant-NGP style) [S1, S4, unofficial].
*   `density_grid`: Float array `[H, H, H]`, pre-computed 3D density grid (alternative to bitfield) [S2, unofficial].
*   `bound`: Float, spatial boundary limit for the scene/grid (e.g., 1.0 for box `[-bound, bound]^3`) [S1, S2, S4, unofficial].
*   `mean_density`: Float, average density value in the grid, used for step size estimation [S2, unofficial].
*   `dt_gamma`: Float, cone angle for exponential acceleration. If `> 0`, accelerates rendering but may reduce quality [S1, S4, unofficial]. Default `1/256` [S1, unofficial].
*   `perturb`: Bool/Int, enables jittering of sample positions for anti-aliasing. Int `> 0` acts as a random seed [S1, S2, S4, unofficial].
*   `max_steps`: Int, maximum number of sampled points along each ray. Affects minimum step size [S1, S4, unofficial]. Default `1024` [S4, unofficial].
*   `align`: Int, pad output size to be divisible by `align`. Set to `-1` to disable [S1, S2, unofficial].
*   `n_alive`: Int, number of active rays currently being processed [S1, S2, S4, unofficial].
*   `rays_alive`: Int array `[N]`, IDs of alive rays (only first `n_alive` are used) [S1, S2, S4, unofficial].
*   `rays_t`: Float array `[N]`, current traversal position (distance `t`) along each ray [S1, S2, S4, unofficial].

**Outputs:**
*   `xyzs`: Float array `[M, 3]`, 3D coordinates of sampled points [S2, S4, unofficial].
*   `dirs`: Float array `[M, 3]`, ray directions corresponding to each sampled point [S2, S4, unofficial].
*   `deltas`: Float array `[M, 2]`, two delta values per point: first for RGB contribution, second for depth contribution [S2, S4, unofficial].
*   `weights_sum`: Float array, accumulated weights (opacity contribution) [S2, unofficial].
*   `depth`: Float array, accumulated depth [S2, unofficial].
*   `image`: Float array, final RGB image [S2, unofficial].

### Data Structures
*   **Alive Ray Buffers:**
    *   `rays_alive`: `torch.zeros(2, n_alive, dtype=torch.int32)` [S2, unofficial].
    *   `rays_t`: `torch.zeros(2, n_alive, dtype=dtype)` [S2, unofficial].
*   **Output Tensors:**
    *   `xyzs`: Shape `[M, 3]`, initialized to zeros [S2, unofficial].
    *   `dirs`: Shape `[M, 3]`, initialized to zeros [S2, unofficial].
    *   `deltas`: Shape `[M, 2]`, initialized to zeros [S2, unofficial].
    *   `M`: Total output size, calculated as `n_alive * n_step`, padded to `align` if `align > 0` [S1, S2, unofficial].

### Algorithm Steps

1.  **Initialization:**
    *   Ensure `rays_o` and `rays_d` are on CUDA [S1, unofficial].
    *   Reshape `rays_o` and `rays_d` to contiguous views of shape `[-1, 3]` [S1, S2, unofficial].
    *   Determine grid resolution `H` from `density_grid.shape[0]` [S2, unofficial].
    *   Calculate `M = n_alive * n_step`. If `align > 0`, pad `M`: `M += align - (M % align)` [S1, S2, unofficial].
    *   Initialize output tensors `xyzs`, `dirs`, `deltas` with zeros of shape `[M, 3]`, `[M, 3]`, `[M, 2]` respectively [S2, unofficial].
    *   If `perturb` is true, generate random noises of size `[n_alive]`; otherwise, use zeros [S1, unofficial].
    *   Initialize `rays_alive[0]` with indices `0..n_alive-1` and `rays_t[0]` with `near` distances [S2, unofficial].
    *   Initialize rendering state: `rays_rgbd` (RGBA) to zeros, `rays_T` (transmittance) to ones, `rays_cost` to zeros if `render_cost` is True [S3, unofficial].

2.  **Step Size Calculation:**
    *   Determine `n_step` per ray: `n_step = max(min(B * N // n_alive, 8), 1)` [S2, unofficial]. (Note: `B` and `N` are batch/total ray counts).
    *   Alternatively, use adaptive step sizing based on `dt_gamma` and distance from camera [S1, S4, unofficial].

3.  **Ray Marching Loop (Per Batch/Iteration):**
    *   Loop while `n_alive > 0` and step count `< max_steps` [S2, unofficial].
    *   **Compact Alive Rays:** Use `raymarching.compact_rays(...)` to remove terminated rays from the active list [S2, unofficial].
    *   **Query CUDA Kernel:** Call `_backend.march_rays` with parameters `n_alive`, `n_step`, `rays_alive`, `rays_t`, `rays_o`, `rays_d`, `bound`, `density_bitfield`, `C`, `H`, `near`, `far`, `align`, `perturb`, `dt_gamma`, `max_steps` [S1, S2, S4, unofficial].
        *   Inside the kernel:
            *   For each alive ray, compute thresholds and perturb if needed [S2, unofficial].
            *   Loop while `t < far` and `step < n_step`:
                *   Query grid/bitfield at current `t` [S2, unofficial].
                *   If occupied, write `xyz`, `dir`, `delta` to output buffers [S2, unofficial].
                *   If empty, skip [S2, unofficial].
                *   Update `t` based on step size [S2, unofficial].
    *   **Network Evaluation:** Evaluate NeRF network at sampled points `xyzs` with `stop_gradient` on inputs to prevent gradient flow during inference [S3, unofficial].
    *   **Integration:** Call `integrate_rays_inference` to update ray colors, transmittance, and termination status, returning `terminate_cnt` [S3, unofficial].
    *   **Termination Check:** Update `n_alive` with `terminate_cnt`. If all rays are terminated, break [S3, unofficial].

4.  **Post-Processing:**
    *   Split `rays_rgbd` into RGB and depth [S3, unofficial].
    *   Convert to uint8 images [S3, unofficial].
    *   Optionally render cost map [S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import raymarching # Hypothetical CUDA backend module

def inference_ray_marching_and_integration(
    rays_o: torch.Tensor,
    rays_d: torch.Tensor,
    near: torch.Tensor,
    far: torch.Tensor,
    density_bitfield: torch.Tensor,
    density_grid: torch.Tensor,
    bound: float,
    mean_density: float,
    dt_gamma: float = 0.0,
    perturb: bool = False,
    max_steps: int = 1024,
    align: int = -1,
    n_alive: int = None
):
    """
    Performs inference ray marching and integration.
    rays_o, rays_d: [N, 3]
    near, far: [N]
    density_bitfield: [CHHH // 8]
    density_grid: [H, H, H]
    """
    # 1. Initialization
    rays_o = rays_o.contiguous().view(-1, 3)
    rays_d = rays_d.contiguous().view(-1, 3)
    
    N = rays_o.shape[0]
    if n_alive is None:
        n_alive = N
        
    # Calculate M (total output size)
    # n_step is determined dynamically based on batch size and alive rays
    # For simplicity, assume a fixed n_step or calculate as per source [S2, unofficial]
    n_step = 8 # Example value, typically max(min(B * N // n_alive, 8), 1)
    M = n_alive * n_step
    if align > 0:
        M += align - (M % align)
        
    # Initialize output tensors
    xyzs = torch.zeros(M, 3, dtype=torch.float32, device=rays_o.device)
    dirs = torch.zeros(M, 3, dtype=torch.float32, device=rays_o.device)
    deltas = torch.zeros(M, 2, dtype=torch.float32, device=rays_o.device)
    
    # Initialize alive ray buffers
    rays_alive = torch.arange(n_alive, dtype=torch.int32, device=rays_o.device)
    rays_t = near[:n_alive].clone()
    
    # Perturbation
    if perturb:
        noise = torch.rand(n_alive, device=rays_o.device)
    else:
        noise = torch.zeros(n_alive, device=rays_o.device)
        
    # 2. Marching Loop
    # Note: In practice, this is often handled by a CUDA kernel loop
    # Here we simulate the call to the backend kernel
    xyzs, dirs, deltas = raymarching.march_rays(
        n_alive=n_alive,
        n_step=n_step,
        rays_alive=rays_alive,
        rays_t=rays_t,
        rays_o=rays_o,
        rays_d=rays_d,
        bound=bound,
        density_bitfield=density_bitfield,
        C=density_bitfield.shape[0] * 8 // (density_grid.shape[0] ** 3) if density_bitfield.numel() > 0 else 0, # Placeholder for C
        H=density_grid.shape[0],
        near=near[:n_alive],
        far=far[:n_alive],
        align=align,
        perturb=perturb,
        dt_gamma=dt_gamma,
        max_steps=max_steps
    )
    
    # 3. Integration (Simplified)
    # Evaluate network at xyzs
    # sigmas, rgbs = nerf_network(xyzs)
    # Composite using front-to-back rendering
    # weights = sigmas * torch.exp(-torch.cumsum(sigmas, dim=0) * deltas[:, 0])
    # image = torch.sum(weights[..., None] * rgbs, dim=0)
    
    return xyzs, dirs, deltas

```

### Edge Cases and Pitfalls
*   **Precision:** Use `float32` for ray positions and deltas to avoid accumulation errors over long rays [S2, unofficial].
*   **Boundaries:** Ensure rays are clipped to `near` and `far` planes to avoid sampling outside the scene [S1, S2, unofficial].
*   **Empty Space:** If the density grid is sparse, ensure the occupancy check is efficient to avoid unnecessary network queries [S2, unofficial].
*   **Perturbation:** If `perturb` is enabled, ensure the random seed is consistent across frames to avoid flickering [S1, unofficial].
*   **Alignment:** If `align` is not set correctly, memory access patterns in the CUDA kernel may be inefficient [S1, S2, unofficial].

## Subtopics

### [Inference Ray Compositing](3790-inference-ray-compositing.md)
Inference Ray Compositing aggregates per-step volumetric properties (density, color, depth, ambient light, uncertainty) into final pixel buffers. It differs from training-time compositing by operating on pre-computed ray samples with strict early-termination logic to maximize throughput during real-time inference. It serves as the bridge between sparse ray marching results and the final image output, handling multiple output channels in parallel.

### [Sparse Ray Marching (Training)](3890-sparse-ray-marching-training.md)
Sparse Ray Marching (Training) is a GPU kernel that samples points along rays during the training phase of neural volume rendering. It efficiently traverses a hashed density grid to identify occupied regions while adapting step sizes and applying perturbation to maintain sampling quality. It differs from inference marchers by prioritizing training stability and memory efficiency through adaptive step sizing and perturbation, and from static allocation methods by dynamically managing sample buffers based on occupancy.

### [Chunked Volume Rendering](4310-chunked-volume-rendering.md)
Chunked Volume Rendering is a memory-management technique for image-order volume rendering that processes rays in fixed-size batches (chunks) to bound GPU memory usage during inference. It solves the problem of exceeding available VRAM when rendering high-resolution images or large ray bundles by decoupling the total ray count from the per-forward-pass memory footprint. It differs from [Sparse Ray Marching (Training)](3890-sparse-ray-marching-training.md), which optimizes sampling density during training, and [Inference Ray Compositing](3790-inference-ray-compositing.md), which focuses on the compositing logic itself; Chunked Volume Rendering specifically addresses the *batching* of the entire ray processing pipeline.

### [BRDF Evaluation Kernel](5324-brdf-evaluation-kernel.md)
The BRDF Evaluation Kernel is a CUDA kernel designed to evaluate the Bidirectional Reflectance Distribution Function (BRDF) for a batch of rays during inference ray marching and integration. It serves as the shading component within the Inference Ray Marching and Integration pipeline, transforming geometric and material inputs into final pixel colors. It is distinct from compositing operations, focusing solely on the local reflection model evaluation.

### [Inference Pipeline](5924-inference-pipeline.md)
The Inference Pipeline renders images from a trained dense point cloud (specifically 3D Gaussians) and new camera poses without requiring further optimization. It distinguishes itself from training pipelines by executing a fixed forward flow to generate visualization outputs, including color images, depth maps, and colored depth maps. It explicitly handles "near-camera" artifacts through point pruning before rendering, ensuring depth values are physically meaningful for downstream tasks like video generation or multi-view consistency checks.

### [DVGO Volume Rendering Forward](8157-dvgo-volume-rendering-forward.md)
Performs volume rendering for a batch of rays, including sampling, density/alpha computation, transmittance calculation, color estimation, and ray marching. It sets itself apart by using a direct voxel grid opacity representation where density is queried from a network and converted to alpha, often utilizing mask caching for free-space pruning and fast color thresholds for early termination.

### [Light-space opacity precomputation](9461-light-space-opacity-precomputation.md)
This method accelerates non-collocated view and light rendering by precomputing light-ray opacity accumulation to avoid expensive per-shading-point light-ray marching. It sets itself apart from naive non-collocated rendering, which connects each shading point to the light source with additional light-dependent opacity accumulation per point, by computing this accumulation once from the light's point of view and storing it in a volume.

### Gradient Magnitude Threshold Acceleration
An acceleration technique that discards voxels or ray samples with gradient magnitude below a threshold, assuming they do not lie on object surfaces [S5, unofficial]. It calculates gradient magnitude for samples and discards those smaller than the threshold [S5, unofficial]. This acceleration is advantageous because gradient estimation and interpolation are expensive operations [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of rays and $S$ is the average number of steps per ray. Early termination and space skipping reduce the effective $S$ [S2, S3, unofficial].
*   **Memory Complexity:** $O(M)$, where $M$ is the total number of samples generated. $M$ is bounded by `n_alive * n_step` and padded to `align` [S1, S2, unofficial].
*   **Performance:** Real-time inference is achievable with optimized CUDA kernels and occupancy grids [S3, unofficial]. Performance scales with GPU memory bandwidth and compute capability [S2, unofficial].
*   **Strengths:** High throughput for inference, efficient memory usage with chunking, compatibility with various density representations (hashed grid, voxel grid) [S1, S2, S3, unofficial].
*   **Limitations:** Quality may degrade with aggressive early termination or large `dt_gamma` [S1, unofficial]. Perturbation adds computational overhead [S1, unofficial].
*   **When to Use:** Real-time neural volume rendering, interactive applications, VR/AR [S3, unofficial].
*   **When Not to Use:** Training (use Sparse Ray Marching (Training)), high-fidelity offline rendering (use path tracing or higher sample counts) [S1, S2, unofficial].

## Background
Inference Ray Marching and Integration originated from the need for real-time neural rendering, particularly with the advent of Instant-NGP and similar techniques [S3, S4, unofficial]. It builds on earlier volume rendering acceleration methods like space skipping and early termination [S2, S3, unofficial]. The method is widely used in modern neural rendering frameworks for inference [S1, S2, S3, S4, unofficial].

## Sources
[S1] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
[S2] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S3] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
[S4] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
[S5] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
