# Scatter-Add Accumulation

Also called: Accumulation Renderer, Accumulation Rendering, Ray Accumulation

## Overview
Scatter-Add Accumulation is a compositing technique used in direct volume ray casting and neural rendering to aggregate per-sample color and opacity values into final pixel colors when samples are not processed in strict ray order. It solves the problem of non-deterministic or parallelized sample generation by using atomic scatter-add operations to sum weighted contributions into a per-ray output buffer. Unlike standard front-to-back compositing which requires sorted samples, this method supports both packed (ray-indexed) and unpacked layouts, handling feature accumulation and weight-only accumulation [S2, S3, S4, unofficial].

## How it works
**Inputs:**
- `weights`: Tensor of shape $[M, 1]$ or $[M, C]$ containing per-sample weights (e.g., opacity or density).
- `values`: Optional Tensor of shape $[M, C]$ containing per-sample features (e.g., color).
- `ray_indices`: Tensor of shape $[M]$ mapping each sample to a specific ray index.
- `num_rays`: Integer, total number of rays.
- `C`: Number of feature channels (default 1 for weight-only).

**Outputs:**
- `accumulated_values`: Tensor of shape $[num\_rays, C]$ containing the final accumulated result per ray.

**Data Structures:**
- Output buffer initialized to zeros: $O \in \mathbb{R}^{num\_rays \times C}$.

**Steps:**

1. **Check Layout:** Determine if samples are packed (associated with `ray_indices`) or unpacked (sequential per ray).

2. **Packed Samples (Scatter-Add):**
   - If `ray_indices` and `num_rays` are provided:
     - Initialize output buffer:
       $$ O = \text{zeros}((num\_rays, C), \text{dtype}=float32) $$
     - If `values` is provided:
       - Compute weighted features:
         $$ W_{val} = weights[:, \text{None}] \times values \quad \text{shape: } [M, C] $$
       - Perform scatter-add:
         $$ O = O.\text{at}[ray\_indices].\text{add}(W_{val}) $$
     - If `values` is `None` (weight-only accumulation):
       - Compute weight contribution:
         $$ W_{only} = weights[:, \text{None}] \quad \text{shape: } [M, 1] $$
       - Perform scatter-add into buffer of shape $[num\_rays, 1]$:
         $$ O = O.\text{at}[ray\_indices].\text{add}(W_{only}) $$
     - Alternatively, use optimized library routines such as `nerfacc.accumulate_along_rays(weights, ray_indices, values, num_rays)` [S1, S2, unofficial].

3. **Unpacked Samples (Sequential Sum):**
   - If samples are ordered by ray (no `ray_indices` needed):
     - Sum weights along the sample dimension (typically `dim=-2`):
       $$ O = \text{sum}(weights, \text{dim}=-2) $$
     - If values are present, sum weighted values similarly:
       $$ O = \text{sum}(weights[:, \text{None}] \times values, \text{dim}=-2) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def scatter_add_accumulation(weights, values, ray_indices, num_rays, C=1):
    """
    Accumulate weighted values along rays using scatter-add.
    
    Args:
        weights: Tensor [M, 1] or [M, C], sample weights.
        values: Tensor [M, C] or None, sample features.
        ray_indices: Tensor [M], ray index for each sample.
        num_rays: int, total number of rays.
        C: int, number of feature channels.
        
    Returns:
        Tensor [num_rays, C], accumulated results.
    """
    # Initialize output buffer with zeros
    output = torch.zeros((num_rays, C), dtype=weights.dtype, device=weights.device)
    
    if values is not None:
        # Compute weighted features: weights[:, None] * values
        # Ensure weights has shape [M, 1] for broadcasting
        if weights.dim() == 1:
            w_expanded = weights.unsqueeze(1)
        else:
            w_expanded = weights
            
        weighted_features = w_expanded * values  # Shape: [M, C]
        
        # Scatter-add operation
        # output[ray_indices] += weighted_features
        # Using advanced indexing for scatter-add
        output.index_add_(0, ray_indices, weighted_features)
    else:
        # Weight-only accumulation
        if weights.dim() == 1:
            w_expanded = weights.unsqueeze(1)
        else:
            w_expanded = weights
            
        # If C=1, ensure shape is [M, 1]
        if C == 1 and w_expanded.dim() == 2 and w_expanded.shape[1] != 1:
            w_expanded = w_expanded[:, :1]
            
        output.index_add_(0, ray_indices, w_expanded)
        
    return output

def sequential_accumulation(weights, values):
    """
    Accumulate weighted values along rays when samples are ordered.
    
    Args:
        weights: Tensor [num_rays, num_samples] or [num_rays, num_samples, C]
        values: Tensor [num_rays, num_samples, C] or None
        
    Returns:
        Tensor [num_rays, C]
    """
    if values is not None:
        # Weighted sum along sample dimension (dim=1)
        # weights shape: [num_rays, num_samples, 1] or [num_rays, num_samples]
        # values shape: [num_rays, num_samples, C]
        if weights.dim() == 2:
            w_expanded = weights.unsqueeze(2)
        else:
            w_expanded = weights
            
        weighted_values = w_expanded * values
        return torch.sum(weighted_values, dim=1)
    else:
        # Sum weights along sample dimension
        if weights.dim() == 2:
            return torch.sum(weights, dim=1)
        else:
            return torch.sum(weights, dim=1)
```

**Edge Cases and Pitfalls:**
- **Precision:** Use `float32` for accumulation to balance precision and performance. `float16` may cause overflow or underflow in large accumulations [S3, unofficial].
- **Boundary Conditions:** Ensure `ray_indices` are within $[0, num\_rays - 1]$. Out-of-bounds indices cause undefined behavior or errors [S3, unofficial].
- **Degenerate Input:** If `weights` is empty, return zero-initialized output [S3, unofficial].
- **Atomicity:** In GPU implementations, scatter-add must be atomic to handle concurrent writes to the same ray index [S3, unofficial].

## Subtopics

### Concentric Rectangular Band Accumulation
A multipass approach where pixel shader summation is performed by accumulating results from concentric rectangular bands emanating from the light source into successive outer bands. The accumulation formula is $L(s, \theta, \phi) = L_{i-1}(s, \theta, \phi) + L_i(s, \theta, \phi)$, where $L$ represents accumulated light and $i$ is the band index [S5, book]. This method is described as the minimal required sampling and computation limit but may not be optimal for current hardware design compared to single-pass execution [S5, book].

## Operating characteristics
- **Time Complexity:** $O(M)$ for scatter-add, where $M$ is the number of samples. Parallelizable on GPU via atomic operations [S3, unofficial].
- **Memory Complexity:** $O(num\_rays \times C)$ for the output buffer [S3, unofficial].
- **Performance:** Efficient for large-scale neural rendering where samples are generated in parallel batches [S2, unofficial].
- **Strengths:** Supports non-ray-order sample presentation; flexible for both packed and unpacked data [S2, S3, unofficial].
- **Limitations:** Requires atomic operations which may cause contention on GPU; not suitable for front-to-back compositing where transparency blending is required [S4, unofficial].
- **When to Use:** When samples are generated out of order or in parallel batches; when only weight or feature accumulation is needed without transparency blending [S2, S3, unofficial].
- **When Not to Use:** When front-to-back compositing with transparency is required; when samples are already sorted by ray [S4, unofficial].

## Background
Scatter-Add Accumulation is widely used in neural radiance fields (NeRF) and related neural rendering techniques to handle parallel sample generation. It is implemented in libraries such as `nerfacc` and `gsplat-mlx` [S1, S2, S3, unofficial]. The method is also referenced in volumetric light scattering post-processing as a multipass accumulation technique [S5, book].

## Sources
[S1] nerfstudio-project/nerfstudio, https://github.com/nerfstudio-project/nerfstudio, fetched 2026-10-26 (unofficial)
[S2] autonomousvision/sdfstudio, https://github.com/autonomousvision/sdfstudio, fetched 2026-10-26 (unofficial)
[S3] RobotFlow-Labs/gsplat-mlx, https://github.com/RobotFlow-Labs/gsplat-mlx, fetched 2026-10-27 (unofficial)
[S4] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S5] Volumetric Light Scattering as a Post-Process, https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-13-volumetric-light-scattering-post-process, fetched 2026-10-27 (book)
