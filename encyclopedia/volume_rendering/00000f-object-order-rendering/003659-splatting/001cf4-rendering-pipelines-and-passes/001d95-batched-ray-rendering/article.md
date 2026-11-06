# Batched Ray Rendering

Also called: Chunked Rendering, Memory-Efficient Rendering, NeRF Ray Rendering Pipeline

## Overview
Batched Ray Rendering is a memory-management pipeline for volume rendering that splits large ray batches into fixed-size chunks to prevent GPU Out-Of-Memory (OOM) errors [S3, unofficial]. It orchestrates the sequential execution of coarse and fine network passes per chunk, distinguishing itself from full pipeline descriptions by isolating the batching mechanism (`batchify_rays`) and per-chunk rendering logic (`render_rays`) [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray batch tensor of shape $(B, 11)$ where $B$ is the number of rays, composed of `rays_o` (3), `rays_d` (3), `near` (1), `far` (1), and `viewdirs` (3) [S3, unofficial].
*   **Output:** Rendered `rgb_map`, `disp_map`, `acc_map`, and optionally `raw` and coarse outputs, reshaped to match the original spatial dimensions $(H, W, \dots)$ [S3, unofficial].

**Data Structures:**
*   `rays_flat`: Tensor of shape $(N_{rays}, 11)$ containing concatenated ray parameters [S3, unofficial].
*   `all_ret`: Dictionary mapping output keys to lists of tensors, one list per chunk [S3, unofficial].

**Numbered Steps:**
1.  **Ray Tensor Construction:** Reshape `rays_o` and `rays_d` to $[-1, 3]$ and float type [S3, unofficial]. Broadcast scalar `near` and `far` values to match `rays_d[..., :1]` [S3, unofficial]. Concatenate `rays_o`, `rays_d`, `near`, and `far` along the last dimension [S3, unofficial]. If `use_viewdirs` is true, concatenate `viewdirs` to the ray tensor [S3, unofficial].
2.  **Batching Loop:** Initialize `all_ret` [S3, unofficial]. Iterate through `rays_flat` in chunks of size `chunk` [S3, unofficial]. For each chunk, call `render_rays` [S3, unofficial]. Append each returned tensor to the corresponding list in `all_ret` [S3, unofficial]. Concatenate all lists in `all_ret` along dimension 0 [S3, unofficial].
3.  **Per-Chunk Rendering:** Unpack `rays_o`, `rays_d`, `viewdirs`, `near`, and `far` [S3, unofficial]. Generate sample depths `z_vals` via linear or inverse depth interpolation [S3, unofficial]. Apply stratified sampling if `perturb > 0` [S3, unofficial]. Compute point positions `pts` and query the coarse network [S3, unofficial]. Process outputs via `raw2outputs` [S3, unofficial]. If `N_importance > 0`, perform hierarchical sampling: store coarse outputs, compute midpoints, sample fine depths, detach gradients, sort combined depths, recompute `pts`, query the fine network, and process outputs again [S3, unofficial].
4.  **Reshaping:** Reshape all output tensors in `all_ret` to match the original batch dimensions excluding the sample dimension [S3, unofficial].

**Formulas:**
*   Sample depths (linear): $z_{vals} = near \cdot (1. - t_{vals}) + far \cdot t_{vals}$ [S3, unofficial].
*   Sample depths (inverse depth): $z_{vals} = \frac{1.}{\frac{1.}{near} \cdot (1. - t_{vals}) + \frac{1.}{far} \cdot t_{vals}}$ [S3, unofficial].
*   Stratified sampling bounds: $mids = 0.5 \cdot (z_{vals}[:, 1:] + z_{vals}[:, :-1])$, $upper = \text{concat}(mids, z_{vals}[:, -1:])$, $lower = \text{concat}(z_{vals}[:, :1], mids)$ [S3, unofficial].
*   Stratified update: $z_{vals} = lower + (upper - lower) \cdot t_{rand}$ where $t_{rand} \sim \mathcal{U}[0, 1]$ [S3, unofficial].
*   Point positions: $pts = rays_o[..., None, :] + rays_d[..., None, :] \cdot z_{vals}[..., :, None]$ [S3, unofficial].
*   Hierarchical midpoints: $z_{vals\_mid} = 0.5 \cdot (z_{vals}[:, 1:] + z_{vals}[:, :-1])$ [S3, unofficial].

**Parameters:**
*   `chunk`: Integer, mini-batch size. Default: $1024 \times 32 = 32768$ [S3, unofficial].
*   `N_samples`: Integer, coarse pass samples per ray [S3, unofficial].
*   `N_importance`: Integer, fine pass additional samples. If 0, hierarchical sampling is skipped [S3, unofficial].
*   `lindisp`: Boolean. `False` for linear depth sampling, `True` for linear disparity sampling. Default: `False` [S3, unofficial].
*   `perturb`: Float, stratified sampling noise. Default: $0.0$ (overridden to $1.0$ during training) [S3, unofficial].
*   `network_fn`: Function querying the coarse network [S3, unofficial].
*   `network_fine`: Function querying the fine network. If `None`, `network_fn` is reused [S3, unofficial].
*   `raw_noise_std`: Float, standard deviation for noise added to raw sigma values [S3, unofficial].

**Edge Cases and Pitfalls:**
*   Stratified sampling implementation deviates from theoretical equal-bin sampling by using half-sized bins at the boundaries to fit $N$ samples into $[0, 1]$ [S3, unofficial].
*   Hierarchical sampling detaches `z_samples` from the computation graph to break gradient flow from the fine pass to the coarse pass [S3, unofficial].
*   Numerical errors (NaN or Inf) in outputs are checked and printed if DEBUG mode is enabled [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def batchify_rays(rays_flat, chunk=32768, **kwargs):
    """Renders rays in smaller mini-batches to avoid OOM [S3, unofficial]."""
    all_ret = {}
    num_rays = rays_flat.shape[0]
    for i in range(0, num_rays, chunk):
        ray_chunk = rays_flat[i : i + chunk]
        ret = render_rays(ray_chunk, **kwargs)
        for k in ret:
            if k not in all_ret:
                all_ret[k] = []
            all_ret[k].append(ret[k])
    for k in all_ret:
        all_ret[k] = torch.cat(all_ret[k], dim=0)
    return all_ret

def render_rays(ray_batch, network_fn, network_query_fn, N_samples, retraw=False,
                lindisp=False, perturb=0.0, N_importance=0, network_fine=None,
                white_bkgd=False, raw_noise_std=0.0, verbose=False):
    N_rays = ray_batch.shape[0]
    rays_o = ray_batch[:, 0:3]
    rays_d = ray_batch[:, 3:6]
    viewdirs = ray_batch[:, -3:] if ray_batch.shape[-1] > 8 else None
    bounds = ray_batch[..., 6:8].reshape([-1, 1, 2])
    near, far = bounds[..., 0], bounds[..., 1]
    
    t_vals = torch.linspace(0., 1., steps=N_samples)
    if not lindisp:
        z_vals = near * (1. - t_vals) + far * t_vals
    else:
        z_vals = 1. / (1./near * (1. - t_vals) + 1./far * t_vals)
    z_vals = z_vals.expand([N_rays, N_samples])
    
    if perturb > 0.:
        mids = 0.5 * (z_vals[:, 1:] + z_vals[:, :-1])
        upper = torch.cat([mids, z_vals[:, -1:]], -1)
        lower = torch.cat([z_vals[:, :1], mids], -1)
        t_rand = torch.rand(z_vals.shape)
        z_vals = lower + (upper - lower) * t_rand
        
    pts = rays_o[..., None, :] + rays_d[..., None, :] * z_vals[..., :, None]
    raw = network_query_fn(pts, viewdirs, network_fn)
    rgb_map, disp_map, acc_map, weights, depth_map = raw2outputs(raw, z_vals, rays_d, raw_noise_std, white_bkgd)
    
    if N_importance > 0:
        rgb_map_0, disp_map_0, acc_map_0 = rgb_map, disp_map, acc_map
        z_vals_mid = 0.5 * (z_vals[:, 1:] + z_vals[:, :-1])
        z_samples = sample_pdf(z_vals_mid, weights[:, 1:-1], N_importance, det=(perturb==0.))
        z_samples = z_samples.detach()
        z_vals, _ = torch.sort(torch.cat([z_vals, z_samples], -1), -1)
        pts = rays_o[..., None, :] + rays_d[..., None, :] * z_vals[..., :, None]
        run_fn = network_fn if network_fine is None else network_fine
        raw = network_query_fn(pts, viewdirs, run_fn)
        rgb_map, disp_map, acc_map, weights, depth_map = raw2outputs(raw, z_vals, rays_d, raw_noise_std, white_bkgd)
        
    ret = {'rgb_map': rgb_map, 'disp_map': disp_map, 'acc_map': acc_map}
    if retraw:
        ret['raw'] = raw
    if N_importance > 0:
        ret['rgb_map_0'] = rgb_map_0
        ret['disp_map_0'] = disp_map_0
        ret['acc_map_0'] = acc_map_0
    return ret

def raw2outputs(raw, z_vals, rays_d, raw_noise_std=0.0, white_bkgd=False, pytest=False):
    """Converts raw network output (rgb, sigma) to volume rendering outputs [G]."""
    rgb = raw[..., :3]
    sigma = raw[..., 3:4]
    noise = torch.randn(sigma.shape) * raw_noise_std if sigma.shape[-1] > 0 else 0.
    sigma = sigma + noise
    deltas = z_vals[:, 1:] - z_vals[:, :-1]
    delta_inf = 1e10 * torch.ones_like(deltas[:, :1])
    deltas = torch.cat([deltas, delta_inf], -1)
    alpha = 1.0 - torch.exp(-sigma * deltas)
    T = torch.cumprod(torch.cat([torch.ones_like(alpha[:, :1]), 1. - alpha + 1e-10], -1), -1)[:, :-1]
    weights = alpha * T
    rgb_map = torch.sum(weights[..., None] * rgb, -2)
    depth_map = torch.sum(weights * z_vals, -1)
    disp_map = 1. / torch.max(1e-10 * torch.ones_like(depth_map), depth_map / torch.sum(weights, -1))
    acc_map = torch.sum(weights, -1)
    return rgb_map, disp_map, acc_map, weights, depth_map

def sample_pdf(z_vals_mid, weights, N_importance, det=False, pytest=False):
    """Inverse CDF sampling for hierarchical sampling [G]."""
    weights = weights + 1e-5
    pdf = weights / torch.sum(weights, -1, keepdim=True)
    cdf = torch.cumsum(pdf, -1)
    cdf = torch.cat([torch.zeros_like(cdf[:, :1]), cdf], -1)
    if det:
        u = torch.linspace(0., 1., steps=N_importance)
        u = u.expand(list(cdf.shape[:-1]) + [N_importance])
    else:
        u = torch.rand(list(cdf.shape[:-1]) + [N_importance])
    u = u.contiguous()
    inds = torch.searchsorted(cdf, u, right=True)
    below = torch.max(torch.zeros_like(inds - 1), inds - 1)
    above = torch.min((cdf.shape[-1] - 1) * torch.ones_like(inds), inds)
    inds_g = torch.stack([below, above], -1)
    cdf_g = torch.gather(cdf, 1, inds_g)
    cdf_g = cdf_g.view(list(inds_g.shape[:-1]) + [1])
    bins = z_vals_mid
    w = torch.gather(weights, 1, inds_g)
    bins = bins.view(list(inds_g.shape[:-1]) + [1])
    w = w.view(list(inds_g.shape[:-1]) + [1])
    z_samples = below * (cdf_g - cdf[:, :1]) / (cdf[:, 1:] - cdf[:, :1] + 1e-5) + \
                above * (cdf[:, 1:] - cdf_g) / (cdf[:, 1:] - cdf[:, :1] + 1e-5)
    return z_samples
```

## Subtopics

### [Chunked Trilinear Volume Rendering](2557-chunked-trilinear-volume-rendering.md)
Processes rays in fixed-size memory chunks to manage GPU memory constraints during neural field evaluation. Differs from standard batched rendering by explicitly iterating over subsets of rays to prevent out-of-memory errors, and from Chunked Ray Rendering by relying on a neural field forward pass rather than explicit geometric traversal. Choose when rendering high-resolution neural radiance fields or large-volume trilinear grids where the full ray batch exceeds available VRAM.

### [Chunked Ray Rendering](2764-chunked-ray-rendering.md)
Memory-management strategy for object-order volume rendering that processes large batches of rays in smaller, fixed-size chunks to prevent Out-Of-Memory (OOM) errors on the GPU. Primarily used during validation and inference phases where full-resolution images must be generated without training-time optimizations like CUDA-accelerated raymarching. Differs from Chunked Trilinear Volume Rendering by chunking the *rays* rather than the *volume data*. Choose when generating full-resolution images during validation/inference on memory-constrained hardware.

## Operating characteristics
*   **Memory Complexity:** Effectively limited to $O(chunk \cdot N_{samples} \cdot D)$ where $D$ is the hidden dimension, preventing full-batch OOM [S3, unofficial].
*   **Time Complexity:** Linear with respect to the number of rays, $O(N_{rays})$, with constant overhead from chunk iteration and concatenation [S3, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Scales linearly with ray count; chunk size must be adjusted relative to available VRAM [S3, unofficial].
*   **Strengths:** Prevents OOM errors on large batches or high-resolution images; enables training on hardware with limited VRAM [S3, unofficial].
*   **Limitations:** Sequential processing of chunks may underutilize GPU parallelism compared to a single massive batch if VRAM allows; slight overhead from list concatenation [S3, unofficial].
*   **When to use:** When rendering high-resolution images or large batches that exceed GPU memory capacity [S3, unofficial].
*   **When not to use:** When VRAM is sufficient for the entire batch, as single-batch rendering is faster due to reduced overhead [S3, unofficial].

## Background
Batched Ray Rendering is a core component of the NeRF (Neural Radiance Fields) pipeline, introduced by Mildenhall et al. in 2020 [S3, unofficial]. It addresses the memory constraints of rendering millions of rays simultaneously by processing them in mini-batches [S3, unofficial].

## Sources
[S1] yashbhalgat/HashNeRF-pytorch, https://github.com/yashbhalgat/HashNeRF-pytorch, fetched 2026-10-26 (unofficial)
[S2] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S3] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
