# Training Ray Compositing

Also called: Volume Rendering Composite, Volume Rendering Composite Forward

## Overview
Training Ray Compositing is a differentiable volume rendering method that composites per-ray samples of density ($\sigma$) and color ($\mathbf{c}$) into final image colors, alpha weights, and depth, while simultaneously computing gradients with respect to the input parameters for training [S1, unofficial]. It extends standard alpha compositing by including ambient light contributions and supporting a custom backward pass to propagate gradients from the loss function back to the density and color fields [S1, unofficial]. It is distinguished from its sibling [Training Ray Compositing (Backward)](3893-training-ray-compositing-backward.md) by handling the forward composition and the joint forward-backward logic in a unified CUDA kernel interface, whereas the backward variant focuses specifically on the gradient computation logic [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method takes the following inputs:
- `sigmas`: Density values, float, shape $[M]$ [S1, unofficial].
- `rgbs`: RGB colors at sampled points, float, shape $[M, 3]$ [S1, unofficial].
- `deltas`: Step sizes, float, shape $[M, 2]$ (containing step size and real delta) [S1, unofficial].
- `rays`: Ray metadata, int32, shape $[N, 3]$. Each row is $[ray\_index, start\_point\_index, num\_steps]$ [S1, unofficial].
- `T_thresh`: Transparency threshold for early termination, float, default $10^{-4}$ [S1, unofficial].

The method produces the following outputs:
- `image`: Final RGB color per ray, float, shape $[N, 3]$ [S1, unofficial].
- `weights_sum`: Total alpha (opacity) per ray, float, shape $[N]$ [S1, unofficial].
- `depth`: Depth value per ray, float, shape $[N]$ [S1, unofficial].
- `grad_sigmas`: Gradient with respect to density, float, shape $[M]$ [S3, unofficial].
- `grad_rgbs`: Gradient with respect to color, float, shape $[M, 3]$ [S3, unofficial].

**Data Structures:**
The implementation typically uses contiguous CUDA tensors. The ray metadata `rays` maps active rays to their sample data. The `weights_sum` buffer tracks the accumulated opacity for early termination checks.

**Algorithm Steps:**
1. **Initialization**: Ensure input tensors are contiguous. Initialize accumulators for the current ray: `rgb_temp = 0`, `depth_temp = 0`, `opacity_temp = 0`, and transmittance `T = 1.0` [S2, unofficial].
2. **Ray Iteration**: Iterate over active rays $n$. Retrieve the number of samples $N_{samples}$ for the current ray [S2, unofficial].
3. **Sample Loop**: For each sample $s$ in $0 \dots N_{samples}-1$:
    a. **Alpha Calculation**: Compute the sample opacity $\alpha$ using the density $\sigma$ and step size $\delta$:
       $$ \alpha = 1 - e^{-\sigma \delta} $$
       where $\sigma$ is `sigmas[s]` and $\delta$ is `deltas[s]` [S3, unofficial].
    b. **Weight Calculation**: Compute the weight $w$ for the current sample:
       $$ w = \alpha \cdot T $$
       where $T$ is the current accumulated transmittance [S3, unofficial].
    c. **Accumulation**:
       - Update RGB: `rgb_temp += w * rgbs[s]` [S3, unofficial].
       - Update Depth: `depth_temp += w * ts[s]` (where `ts` is the depth/time of the sample) [S2, unofficial].
       - Update Opacity: `opacity_temp += w` [S2, unofficial].
    d. **Transmittance Update**: Update the transmittance for the next sample:
       $$ T_{new} = T_{old} \cdot (1 - \alpha) $$
       [S3, unofficial].
    e. **Early Termination**: If $T < T_{thresh}$ (default $10^{-4}$), mark the ray as dead and break the sample loop [S2, unofficial].
4. **Finalization**: Add accumulated values to the final buffers:
   - `self.rgb[r] += rgb_temp`
   - `self.depth[r] += depth_temp`
   - `self.opacity[r] += opacity_temp`
   - `weights_sum[index] = 1.0 - T` [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def composite_rays_train_forward(sigmas, rgbs, deltas, rays, T_thresh=1e-4):
    """
    Forward pass for Training Ray Compositing.
    sigmas: [M] float
    rgbs: [M, 3] float
    deltas: [M, 2] float (step_size, real_delta)
    rays: [N, 3] int32 (ray_index, start_point_index, num_steps)
    """
    N = rays.shape[0]
    M = sigmas.shape[0]
    
    # Output buffers
    image = torch.zeros((N, 3), dtype=torch.float32)
    weights_sum = torch.zeros((N), dtype=torch.float32)
    depth = torch.zeros((N), dtype=torch.float32)
    
    # Save tensors for backward pass
    # Note: In a real CUDA kernel, this is handled by the autograd engine
    saved_tensors = (sigmas, rgbs, deltas, rays, weights_sum, depth, image)
    
    for n in range(N):
        ray_idx = rays[n, 0]
        start_idx = rays[n, 1]
        num_steps = rays[n, 2]
        
        if num_steps == 0:
            continue
            
        # Initialize accumulators
        T = 1.0
        rgb_acc = torch.zeros(3, dtype=torch.float32)
        depth_acc = 0.0
        opacity_acc = 0.0
        
        for s in range(num_steps):
            global_idx = start_idx + s
            
            # Get sample data
            sigma = sigmas[global_idx]
            rgb = rgbs[global_idx]
            delta = deltas[global_idx, 0] # Step size
            ts = deltas[global_idx, 1]   # Depth/Time
            
            # Alpha calculation: alpha = 1 - exp(-sigma * delta)
            alpha = 1.0 - torch.exp(-sigma * delta)
            
            # Weight calculation: w = alpha * T
            w = alpha * T
            
            # Accumulate
            rgb_acc += w * rgb
            depth_acc += w * ts
            opacity_acc += w
            
            # Update transmittance: T = T * (1 - alpha)
            T *= (1.0 - alpha)
            
            # Early termination
            if T < T_thresh:
                break
        
        # Write results
        image[ray_idx] = rgb_acc
        weights_sum[ray_idx] = 1.0 - T
        depth[ray_idx] = depth_acc
        
    return image, weights_sum, depth, saved_tensors

def composite_rays_train_backward(grad_back, saved_tensors):
    """
    Backward pass for Training Ray Compositing.
    Computes gradients for sigmas and rgbs.
    """
    sigmas, rgbs, deltas, rays, weights_sum, depth, image = saved_tensors
    
    # Gradient for depth is not propagated to sigmas [S1, unofficial]
    # The backward pass computes gradients for sigmas, rgbs, and ambient [S1, unofficial]
    
    # Implementation details for backward pass are complex and typically
    # handled by a separate CUDA kernel or autograd function.
    # The core logic involves propagating gradients through the alpha compositing chain.
    
    # Placeholder for the actual backward kernel logic which is non-trivial
    # and depends on the specific implementation of the custom CUDA backend.
    # Typically involves:
    # 1. Computing dL/dC_acc and dL/dT_acc from grad_back
    # 2. Propagating back through the compositing equation to get dL/d_sigma and dL/d_rgb
    
    grad_sigmas = torch.zeros_like(sigmas)
    grad_rgbs = torch.zeros_like(rgbs)
    
    # ... (Actual backward kernel implementation would go here) ...
    
    return grad_sigmas, grad_rgbs, None, None, None
```

**Parameters:**
- `T_thresh`: Transparency threshold for early termination. Default $10^{-4}$ [S1, unofficial].
- `N_THREAD`: Number of threads per block in CUDA implementation, typically 256 [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Empty Rays**: If `num_steps` is 0, the ray should be skipped or initialized to 0 [S3, unofficial].
- **Overflow**: Check for `offset + num_steps >= M` to prevent out-of-bounds access [S3, unofficial].
- **Gradient for Depth**: The gradient for depth is not propagated to sigmas [S1, unofficial].
- **Precision**: Use `float32` for standard training, but `float64` may be needed for high-precision requirements [S1, unofficial].

## Subtopics

### [Training Ray Compositing (Sigma)](3868-training-ray-compositing-sigma.md)
Training Ray Compositing (Sigma) is a differentiable volume rendering method that composites per-ray samples of density ($\sigma$) and color ($\mathbf{c}$) into final image colors, alpha weights, and depth, while simultaneously computing gradients with respect to the input parameters for training. It extends standard alpha compositing by including ambient light contributions and supporting a custom backward pass to propagate gradients from the loss function back to the density and color fields. It is distinguished from its sibling [Training Ray Compositing (Backward)](3893-training-ray-compositing-backward.md) by handling the forward composition and the joint forward-backward logic in a unified CUDA kernel interface, whereas the backward variant focuses specifically on the gradient computation logic.

### [Ray Functions](10623-ray-functions.md)
Ray functions determine the final color and opacity contribution of a ray passing through a volume by aggregating sample values along the ray path. While standard compositing accumulates opacity and color, alternative ray functions such as Maximum Intensity Projection (MIP), average value, and distance-to-threshold provide different visualizations where the result is a scalar value or distance rather than an accumulated color. The alpha compositing ray function is distinct because its result is not a scalar value or distance that can be represented on the ray profile, but rather an accumulated color and alpha weight.

### Multi-Layer Compositing with Gradient Backpropagation
Multi-Layer Compositing with Gradient Backpropagation caps the number of composited layers at $c_{max} = 16$ per pixel to bound memory for backward pass statistics [S5, unofficial]. The forward pass performs depth-interval grouping, MIP-filtered kernel evaluation, Shepard normalization, and front-to-back compositing fused into a single kernel invocation per tile [S5, unofficial]. The backward pass first replays forward gap detection to reconstruct per-layer totals $(C_c, \alpha_c)$ for $c=1$ to $c_{max}$, then retraverses primitives, assigns each to its layer, and decomposes per-layer gradients into per-surfel gradients through normalized accumulation using atomic operations [S5, unofficial]. Concurrent writes from different pixels during gradient decomposition are handled by atomic operations [S5, unofficial].

### Partial Image Source Signature Integration
Partial Image Source Signature Integration is a post-processing step that combines pre-computed ray buffer data (absorption/color) with a source signature vector to compute final emerging intensities, using OpenMP parallelization [S6, unofficial]. It retrieves control portals for the buffer and intensity arrays, determines pixel and channel counts, and iterates over all pixels in parallel. For each pixel and channel, it computes the new intensity as: $intensity = emission + buffer \cdot source\_sig[channel]$ [S6, unofficial]. The data structures include `PartialImage::m_buffer` (storing color or absorption values), `PartialImage::m_intensities` (storing final intensity), and `PartialImage::m_source_sig` (per-channel source signature weights) [S6, unofficial].

### Shaded Compositing
Shaded Compositing modifies the compositing ray function to include shading by computing ambient, diffuse, and specular illumination at each sample using estimated volume gradients [S7, book]. The shaded compositing equation is:
$$ I(t_{near}, \vec{\omega}) = \sum_{t = t_{near}}^{t \leq t_{far}} \alpha(t)(I_a + I_d + I_s) \prod_{t' = t_{near}}^{t' < t_{far}}(1 - a(t')) $$
where $I_a, I_d, I_s$ are ambient, diffuse, specular illumination computed using estimated volume gradient as normal, and $\alpha(t)$ is light reflected per unit length [S7, book]. This requires estimating gradients at many locations within the volume for each pixel, making 3D gradient estimation more suitable than 2D [S7, book].

## Operating characteristics
- **Time Complexity**: $O(N \cdot M)$ where $N$ is the number of rays and $M$ is the number of samples per ray. Early termination can reduce the effective $M$.
- **Memory Complexity**: $O(N \cdot 3 + N + N)$ for outputs, plus $O(M \cdot (1 + 3 + 2))$ for inputs. The backward pass requires storing intermediate values for gradient computation.
- **Measured Performance**: Performance depends on the hardware and resolution. CUDA implementations can achieve high throughput by parallelizing ray processing [S3, unofficial].
- **Scaling**: Scales well with the number of rays and samples, but memory usage can become a bottleneck for high-resolution scenes or large numbers of samples.
- **Strengths**: Differentiable, allowing for end-to-end training of neural radiance fields. Supports early termination for efficiency.
- **Limitations**: Gradient computation can be complex and memory-intensive. Early termination may lead to biased gradients if not handled carefully.
- **When to Use**: When training neural radiance fields or other differentiable volume rendering models.
- **When Not to Use**: When real-time inference without training is required, or when memory constraints are severe.

## Background
Training Ray Compositing is a key component in the training pipeline of Neural Radiance Fields (NeRF) and related methods. It was introduced in the context of efficient region-aware neural radiance fields for high-fidelity talking portrait synthesis [S1, unofficial]. The method builds on the standard volume rendering integral and adapts it for differentiable programming frameworks like PyTorch.

## Sources
[S1] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
[S2] Linyou/taichi-ngp-renderer, https://github.com/Linyou/taichi-ngp-renderer, fetched 2026-10-27 (unofficial)
[S3] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S4] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
[S5] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S6] llnl/rover, https://github.com/llnl/rover, fetched 2026-10-26 (unofficial)
[S7] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
