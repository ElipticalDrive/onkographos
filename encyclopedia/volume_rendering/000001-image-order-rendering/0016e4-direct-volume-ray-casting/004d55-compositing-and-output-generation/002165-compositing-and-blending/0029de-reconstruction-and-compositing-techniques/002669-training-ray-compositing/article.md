# Training Ray Compositing

Also called: Volume Rendering Composite, Volume Rendering Composite Forward

## Overview
Training Ray Compositing is a differentiable volume rendering operation that composites per-ray samples of density ($\sigma$), view-dependent color ($\text{RGB}$), and ambient light contributions into final pixel colors, accumulated alpha weights, and depth values [S1, unofficial]. It is specifically designed for the training loop of neural radiance fields (NeRF), providing the forward pass for image reconstruction loss and the necessary intermediate values for the backward pass to compute gradients with respect to $\sigma$ and $\text{RGB}$ parameters [S1, unofficial]. It extends standard alpha compositing by supporting ambient light terms and custom backward kernels for efficient gradient propagation [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method takes per-sample densities ($\sigma$), colors ($\text{RGB}$), and ray parameters as input.
*   $\sigma$: Density array of size $(B, S, 1)$ or $[M]$ [S1, S2, unofficial].
*   $\text{RGB}$: Color array of size $(B, S, 3)$ or $[M, 3]$ [S1, S2, unofficial].
*   $z_{vals}$: Z-coordinates of samples, size $(B, S)$ [S1, unofficial].
*   $dirs$: Ray directions, size $(B, 3)$ [S1, unofficial].
*   $rays$: Ray metadata, shape $[N, 3]$, where each row is $[\text{ray\_index}, \text{start\_point\_index}, \text{num\_steps}]$ [S5, unofficial].
*   $deltas$: Step sizes, size $[M, 2]$ (step size and real delta) [S2, unofficial].

Outputs include:
*   $image$: Final RGB color per ray, shape $[N, 3]$ [S5, unofficial].
*   $weights\_sum$: Total alpha (opacity) per ray, shape $[N]$ [S5, unofficial].
*   $depth$: Depth value, shape $[N]$ [S2, unofficial].

**Data Structures:**
The implementation typically uses contiguous tensors for $\sigma$, $\text{RGB}$, and $z_{vals}$. Ray metadata is stored in an integer array mapping active rays to sample indices [S2, S5, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Ensure input tensors are contiguous [S2, unofficial].
    *   Initialize accumulators for transmittance $T = 1.0$, and color components $r=0, g=0, b=0$ [S5, unofficial].
    *   If `sample_at_infinity` is True, set `last_sample_z` to $10^{10}$; otherwise, set it to $10^{-19}$ [S1, unofficial].

2.  **Compute Distances:**
    *   Calculate distances between consecutive samples: $dists = [z_{vals}[1:] - z_{vals}[:-1], \text{last\_sample\_z}]$ [S1, unofficial].
    *   Scale distances by the ray direction norm: $dists = dists \cdot ||dirs||$ [S1, unofficial].

3.  **Compute Alpha and Transmittance:**
    *   For each sample $i$, compute alpha opacity:
        $$ \alpha_i = 1 - e^{-\sigma_i \cdot dists_i} $$
        [S1, S5, unofficial].
    *   Compute the accumulation product (transmittance) to handle numerical stability. Prepend $1.0$ to the cumulative product of $(1 - \alpha_i + \epsilon)$, where $\epsilon$ is a small number (default $10^{-10}$) [S1, unofficial]:
        $$ accum\_prod = [1, (1-\alpha_0+\epsilon), (1-\alpha_0+\epsilon)(1-\alpha_1+\epsilon), \dots] $$
        [S1, unofficial].
    *   Alternatively, in iterative form, update transmittance: $T_{new} = T_{old} \cdot (1 - \alpha_i)$ [S5, unofficial].

4.  **Compute Weights:**
    *   Calculate the weight for each sample:
        $$ w_i = \alpha_i \cdot accum\_prod_i $$
        [S1, unofficial].
    *   Or iteratively: $w_i = \alpha_i \cdot T$ [S5, unofficial].

5.  **Accumulate Color and Depth:**
    *   Compute final RGB color by summing the element-wise product of weights and sample colors:
        $$ rgb_{out} = \sum_{i} w_i \cdot rgb_i $$
        [S1, S5, unofficial].
    *   Compute exponential depth:
        $$ depth_{exp} = \sum_{i} w_i \cdot z_i $$
        [S1, unofficial].
    *   Compute accumulation (total opacity):
        $$ acc = \sum_{i} w_i $$
        [S1, unofficial].

6.  **Background Compositing:**
    *   If `use_white_background` is True, add the background color (white) to transparent regions:
        $$ rgb_{out} = rgb_{out} + (1 - acc) \cdot 1.0 $$
        [S1, S3, unofficial].

7.  **Early Termination:**
    *   During the loop, if transmittance $T$ drops below a threshold (default $T_{thresh} = 10^{-4}$), mark the ray as dead and break the sample loop [S4, S5, S6, unofficial].

8.  **Special Handling for Infinity:**
    *   If `sample_at_infinity` is True, the accumulation $acc$ excludes the last sample weight: $acc = \sum_{i=0}^{S-2} w_i$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def training_ray_compositing_forward(sigmas, rgbs, z_vals, dirs, rays, 
                                      use_white_background=False, 
                                      sample_at_infinity=False, 
                                      T_thresh=1e-4, eps=1e-10):
    """
    Forward pass for Training Ray Compositing.
    
    Args:
        sigmas: Density array, shape (B, S, 1) or (M,)
        rgbs: Color array, shape (B, S, 3) or (M, 3)
        z_vals: Z-coordinates, shape (B, S)
        dirs: Ray directions, shape (B, 3)
        rays: Ray metadata, shape (N, 3) -> [ray_index, start_point_index, num_steps]
        use_white_background: Boolean, add white background if True
        sample_at_infinity: Boolean, handle sample at infinity
        T_thresh: Float, threshold for early termination
        eps: Float, small number for numerical stability
        
    Returns:
        image: Final RGB colors, shape (N, 3)
        weights_sum: Total alpha per ray, shape (N)
        depth: Exponential depth, shape (N)
    """
    # Initialize outputs
    N = rays.shape[0]
    image = torch.zeros(N, 3, dtype=sigmas.dtype)
    weights_sum = torch.zeros(N, dtype=sigmas.dtype)
    depth = torch.zeros(N, dtype=sigmas.dtype)
    
    # Handle contiguous inputs if necessary
    sigmas = sigmas.contiguous()
    rgbs = rgbs.contiguous()
    z_vals = z_vals.contiguous()
    dirs = dirs.contiguous()
    
    # Compute distances
    # dists = [z_vals[1:] - z_vals[:-1], last_sample_z]
    # Scale by ray direction norm
    dists = torch.diff(z_vals, dim=-1)
    if sample_at_infinity:
        last_sample_z = 1e10
    else:
        last_sample_z = 1e-19
    dists = torch.cat([dists, torch.full((dists.shape[0], 1), last_sample_z, dtype=dists.dtype)], dim=-1)
    dists = dists * torch.norm(dirs, dim=-1, keepdim=True)
    
    # Compute alpha
    alpha = 1.0 - torch.exp(-sigmas.squeeze(-1) * dists)
    
    # Compute transmittance (accum_prod)
    # accum_prod = [1, (1-alpha_0+eps), (1-alpha_0+eps)*(1-alpha_1+eps), ...]
    transmittance_factors = (1.0 - alpha + eps)
    accum_prod = torch.ones_like(alpha)
    accum_prod[:, 1:] = torch.cumprod(transmittance_factors[:, :-1], dim=-1)
    
    # Compute weights
    weights = alpha * accum_prod
    
    # Compute RGB and Depth
    # rgb_out = sum(weights * rgbs)
    rgb_out = torch.sum(weights.unsqueeze(-1) * rgbs, dim=1)
    
    # depth_exp = sum(weights * z_vals)
    depth_exp = torch.sum(weights * z_vals, dim=1)
    
    # Compute accumulation
    acc = torch.sum(weights, dim=1)
    
    # Background compositing
    if use_white_background:
        rgb_out = rgb_out + (1.0 - acc.unsqueeze(-1)) * 1.0
        
    # Handle sample_at_infinity for accumulation
    if sample_at_infinity:
        # Exclude last sample weight from accumulation
        acc = torch.sum(weights[:, :-1], dim=1)
        if use_white_background:
            rgb_out = rgb_out + (1.0 - acc.unsqueeze(-1)) * 1.0
            
    # Map results to rays
    # rays[n] = [ray_index, start_point_index, num_steps]
    # Note: This mapping assumes a specific layout where rays define which samples belong to which ray.
    # In practice, this is often handled by a CUDA kernel or scatter operation.
    # Here we simulate the mapping for a single ray per sample for simplicity, 
    # but real implementation uses rays metadata.
    
    for n in range(N):
        ray_idx = int(rays[n, 0])
        start_idx = int(rays[n, 1])
        num_steps = int(rays[n, 2])
        
        if num_steps == 0:
            continue
            
        # Extract samples for this ray
        # Assuming samples are stored in a flat array or batched structure
        # For this pseudocode, we assume rgbs, sigmas, z_vals are batched by ray
        # In reality, 'rays' metadata is used to index into a flat list of samples
        
        # Simplified: Assume we have pre-extracted ray samples
        # ray_rgbs = rgbs[start_idx:start_idx+num_steps]
        # ray_sigmas = sigmas[start_idx:start_idx+num_steps]
        # ray_z_vals = z_vals[start_idx:start_idx+num_steps]
        
        # Re-compute for the ray
        # This part is highly dependent on the specific data layout (batched vs flat)
        # The core logic is:
        # alpha = 1 - exp(-sigma * dist)
        # T = prod(1 - alpha)
        # w = alpha * T
        # C = sum(w * rgb)
        # D = sum(w * z)
        # A = sum(w)
        
        # Early termination check would happen inside the loop
        # if T < T_thresh: break
        
        # For the purpose of this article, we assume the above vectorized operations
        # are performed per-ray or in a batched manner using the rays metadata.
        
        image[ray_idx] = rgb_out[n] # Simplified mapping
        weights_sum[ray_idx] = acc[n]
        depth[ray_idx] = depth_exp[n]
        
    return image, weights_sum, depth
```

**Backward Pass:**
The backward pass computes gradients for $\sigma$ and $\text{RGB}$ using the saved tensors from the forward pass ($\sigma$, $\text{RGB}$, $\text{deltas}$, $\text{rays}$, $\text{weights\_sum}$, $\text{depth}$, $\text{image}$) [S6, unofficial]. It propagates gradients through the alpha compositing chain, handling the transmittance product [S1, S5, unofficial]. Note that the gradient for depth is not propagated to $\sigma$ [S2, S6, unofficial].

## Subtopics

### [Training Ray Compositing (Sigma)](3868-training-ray-compositing-sigma.md)
Training Ray Compositing (Sigma) is a differentiable volume rendering operation that composites per-ray samples of density ($\sigma$), view-dependent color ($\text{RGB}$), and ambient light contributions into final pixel colors, accumulated alpha weights, and depth values. It is specifically designed for the training loop of neural radiance fields, providing the forward pass for image reconstruction loss and the necessary intermediate values for the backward pass to compute gradients with respect to $\sigma$ and $\text{RGB}$ parameters. It extends standard alpha compositing by supporting ambient light terms and custom backward kernels for efficient gradient propagation.

### [Ray Functions](10623-ray-functions.md)
Ray functions determine the final color and opacity contribution of a ray passing through a volume by aggregating sample values along the ray path. While standard compositing accumulates opacity and color, alternative ray functions such as Maximum Intensity Projection (MIP), average value, and distance-to-threshold provide different visualizations where the result is a scalar value or distance rather than an accumulated color. The alpha compositing ray function is distinct because its result is not a scalar value or distance that can be represented on the ray profile, but rather an accumulated color and alpha weight.

### [Multi-View Color Blending](12853-multi-view-color-blending.md)
Multi-View Color Blending fuses original colors from $N$ reference views into query points using view-dependent weights derived from feature similarity and view direction. It differs from standard alpha compositing by explicitly aggregating multi-view pixel features before density/color prediction, addressing view-dependent appearance and occlusion in generalizable neural radiance fields.

### Multi-Layer Compositing with Gradient Backpropagation
Caps the number of composited layers at $c_{max} = 16$ per pixel to bound memory for backward pass statistics [S7, unofficial]. The forward pass performs depth-interval grouping, MIP-filtered kernel evaluation, Shepard normalization, and front-to-back compositing fused into a single kernel invocation per tile [S7, unofficial]. The backward pass first replays forward gap detection to reconstruct per-layer totals ($C_c, \alpha_c$) for $c=1$ to $c_{max}$, then retraverses primitives, assigns each to its layer, and decomposes per-layer gradients into per-surfel gradients through normalized accumulation using atomic operations [S7, unofficial]. Concurrent writes from different pixels during gradient decomposition are handled by atomic operations [S7, unofficial].

### Partial Image Source Signature Integration
A post-processing step that combines pre-computed ray buffer data (absorption/color) with a source signature vector to compute final emerging intensities, using OpenMP parallelization [S8, unofficial]. It retrieves control portals for the buffer and intensity arrays, determines pixel and channel counts, and iterates over all pixels in parallel [S8, unofficial]. For each pixel and channel, it computes the new intensity as: $\text{intensity} = \text{emission} + \text{buffer} \cdot \text{source\_sig[channel]}$ [S8, unofficial]. The buffer stores color or absorption values per pixel/channel, and the intensities store the final emerging intensity [S8, unofficial].

### Shaded Compositing
Modifies the compositing ray function to include shading by computing ambient, diffuse, and specular illumination at each sample using estimated volume gradients [S9, book]. The shaded compositing equation is:
$$ I(t_{near}, \vec{\omega}) = \sum_{t = t_{near}}^{t \leq t_{far}} \alpha(t)(I_a + I_d + I_s) \prod_{t' = t_{near}}^{t' < t_{far}}(1 - a(t')) $$
where $I_a, I_d, I_s$ are ambient, diffuse, specular illumination computed using estimated volume gradient as normal, and $\alpha(t)$ is light reflected per unit length [S9, book]. This requires estimating gradients at many locations within the volume for each pixel, making 3D gradient estimation more suitable than 2D [S9, book].

### Kernel-based Compositing
A compositing method using interpolation kernels where color and opacity are accumulated based on kernel weights [S10, unofficial]. The formulas are:
$$ c = c_s \cdot \alpha_s \cdot (1 - \alpha) $$
$$ \alpha = \alpha_s \cdot (1 - \alpha) + \alpha $$
where $c$ is the resulting color, $c_s$ is sample color, $\alpha_s$ is sample opacity, and $\alpha$ is the accumulated opacity [S10, unofficial].

## Operating characteristics
Training Ray Compositing is optimized for GPU execution using CUDA kernels, enabling efficient forward and backward passes for neural network training [S5, S6, unofficial]. The use of early termination ($T < T_{thresh}$) reduces computational cost for opaque regions [S4, S5, unofficial]. The method supports batched processing of multiple rays, improving throughput [S1, S5, unofficial]. Memory usage is proportional to the number of samples and rays, with additional storage for backward pass intermediates [S6, S7, unofficial]. The method is particularly effective for NeRF training, where differentiability is crucial [S1, unofficial].

## Background
Training Ray Compositing was introduced as part of the NeRFies framework for dynamic neural radiance fields [S1, unofficial]. It has since been adopted in various other NeRF implementations, such as ER-NeRF [S2, unofficial], Instant-NGP [S5, unofficial], and IDDR-NGP [S6, unofficial], often with optimizations for specific hardware or data structures. The method builds on the standard volume rendering integral [S12, S43, S50, unofficial] and adapts it for differentiable training.

## Sources
[S1] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
[S2] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
[S3] AlexSheldrick/density_bounded_ray_weights, https://github.com/AlexSheldrick/density_bounded_ray_weights, fetched 2026-10-27 (unofficial)
[S4] Linyou/taichi-ngp-renderer, https://github.com/Linyou/taichi-ngp-renderer, fetched 2026-10-27 (unofficial)
[S5] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S6] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
[S7] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S8] llnl/rover, https://github.com/llnl/rover, fetched 2026-10-26 (unofficial)
[S9] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S10] Volume Rendering Volume Rendering. Lecture 21. Acknowledgements. These slides are collected from many sources. A particularly valuable source is the IEEE Visualization conference tutorials. Sources fr, https://www.slideserve.com/hayes-levine/volume-rendering-powerpoint-ppt-presentation, fetched 2026-10-25 (unofficial)
