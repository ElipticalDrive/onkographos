# Segment Coalescing Compositing

Also called: Coalesced Ray Marching, Segment-based Volume Rendering, Volume Rendering with Segment Coalescing

## Overview
Segment Coalescing Compositing is a method for aggregating per-sample contributions (RGB, alpha, and depth) into final pixel values during direct volume ray casting. It solves the problem of efficiently mapping scattered ray samples to pixel-space outputs by using indexed reduction operations. It differs from standard sequential accumulation by explicitly grouping samples by `ray_id` and reducing them via sum, allowing for flexible handling of variable-length rays and background blending.

## How it works
The method takes pre-computed ray samples (density, alpha, RGB, and accumulated distance) and produces final image colors, alpha weights, and depth. It relies on a "segment coalescing" operation, which is effectively a gather-and-sum reduction indexed by ray identifier.

### Inputs and Outputs
**Inputs:**
- `rgb`: Tensor of shape `[M, 3]` containing RGB values for $M$ samples.
- `weights`: Tensor of shape `[M]` containing alpha weights for $M$ samples.
- `ray_id`: Tensor of shape `[M]` containing integer ray identifiers for each sample.
- `t`: Tensor of shape `[M]` containing accumulated distance (transmittance denominator) for each sample.
- `alphainv_last`: Tensor of shape `[N]` containing the inverse alpha of the last segment for each of $N$ rays.
- `render_kwargs`: Dictionary containing configuration flags (`rand_bkgd`, `bg`, `render_depth`).

**Outputs:**
- `rgb_marched`: Tensor of shape `[N, 3]` containing final RGB color per ray.
- `wsum_mid`: Tensor of shape `[N]` containing sum of weights for middle segments.
- `depth`: Tensor of shape `[N]` (optional) containing normalized depth.
- `ret_dict`: Dictionary containing all intermediate and final values.

### Data Structures
The implementation uses PyTorch tensors. The core operation is `torch.zeros` initialized to zeros, followed by indexed addition.
- `weights.unsqueeze(-1) * rgb`: Element-wise multiplication broadcasting weights to `[M, 1]` to match `rgb` shape `[M, 3]`.
- Reduction: Summation over the dimension corresponding to samples, indexed by `ray_id`.

### Steps
1. **Compute Weighted RGB:** Calculate the contribution of each sample to its ray's color.
   $$ C_{sample} = w_i \cdot \mathbf{c}_i $$
   where $w_i$ is the weight and $\mathbf{c}_i$ is the RGB color of sample $i$.
2. **Coalesce RGB:** Sum the weighted RGB contributions for all samples belonging to the same ray.
   $$ \mathbf{C}_{ray} = \sum_{i \in \text{ray}} w_i \cdot \mathbf{c}_i $$
   This is implemented by initializing a zero tensor of shape `[N, 3]` and adding `weights.unsqueeze(-1) * rgb` at indices specified by `ray_id`.
3. **Add Background:** Blend the background color into the marched RGB.
   - If `rand_bkgd` is true and training:
     $$ \mathbf{C}_{final} = \mathbf{C}_{ray} + (1 - \alpha_{last}) \cdot \mathbf{r} $$
     where $\mathbf{r}$ is a random tensor of shape `[N, 3]` with the same device/dtype as `rgb_marched`.
   - Otherwise:
     $$ \mathbf{C}_{final} = \mathbf{C}_{ray} + (1 - \alpha_{last}) \cdot \mathbf{B} $$
     where $\mathbf{B}$ is the specified background color from `render_kwargs['bg']`.
4. **Compute Middle Weight Sum:** Calculate the sum of weights for all segments except the last one (which is accounted for in the background term).
   - Create a boolean mask `inner_mask` identifying samples that are not the last segment of their ray.
   - Initialize a zero tensor of shape `[N]`.
   - Add `weights[inner_mask]` to the tensor at indices `ray_id[inner_mask]`.
5. **Compute Normalized Depth Parameter $s$:** For each sample, compute the normalized depth parameter from the accumulated distance $t$.
   $$ s_i = 1 - \frac{1}{1 + t_i} $$
   This maps $t \in [0, \infty)$ to $s \in [0, 1)$.
6. **Compute Depth (Optional):** If `render_depth` is enabled, compute the depth value for each ray.
   - Initialize a zero tensor of shape `[N]`.
   - Add `weights * s` to the tensor at indices `ray_id`.
   - This step is performed within `torch.no_grad()` to avoid storing gradients for depth.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def segment_coalescing_compositing(
    rgb: torch.Tensor,      # [M, 3]
    weights: torch.Tensor,  # [M]
    ray_id: torch.Tensor,   # [M]
    t: torch.Tensor,        # [M]
    alphainv_last: torch.Tensor, # [N]
    render_kwargs: dict,
    inner_mask: torch.Tensor # [M], boolean mask for non-last segments
) -> dict:
    N = alphainv_last.shape[0]
    
    # 1. Compute weighted RGB contributions
    # weights.unsqueeze(-1) makes it [M, 1], broadcasts with rgb [M, 3]
    weighted_rgb = weights.unsqueeze(-1) * rgb
    
    # 2. Coalesce RGB by ray_id
    # Initialize [N, 3] with zeros
    rgb_marched = torch.zeros(N, 3, dtype=rgb.dtype, device=rgb.device)
    # Scatter add: for each sample i, add weighted_rgb[i] to rgb_marched[ray_id[i]]
    rgb_marched.index_add_(0, ray_id, weighted_rgb)
    
    # 3. Add background
    if render_kwargs.get('rand_bkgd', False):
        # Random background noise
        bkgd = torch.rand_like(rgb_marched)
    else:
        bkgd = render_kwargs['bg']
    
    # alphainv_last is (1 - alpha_last), shape [N]
    # Expand to [N, 1] for broadcasting
    rgb_marched = rgb_marched + alphainv_last.unsqueeze(-1) * bkgd
    
    # 4. Compute sum of weights for middle segments
    wsum_mid = torch.zeros(N, dtype=weights.dtype, device=weights.device)
    if inner_mask.any():
        wsum_mid.index_add_(0, ray_id[inner_mask], weights[inner_mask])
        
    # 5. Compute normalized depth parameter s
    # s = 1 - 1/(1+t)
    s = 1.0 - 1.0 / (1.0 + t)
    
    # 6. Compute depth if enabled
    depth = None
    if render_kwargs.get('render_depth', False):
        with torch.no_grad():
            depth = torch.zeros(N, dtype=weights.dtype, device=weights.device)
            depth.index_add_(0, ray_id, weights * s)
            
    ret_dict = {
        'alphainv_last': alphainv_last,
        'weights': weights,
        'wsum_mid': wsum_mid,
        'rgb_marched': rgb_marched,
        'raw_density': None, # Placeholder if not provided
        'raw_alpha': None,   # Placeholder if not provided
        'raw_rgb': rgb,
        'ray_id': ray_id,
        'step_id': None,     # Placeholder if not provided
        'n_max': None,       # Placeholder if not provided
        't': t,
        's': s,
        'density': None,     # Placeholder if not provided
        'ray_pts': None,     # Placeholder if not provided
    }
    
    if depth is not None:
        ret_dict['depth'] = depth
        
    return ret_dict
```

### Parameters
- `rand_bkgd`: Boolean flag. If true, uses random noise for background blending during training.
- `render_depth`: Boolean flag. If true, computes and returns the depth tensor.
- `bg`: Background color tensor, used if `rand_bkgd` is false.

### Edge Cases and Pitfalls
- **Precision:** The calculation of $s$ involves division by $1+t$. If $t$ is very large, $s$ approaches 1. If $t$ is 0, $s$ is 0. Ensure no division by zero occurs (though $1+t \ge 1$).
- **Boundary Conditions:** The `inner_mask` must correctly identify the last segment of each ray. If the last segment is incorrectly included in the weight sum, the total alpha will exceed 1 or the background blending will be incorrect.
- **Memory:** The `index_add_` operation requires contiguous memory for the output tensor. Initializing with `torch.zeros` ensures this.

## Operating characteristics
- **Time Complexity:** $O(M)$ for the element-wise operations and $O(M)$ for the scatter-add operations, where $M$ is the total number of samples. The complexity is linear in the number of samples.
- **Memory Complexity:** $O(N)$ for the output tensors, where $N$ is the number of rays. Intermediate tensors are $O(M)$.
- **Strengths:** Efficiently handles variable-length rays. The use of indexed reduction allows for parallel computation on GPUs.
- **Limitations:** Requires careful handling of the `inner_mask` to correctly separate middle segments from the last segment. The method assumes that `ray_id` is contiguous and sorted or that the scatter-add operation handles unsorted indices correctly (which `index_add_` does).

## Background
This method is derived from the implementation details of the `SegmentAnythingin3D` repository [S1, unofficial]. It is used in the context of training 3D Gaussian Splatting or similar neural rendering frameworks where efficient compositing of ray samples is required. The specific formulation of normalized depth $s$ and the background blending strategy are tailored for training stability and loss computation.

## Sources
[S1] SegmentAnythingin3D, https://github.com/Jumpat/SegmentAnythingin3D, fetched 2026-10-26 (unofficial)
