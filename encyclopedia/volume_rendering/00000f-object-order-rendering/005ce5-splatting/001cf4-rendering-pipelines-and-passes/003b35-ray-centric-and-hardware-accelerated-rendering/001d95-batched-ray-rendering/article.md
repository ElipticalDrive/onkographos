# Batched Ray Rendering

Also called: Chunked Rendering, Memory-Efficient Rendering, NeRF Ray Rendering Pipeline

## Overview
Batched Ray Rendering is a memory-management strategy for object-order volume rendering that processes rays in fixed-size chunks to avoid Out-Of-Memory (OOM) errors [S3, unofficial]. It solves the problem where the number of rays in an image exceeds the capacity of the accelerator's VRAM or when the neural network model is too large for a full-batch forward pass [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origins $\mathbf{rays\_o}$, ray directions $\mathbf{rays\_d}$, near/far bounds, optional view directions $\mathbf{viewdirs}$, and rendering parameters ($N_{samples}$, $N_{importance}$, etc.) [S3, unofficial].
*   **Output:** Rendered color map $\mathbf{rgb\_map}$, disparity map $\mathbf{disp\_map}$, accumulated transmittance $\mathbf{acc\_map}$, and optionally coarse outputs and raw network outputs [S3, unofficial].

**Data Structures:**
*   **Ray Tensor:** A 2D tensor of shape $(B, 11)$ where $B$ is the batch size. The last dimension concatenates: $\mathbf{rays\_o}$ (3 floats), $\mathbf{rays\_d}$ (3 floats), $near$ (1 float), $far$ (1 float), and $\mathbf{viewdirs}$ (3 floats, optional) [S3, unofficial].
*   **Accumulation Dictionary (`all_ret`):** Maps output keys (e.g., `rgb_map`, `rgb0`) to lists of tensors, one list entry per chunk [S3, unofficial].

**Numbered Steps:**
1.  **Preparation:** Reshape $\mathbf{rays\_o}$ and $\mathbf{rays\_d}$ to $[-1, 3]$ and float type [S3, unofficial]. Broadcast scalar $near$ and $far$ values to match the shape of $\mathbf{rays\_d}[..., :1]$ [S3, unofficial]. Concatenate $\mathbf{rays\_o}$, $\mathbf{rays\_d}$, $near$, and $far$ along the last dimension to form the base ray tensor [S3, unofficial]. If `use_viewdirs` is true, concatenate $\mathbf{viewdirs}$ to the ray tensor [S3, unofficial].
2.  **Batching Loop (`batchify_rays`):** Initialize an empty dictionary `all_ret` [S3, unofficial]. Iterate through the ray tensor in chunks of size `chunk` [S1, S3, unofficial]. For each chunk, call `render_rays` [S3, unofficial]. For each key in the returned dictionary, append the tensor to the corresponding list in `all_ret` [S3, unofficial]. After the loop, concatenate all lists in `all_ret` along dimension 0 to form the final output tensors [S3, unofficial].
3.  **Rendering a Chunk (`render_rays`):** Unpack $\mathbf{rays\_o}$, $\mathbf{rays\_d}$, $\mathbf{viewdirs}$, $near$, and $far$ from the ray batch [S3, unofficial]. Generate $t_{vals}$ as a linear space from 0. to 1. with $N_{samples}$ steps [S3, unofficial]. Compute $z_{vals}$ (sample depths): if `lindisp` is false, $z_{vals} = near \times (1. - t_{vals}) + far \times t_{vals}$ [S3, unofficial]; if true, $z_{vals} = 1. / (1./near \times (1. - t_{vals}) + 1./far \times t_{vals})$ [S3, unofficial]. Expand $z_{vals}$ to shape $[N_{rays}, N_{samples}]$ [S3, unofficial]. If `perturb > 0.`, perform stratified sampling: calculate midpoints $mids = 0.5 \times (z_{vals}[..., 1:] + z_{vals}[...,:-1])$ [S3, unofficial], upper bounds $upper = \text{concat}([mids, z_{vals}[...,-1:]])$ [S3, unofficial], lower bounds $lower = \text{concat}([z_{vals}[...,:1], mids])$ [S3, unofficial], generate random noise $t_{rand}$ with shape $[N_{rays}, N_{samples}]$ [S3, unofficial], and update $z_{vals} = lower + (upper - lower) \times t_{rand}$ [S3, unofficial]. Compute point positions $\mathbf{pts} = \mathbf{rays\_o} + \mathbf{rays\_d} \times z_{vals}$ [S3, unofficial]. Query the network using `network_query_fn(pts, viewdirs, network_fn)` to get raw output [S3, unofficial]. Process raw output using `raw2outputs` to get $\mathbf{rgb\_map}$, $\mathbf{disp\_map}$, $\mathbf{acc\_map}$, weights, and depth map [S3, unofficial]. **Hierarchical Sampling (if $N_{importance} > 0$):** Store coarse outputs with `_0` suffix [S3, unofficial]. Calculate midpoints of coarse samples $z_{vals\_mid} = 0.5 \times (z_{vals}[..., 1:] + z_{vals}[...,:-1])$ [S3, unofficial]. Sample fine depths $z_{samples}$ using `sample_pdf` on $z_{vals\_mid}$ and weights $weights[..., 1:-1]$ [S3, unofficial]. Detach $z_{samples}$ from the computation graph to break gradient flow [S3, unofficial]. Concatenate coarse $z_{vals}$ and fine $z_{samples}$, then sort them to form the final $z_{vals}$ [S3, unofficial]. Recompute $\mathbf{pts}$ using the new sorted $z_{vals}$ [S3, unofficial]. Query the fine network (or coarse if `network_fine` is None) [S3, unofficial]. Process the fine raw output using `raw2outputs` [S3, unofficial]. Return results including coarse outputs if $N_{importance} > 0$ and raw output if `retraw` is true [S3, unofficial].
4.  **Finalization:** Reshape all output tensors in `all_ret` to match the original batch dimensions (excluding the last dimension which corresponds to the sample dimension) [S3, unofficial]. Extract $\mathbf{rgb\_map}$, $\mathbf{disp\_map}$, and $\mathbf{acc\_map}$ into a list, and return them along with the remaining dictionary items [S3, unofficial].

**Parameters:**
*   `chunk`: Size of mini-batches. Default is $1024 \times 32 = 32768$ [S3, unofficial].
*   `N_samples`: Number of samples per ray for the coarse pass [S3, unofficial].
*   `N_importance`: Number of additional samples for the fine pass. If 0, hierarchical sampling is skipped [S3, unofficial].
*   `lindisp`: Boolean. If false, samples are linear in depth. If true, samples are linear in disparity (inverse depth). Default is false [S3, unofficial].
*   `perturb`: Amount of noise added for stratified sampling. Default is 0. (often overridden to 1.0 during training) [S3, unofficial].
*   `raw_noise_std`: Standard deviation for noise added to raw sigma values [S3, unofficial].
*   `network_fn`: Coarse network function [S3, unofficial].
*   `network_fine`: Fine network function. If None, the coarse network is reused [S3, unofficial].

**Edge Cases and Pitfalls:**
*   Stratified sampling implementation deviates from theory: the first and last bins are half the size of others to fit $N$ samples into $[0,1]$, whereas theory assumes equal bin lengths [S3, unofficial].
*   Numerical errors (NaN or Inf) in outputs are checked and printed if DEBUG mode is enabled [S3, unofficial].
*   Hierarchical sampling uses `z_samples.detach()` to break the gradient flow from the fine pass to the coarse pass, creating a static compute graph for the coarse-to-fine connection [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def batchify_rays(rays_flat, chunk=1024*32, **kwargs):
    """Render rays in smaller minibatches to avoid OOM."""
    all_ret = {}
    num_rays = rays_flat.shape[0]
    for i in range(0, num_rays, chunk):
        ray_batch = rays_flat[i:i+chunk]
        ret = render_rays(ray_batch, **kwargs)
        for k in ret:
            if k not in all_ret:
                all_ret[k] = []
            all_ret[k].append(ret[k])
    all_ret = {k: torch.cat(all_ret[k], 0) for k in all_ret}
    return all_ret

def render_rays(ray_batch, network_fn, network_query_fn, N_samples, retraw=False,
                lindisp=False, perturb=0., N_importance=0, network_fine=None,
                white_bkgd=False, raw_noise_std=0., verbose=False, pytest=False):
    N_rays = ray_batch.shape[0]
    rays_o = ray_batch[:, 0:3]
    rays_d = ray_batch[:, 3:6]
    viewdirs = ray_batch[:, -3:] if ray_batch.shape[-1] > 8 else None
    bounds = torch.reshape(ray_batch[..., 6:8], [-1, 1, 2])
    near, far = bounds[..., 0], bounds[..., 1]
    
    t_vals = torch.linspace(0., 1., steps=N_samples)
    if not lindisp:
        z_vals = near * (1. - t_vals) + far * t_vals
    else:
        z_vals = 1. / (1./near * (1. - t_vals) + 1./far * t_vals)
    z_vals = z_vals.expand([N_rays, N_samples])
    
    if perturb > 0.:
        mids = .5 * (z_vals[..., 1:] + z_vals[..., :-1])
        upper = torch.cat([mids, z_vals[..., -1:]], -1)
        lower = torch.cat([z_vals[..., :1], mids], -1)
        t_rand = torch.rand(z_vals.shape)
        if pytest:
            np.random.seed(0)
            t_rand = torch.Tensor(np.random.rand(*list(z_vals.shape)))
        z_vals = lower + (upper - lower) * t_rand
        
    pts = rays_o[..., None, :] + rays_d[..., None, :] * z_vals[..., :, None]
    raw = network_query_fn(pts, viewdirs, network_fn)
    rgb_map, disp_map, acc_map, weights, depth_map = raw2outputs(
        raw, z_vals, rays_d, raw_noise_std, white_bkgd, pytest=pytest
    )
    
    if N_importance > 0:
        rgb_map_0, disp_map_0, acc_map_0 = rgb_map, disp_map, acc_map
        z_vals_mid = .5 * (z_vals[..., 1:] + z_vals[..., :-1])
        z_samples = sample_pdf(z_vals_mid, weights[..., 1:-1], N_importance, det=(perturb==0.), pytest=pytest)
        z_samples = z_samples.detach()
        z_vals, _ = torch.sort(torch.cat([z_vals, z_samples], -1), -1)
        pts = rays_o[..., None, :] + rays_d[..., None, :] * z_vals[..., :, None]
        run_fn = network_fn if network_fine is None else network_fine
        raw = network_query_fn(pts, viewdirs, run_fn)
        rgb_map, disp_map, acc_map, weights, depth_map = raw2outputs(
            raw, z_vals, rays_d, raw_noise_std, white_bkgd, pytest=pytest
        )
        
    ret = {'rgb_map': rgb_map, 'disp_map': disp_map, 'acc_map': acc_map}
    if N_importance > 0:
        ret['rgb0'] = rgb_map_0
        ret['disp0'] = disp_map_0
        ret['acc0'] = acc_map_0
    if retraw:
        ret['raw'] = raw
    return ret

def raw2outputs(raw, z_vals, rays_d, raw_noise_std=0., white_bkgd=False, pytest=False):
    """Converts raw network output (RGB + sigma) to color, depth, and acc [G]."""
    rgb = raw[..., :3]
    sigma = raw[..., 3:]
    if raw_noise_std > 0.:
        sigma = sigma + torch.randn_like(sigma) * raw_noise_std
    delta_z = z_vals[..., 1:] - z_vals[..., :-1]
    delta_z = torch.cat([delta_z, torch.full_like(delta_z[..., :1], 1e10)], -1)
    alpha = 1. - torch.exp(-torch.relu(sigma) * delta_z)
    transmittance = torch.cumprod(1. - alpha + 1e-10, dim=-1)
    transmittance = torch.cat([torch.ones_like(transmittance[..., :1]), transmittance[..., :-1]], dim=-1)
    weights = alpha * transmittance
    rgb_map = torch.sum(weights[..., None] * rgb, -2)
    depth_map = torch.sum(weights * z_vals, -1)
    acc_map = torch.sum(weights, -1)
    disp_map = 1. / torch.max(1e-10 * torch.ones_like(depth_map), depth_map / acc_map)
    if white_bkgd:
        rgb_map = rgb_map + (1. - acc_map[..., None])
    return rgb_map, disp_map, acc_map, weights, depth_map

def sample_pdf(z_vals_mid, weights, N_importance, det=False, pytest=False):
    """Hierarchical sampling using inverse CDF [G]."""
    weights = weights + 1e-5
    weights = weights / torch.sum(weights, dim=-1, keepdim=True)
    cdf = torch.cumsum(weights, dim=-1)
    cdf = torch.cat([torch.zeros_like(cdf[..., :1]), cdf], dim=-1)
    if det:
        u = torch.linspace(0., 1., steps=N_importance)
        u = u.expand(list(cdf.shape[:-1]) + [N_importance])
    else:
        u = torch.rand(list(cdf.shape[:-1]) + [N_importance])
    indices = torch.searchsorted(cdf, u, side='right')
    indices = indices[..., :-1].clamp(0, cdf.shape[-1] - 2)
    lower = indices
    upper = indices + 1
    cdf_lower = cdf.gather(-1, lower.unsqueeze(-1)).squeeze(-1)
    cdf_upper = cdf.gather(-1, upper.unsqueeze(-1)).squeeze(-1)
    z_vals_mid = z_vals_mid.gather(-1, lower.unsqueeze(-1)).squeeze(-1)
    z_vals = z_vals_mid + (u - cdf_lower) / (cdf_upper - cdf_lower + 1e-5) * (z_vals_mid[..., 1:] - z_vals_mid[..., :-1])
    return z_vals
```

## Subtopics

### [Chunked Trilinear Volume Rendering](2557-chunked-trilinear-volume-rendering.md)
Chunked Trilinear Volume Rendering evaluates a neural field or trilinear volume by processing rays in fixed-size batches. It prevents memory overflow by iterating over subsets of rays rather than evaluating the full batch at once.

### [Chunked Ray Rendering](2764-chunked-ray-rendering.md)
Chunked Ray Rendering is a memory-management strategy for object-order volume rendering that processes rays in fixed-size batches (chunks) rather than all at once. It solves Out-Of-Memory (OOM) errors that occur when the number of rays in an image exceeds the accelerator's VRAM capacity or when the neural network model is too large for a full-batch forward pass. It sets itself apart from its parent, Batched Ray Rendering, by explicitly handling the sharding of these chunks across multiple hosts and devices in distributed settings, ensuring that partial results are correctly concatenated and reshaped into the final image.

### [Hybrid Splatting/Raycasting with Smooth Transition](14147-hybrid-splatting-raycasting-with-smooth-transition.md)
This method renders volumetric datasets as a collection of overlapping 3D detailed kernels, combining GPU-accelerated splatting for coarse structure with raycasting for local detail. It solves the trade-off between the speed of splatting and the accuracy of raycasting by using a hierarchical representation where each node contains a 3D texture. Unlike pure splatting, it enhances each footprint with raycasted details along the kernel's extent, and unlike pure raycasting, it batches these operations per fragment.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays $B$ and samples per ray $N_{samples}$, i.e., $O(B \cdot N_{samples})$ [S3, unofficial]. The batching introduces negligible overhead compared to the network query cost [S3, unofficial].
*   **Memory Complexity:** Dominated by the chunk size [S3, unofficial]. VRAM usage is proportional to `chunk` $\times$ $N_{samples}$, independent of the total image resolution [S3, unofficial]. This allows rendering high-resolution images on hardware with limited VRAM [S3, unofficial].
*   **Measured Performance:** Default chunk size is $1024 \times 32 = 32768$ rays [S3, unofficial].
*   **Strengths:** Prevents OOM errors; enables training and inference on large scenes or high-resolution images with limited GPU memory [S3, unofficial].
*   **Limitations:** Sequential processing of chunks may underutilize GPU parallelism if chunks are too small; does not inherently optimize ray traversal (relies on the underlying `render_rays` implementation) [S1, unofficial].
*   **When to use:** When rendering large images or using large neural networks that exceed VRAM capacity for a full-batch forward pass [S3, unofficial].
*   **When not to use:** When VRAM is sufficient for full-batch rendering, as full-batch may be faster due to reduced kernel launch overhead and better parallelism [S3, unofficial].

## Background
Batched Ray Rendering is a core component of the NeRF (Neural Radiance Fields) rendering pipeline [S3, unofficial]. It was introduced in the original NeRF paper to enable the training of neural networks on high-resolution images by managing memory usage during the forward pass [S3, unofficial]. The method is widely used in subsequent NeRF variants and neural rendering frameworks [S1, S2, unofficial].

## Sources
[S1] yashbhalgat/HashNeRF-pytorch, https://github.com/yashbhalgat/HashNeRF-pytorch, fetched 2026-10-26 (unofficial)
[S2] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S3] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
