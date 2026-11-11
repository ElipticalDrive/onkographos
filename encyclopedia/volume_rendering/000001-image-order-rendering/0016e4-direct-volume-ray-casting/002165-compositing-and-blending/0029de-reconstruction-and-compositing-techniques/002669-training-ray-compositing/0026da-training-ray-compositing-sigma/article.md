# Training Ray Compositing (Sigma)

Also called: Ray Compositing for Training (Sigma + Ambient)

## Overview
Training Ray Compositing (Sigma) is a differentiable volume rendering operation that composites per-ray samples of density ($\sigma$), view-dependent color ($\text{RGB}$), and ambient light contributions into final pixel colors, accumulated alpha weights, and depth values [S1, unofficial]. It is specifically designed for the training loop of neural radiance fields, providing the forward pass for image reconstruction loss and the necessary intermediate values for the backward pass to compute gradients with respect to $\sigma$ and $\text{RGB}$ parameters [S1, unofficial]. It extends standard alpha compositing by supporting ambient light terms and custom backward kernels for efficient gradient propagation [S1, unofficial].

## How it works
The method performs alpha compositing along rays, accumulating transparency and weighted color/depth contributions. It supports an additional ambient light term per sample. The implementation relies on a CUDA backend kernel for the forward pass and a separate kernel for the backward pass [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   `sigmas`: Float tensor of shape $[M,]$, representing volume density at $M$ sample points [S1, unofficial].
*   `rgbs`: Float tensor of shape $[M, 3]$, representing view-dependent color at $M$ sample points [S1, unofficial].
*   `ambient`: Float tensor of shape $[M,]$, representing ambient light contribution at $M$ sample points [S1, unofficial].
*   `deltas`: Float tensor of shape $[M, 2]$, representing step sizes (distances) between samples [S1, unofficial].
*   `rays`: Integer tensor of shape $[N, 3]$, containing ray indices or identifiers mapping samples to $N$ rays [S1, unofficial].
*   `T_thresh`: Float, transparency threshold. Default $10^{-4}$ [S1, unofficial].

**Outputs:**
*   `image`: Float tensor of shape $[N, 3]$, the final composited RGB color for each of the $N$ rays [S1, unofficial].
*   `weights_sum`: Float tensor of shape $[N,]$, the accumulated alpha (opacity) weights for each ray [S1, unofficial].
*   `ambient_sum`: Float tensor of shape $[N,]$, the accumulated ambient light contribution for each ray [S1, unofficial].
*   `depth`: Float tensor of shape $[N,]$, the accumulated depth for each ray [S1, unofficial].

### Data Structures and Layout
The input tensors must be contiguous in memory before kernel invocation [S1, unofficial]. The output tensors are allocated with size $N$ (number of rays) or $[N, 3]$ for color [S1, unofficial].

### Algorithm Steps
1.  **Validation**: Ensure `sigmas`, `rgbs`, and `ambient` are contiguous tensors [S1, unofficial].
2.  **Dimension Extraction**: Let $M$ be the number of samples (`sigmas.shape[0]`) and $N$ be the number of rays (`rays.shape[0]`) [S1, unofficial].
3.  **Allocation**: Allocate output tensors `weights_sum` $[N]$, `ambient_sum` $[N]$, `depth` $[N]$, and `image` $[N, 3]$ initialized to zero [S1, unofficial].
4.  **Forward Kernel**: Invoke the CUDA kernel `_backend.composite_rays_train_sigma_forward` with inputs `sigmas`, `rgbs`, `ambient`, `deltas`, `rays`, `T_thresh`, and outputs `weights_sum`, `ambient_sum`, `depth`, `image` [S1, unofficial].
5.  **Save for Backward**: Store `sigmas`, `rgbs`, `ambient`, `deltas`, `rays`, `weights_sum`, `ambient_sum`, `depth`, `image`, $M$, $N$, and $T\_thresh$ in the autograd engine's save buffer [S1, unofficial].
6.  **Return**: Return `image`, `weights_sum`, `ambient_sum`, and `depth` [S1, unofficial].

### Backward Pass
1.  **Gradient Validation**: Ensure `grad_weights_sum`, `grad_ambient_sum`, and `grad_image` are contiguous [S1, unofficial].
2.  **Retrieve**: Load saved tensors and dimensions from the forward pass [S1, unofficial].
3.  **Initialize**: Create zero-initialized gradient tensors `grad_sigmas` $[M]$, `grad_rgbs` $[M, 3]$, and `grad_ambient` $[M]$ [S1, unofficial].
4.  **Backward Kernel**: Invoke `_backend.composite_rays_train_sigma_backward` to compute gradients for `sigmas` and `rgbs` using the chain rule from `grad_image` and `grad_weights_sum` [S1, unofficial].
5.  **Ambient Gradient**: Invoke `_backend.composite_rays_ambient_sigma` to compute gradients for `ambient` using `grad_ambient_sum` [S1, unofficial].
6.  **Return**: Return `grad_sigmas`, `grad_rgbs`, `grad_ambient`, and `None` for other inputs [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.autograd as autograd

class CompositeRaysTrainSigma(autograd.Function):
    @staticmethod
    def forward(ctx, sigmas, rgbs, ambient, deltas, rays, T_thresh=1e-4):
        # Ensure contiguous
        sigmas = sigmas.contiguous()
        rgbs = rgbs.contiguous()
        ambient = ambient.contiguous()
        
        M = sigmas.shape[0]
        N = rays.shape[0]
        
        # Allocate outputs
        weights_sum = torch.zeros(N, device=sigmas.device)
        ambient_sum = torch.zeros(N, device=sigmas.device)
        depth = torch.zeros(N, device=sigmas.device)
        image = torch.zeros(N, 3, device=sigmas.device)
        
        # Call forward kernel
        # _backend.composite_rays_train_sigma_forward(
        #     sigmas, rgbs, ambient, deltas, rays, 
        #     weights_sum, ambient_sum, depth, image, T_thresh
        # )
        # [G] Placeholder for actual CUDA kernel call. In a real implementation, 
        # this would invoke the custom C++/CUDA extension.
        
        # Save for backward
        ctx.save_for_backward(sigmas, rgbs, ambient, deltas, rays, 
                              weights_sum, ambient_sum, depth, image)
        ctx.T_thresh = T_thresh
        
        return image, weights_sum, ambient_sum, depth

    @staticmethod
    def backward(ctx, grad_image, grad_weights_sum, grad_ambient_sum, grad_depth):
        # Retrieve saved tensors
        sigmas, rgbs, ambient, deltas, rays, weights_sum, ambient_sum, depth, image = ctx.saved_tensors
        T_thresh = ctx.T_thresh
        
        # Ensure gradients are contiguous
        grad_image = grad_image.contiguous()
        grad_weights_sum = grad_weights_sum.contiguous()
        grad_ambient_sum = grad_ambient_sum.contiguous()
        
        # Initialize gradients
        grad_sigmas = torch.zeros_like(sigmas)
        grad_rgbs = torch.zeros_like(rgbs)
        grad_ambient = torch.zeros_like(ambient)
        
        # Call backward kernel for sigmas and rgbs
        # _backend.composite_rays_train_sigma_backward(
        #     sigmas, rgbs, ambient, deltas, rays,
        #     weights_sum, ambient_sum, depth, image,
        #     grad_image, grad_weights_sum,
        #     grad_sigmas, grad_rgbs
        # )
        # [G] Placeholder for actual CUDA kernel call.
        
        # Call kernel for ambient gradients
        # _backend.composite_rays_ambient_sigma(
        #     sigmas, ambient, deltas, rays,
        #     ambient_sum,
        #     grad_ambient_sum,
        #     grad_ambient
        # )
        # [G] Placeholder for actual CUDA kernel call.
        
        # Note: grad_depth is not propagated to sigmas [S1, unofficial]
        
        return grad_sigmas, grad_rgbs, grad_ambient, None, None, None
```

### Parameters
*   `T_thresh`: Float, transparency threshold. Default $10^{-4}$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Depth Gradient**: The gradient for the depth output is not propagated back to the `sigmas` parameters [S1, unofficial].
*   **Contiguity**: Input tensors must be contiguous; non-contiguous tensors will cause kernel errors or incorrect results [S1, unofficial].

## Subtopics

### [Training Ray Compositing (Backward)](3893-training-ray-compositing-backward.md)
This method computes the gradients of the volume rendering loss with respect to the input density (`sigmas`) and color (`rgbs`) parameters during the backward pass of differentiable volume rendering. It implements the chain rule to propagate gradients from the final rendered image and the total accumulated alpha weights back to the individual sample points along each ray. It is the backward counterpart to [Training Ray Compositing (Sigma)](3868-training-ray-compositing-sigma.md).

### [Gradient Scaling by Distance Squared](8937-gradient-scaling-by-distance-squared.md)
Gradient Scaling by Distance Squared is a technique used during the backpropagation phase of training ray compositing to improve training stability for distant objects. It scales the gradients of field outputs based on the squared distance of samples from the camera. This method is a specific variant of [Training Ray Compositing (Sigma)](3868-training-ray-compositing-sigma.md) that addresses the imbalance in gradient magnitudes caused by varying sample depths along a ray.

### Ray Compositing for Training (Uncertainty)
Composites rays for training including uncertainty estimation, computing gradients for sigmas, RGBs, ambient, and uncertainty [S1, unofficial]. It ensures uncertainty tensors are contiguous, allocates `uncertainty_sum` $[N]$, and calls `_backend.composite_rays_train_uncertainty_forward` and `_backward` kernels. The parameter `T_thresh` defaults to $10^{-4}$ [S1, unofficial].

### Ray Compositing for Training (Triplane/Aud/Eye)
Composites rays for training with triplane-based ambient lighting (auditory and eye-specific), computing gradients for sigmas, RGBs, and various ambient components [S1, unofficial]. It ensures `amb_aud`, `amb_eye`, and `uncertainty` are contiguous, allocates their respective sums, and calls `_backend.composite_rays_train_triplane_forward`. The parameter `T_thresh` defaults to $10^{-4}$ [S1, unofficial]. Like the base method, the gradient for depth is not propagated to sigmas [S1, unofficial].

### CUDA Ray Compositing
A GPU-accelerated kernel that performs alpha compositing of pre-sampled RGB and sigma values along rays to produce final image and depth [S2, unofficial]. It iterates through samples for each alive ray, accumulating transparency and computing weighted RGB and depth contributions [S2, unofficial]. Inputs are `sigmas` $[n\_alive \times n\_step]$, `rgbs` $[n\_alive \times n\_step, 3]$, and `deltas` $[n\_alive \times n\_step, 2]$. Outputs `weights_sum` $[N]$, `depth` $[N]$, and `image` $[N, 3]$ are in-place [S2, unofficial]. The parameter `T_thresh` is the transparency threshold below which rays are considered fully opaque [S2, unofficial].

## Operating characteristics
*   **Complexity**: Time complexity is linear with respect to the number of samples $M$ and rays $N$, $O(M + N)$, as each sample is processed once per ray it belongs to. Memory complexity is $O(M)$ for inputs and $O(N)$ for outputs [S1, unofficial][S2, unofficial].
*   **Performance**: Implemented as a CUDA kernel, it is optimized for GPU execution, leveraging parallel processing for ray compositing [S2, unofficial].
*   **Strengths**: Differentiable, allowing gradient computation for training neural fields. Supports ambient lighting and uncertainty terms [S1, unofficial].
*   **Limitations**: Requires contiguous memory layout. Depth gradients are not propagated, which may limit certain optimization scenarios [S1, unofficial].
*   **When to use**: During the training phase of neural radiance fields where gradients with respect to density and color are needed [S1, unofficial].
*   **When not to use**: For inference-only scenarios where a simpler, non-differentiable compositing might suffice, or if depth gradients are required for the optimization loop.

## Background
Training Ray Compositing (Sigma) is part of the ER-NeRF (Efficient Radiance Fields) framework, which integrates audio and eye features for dynamic human head rendering [S1, unofficial]. The method was developed to support the training of these complex neural fields by providing a differentiable compositing operation that handles not just color and density, but also ambient lighting and uncertainty estimates [S1, unofficial]. It builds on the principles of differentiable volume rendering [S1, unofficial].

## Sources
[S1] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
[S2] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
