# Hierarchical NeRF Rendering

Also called: Coarse-Fine Rendering, Hierarchical Rendering Pipeline, Two-Pass NeRF

## Overview
Hierarchical NeRF Rendering is a two-pass volume rendering pipeline that first estimates scene density with a coarse uniform or stratified pass, then uses the resulting sample weights to importance-sample a fine pass [S1, unofficial][S2, unofficial]. It distinguishes itself from uniform ray marching by adaptively concentrating samples in high-density regions identified by the coarse network, thereby reducing computational waste in empty space while maintaining high-frequency detail [S1, unofficial][S2, unofficial].

## How it works
**Inputs:** Ray origins $\mathbf{o}$, ray directions $\mathbf{d}$, near/far clipping planes $t_{near}, t_{far}$, number of coarse samples $N_c$, number of fine samples $N_f$, perturbation factor $\epsilon$, and chunk size for memory management [S2, unofficial].
**Outputs:** Final RGB color $\mathbf{C}(\mathbf{r})$, accumulated opacity $\alpha(\mathbf{r})$, depth, and auxiliary coarse RGB for loss calculation [S1, unofficial].

**Data Structures:**
*   **Rays:** Tensors of shape $(N_{rays}, 3)$ for origins and directions.
*   **Coarse Samples:** $t_{c} \in \mathbb{R}^{N_{rays} \times N_c}$, $\mathbf{pts}_{c} \in \mathbb{R}^{N_{rays} \times N_c \times 3}$, $\mathbf{rgb}_{c} \in \mathbb{R}^{N_{rays} \times N_c \times 3}$, $\sigma_{c} \in \mathbb{R}^{N_{rays} \times N_c \times 1}$ [S2, unofficial].
*   **Fine Samples:** $t_{f} \in \mathbb{R}^{N_{rays} \times N_f}$, $\mathbf{pts}_{f} \in \mathbb{R}^{N_{rays} \times N_f \times 3}$, $\mathbf{rgb}_{f} \in \mathbb{R}^{N_{rays} \times N_f \times 3}$, $\sigma_{f} \in \mathbb{R}^{N_{rays} \times N_f \times 1}$ [S2, unofficial].
*   **Merged Samples:** $t_{all} \in \mathbb{R}^{N_{rays} \times (N_c + N_f)}$ [S1, unofficial].

**Parameters:**
*   $N_c$: Number of coarse samples. Default: 64 [S1, unofficial][S2, unofficial].
*   $N_f$: Number of fine samples. Default: 128 [S1, unofficial][S2, unofficial].
*   $\epsilon$: Perturbation for stratified sampling. Default: 1.0 (on) [S2, unofficial].
*   $chunk$: Batch size for model inference to avoid Out-Of-Memory (OOM). Default: 1024 points [S1, unofficial] or $1024 \times 32$ [S2, unofficial].
*   $netchunk$: Max points per network forward pass. Default: $1024 \times 64$ [S2, unofficial].

**Steps:**

1.  **Coarse Sampling:** Generate $N_c$ samples uniformly along the ray between $t_{near}$ and $t_{far}$. If $\epsilon > 0$, apply stratified sampling by adding noise to the sample positions [S1, unofficial][S2, unofficial].
2.  **Coarse Evaluation:** Evaluate the coarse neural network on $\mathbf{pts}_{c}$ in chunks of size $netchunk$ to obtain color $\mathbf{rgb}_{c}$ and density $\sigma_{c}$ [S1, unofficial][S2, unofficial].
3.  **Coarse Volume Rendering:** Compute weights $w_{c}$ and accumulated color $\mathbf{C}_c$, depth $D_c$, and opacity $\alpha_c$ using standard volume rendering integration [S1, unofficial][S2, unofficial].
4.  **Fine Sampling (Importance Sampling):** Use the coarse weights $w_{c}$ as a probability distribution to sample $N_f$ additional points using inverse CDF sampling [S1, unofficial][S2, unofficial].
5.  **Fine Evaluation:** Evaluate the fine neural network on the fine points $\mathbf{pts}_{f}$ in chunks to obtain $\mathbf{rgb}_{f}$ and $\sigma_{f}$ [S1, unofficial][S2, unofficial].
6.  **Merge and Sort:** Concatenate coarse and fine depth values $t_{c}$ and $t_{f}$, then sort them in ascending order to form $t_{all}$ [S1, unofficial]. Compute corresponding merged points $\mathbf{pts}_{all}$ using $\mathbf{o}$ and $\mathbf{d}$ [S1, unofficial].
7.  **Fine Volume Rendering:** Evaluate the fine network on $\mathbf{pts}_{all}$ and render the final image using the merged samples [S1, unofficial]. The coarse RGB $\mathbf{C}_c$ is attached to the output for auxiliary loss calculation [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_rays(model_coarse, model_fine, rays_o, rays_d, near, far, N_c, N_f, perturb, chunk):
    # 1. Coarse Sampling
    t_c = stratified_sample(rays_o, rays_d, near, far, N_c, perturb)
    pts_c = rays_o[:, None, :] + rays_d[:, None, :] * t_c[:, :, None]
    
    # 2. Coarse Evaluation
    rgb_c, sigma_c = evaluate_chunks(model_coarse, pts_c, rays_d, chunk)
    
    # 3. Coarse Volume Rendering
    coarse_out = volume_render(rgb_c, sigma_c, t_c, rays_d, far, white_bg=True)
    weights_c = coarse_out["weights"]
    
    # 4. Fine Sampling
    t_f = importance_sample(t_c, weights_c, N_f, perturb)
    pts_f = rays_o[:, None, :] + rays_d[:, None, :] * t_f[:, :, None]
    
    # 5. Fine Evaluation
    rgb_f, sigma_f = evaluate_chunks(model_fine, pts_f, rays_d, chunk)
    
    # 6. Merge and Sort
    t_all = sort(concatenate([t_c, t_f], axis=1), axis=1)
    pts_all = rays_o[:, None, :] + rays_d[:, None, :] * t_all[:, :, None]
    
    # Re-evaluate fine model on all merged points
    rgb_all, sigma_all = evaluate_chunks(model_fine, pts_all, rays_d, chunk)
    
    # 7. Fine Volume Rendering
    final_out = volume_render(rgb_all, sigma_all, t_all, rays_d, far, white_bg=True)
    final_out["rgb_coarse"] = coarse_out["rgb_map"]
    return final_out

def stratified_sample(rays_o, rays_d, near, far, N, perturb):
    t_vals = linspace(0.0, 1.0, N + 1)
    t_vals = near + (far - near) * t_vals
    if perturb > 0:
        mids = 0.5 * (t_vals[:, 1:] + t_vals[:, :-1])
        upper = concatenate([mids, t_vals[:, -1:]], axis=1)
        lower = concatenate([t_vals[:, :1], mids], axis=1)
        rand_vals = random_uniform(t_vals.shape) * perturb
        t_vals = lower + (upper - lower) * rand_vals
    return t_vals

def volume_render(rgb, sigma, t_vals, rays_d, far, white_bg):
    deltas = concatenate([t_vals[:, 1:] - t_vals[:, :-1],
                          (far - t_vals[:, -1:]).reshape(-1, 1)], axis=1)
    alpha = 1.0 - exp(-sigma * deltas)
    # Transmittance T_i = product_{j<i} (1 - alpha_j)
    T = ones_like(alpha[:, :1])
    T = concatenate([T, cumprod(1.0 - alpha, axis=1)[:, :-1]], axis=1)
    weights = alpha * T
    rgb_map = sum(weights[:, :, None] * rgb, axis=1)
    depth = sum(weights[:, :, None] * t_vals, axis=1)
    acc = sum(weights, axis=1)
    if white_bg:
        rgb_map = rgb_map + (1.0 - acc[:, None]) * 1.0
    return {"rgb_map": rgb_map, "depth": depth, "acc": acc, "weights": weights}

def importance_sample(t_c, weights, N_f, perturb):
    cumsum = cumsum(weights, axis=1)
    cumsum = cumsum / cumsum[:, -1:]
    u = random_uniform((t_c.shape[0], N_f))
    if perturb > 0:
        u = u * (1.0 - 1e-5) + 1e-5
    indices = searchsorted(cumsum, u, side='right')
    indices = clamp(indices, 0, t_c.shape[1] - 1)
    t_f = gather(t_c, 1, indices)
    return t_f

def evaluate_chunks(model_fn, pts, dirs, chunk):
    N = pts.shape[0]
    rgb_list = []
    sigma_list = []
    for i in range(0, N, chunk):
        pts_chunk = pts[i:i+chunk]
        dirs_chunk = dirs[i:i+chunk]
        rgb, sigma = model_fn(pts_chunk, dirs_chunk)
        rgb_list.append(rgb)
        sigma_list.append(sigma)
    return concatenate(rgb_list, axis=0), concatenate(sigma_list, axis=0)

def linspace(start, stop, num):
    return [start + i * (stop - start) / (num - 1) for i in range(num)]

def random_uniform(shape):
    import random
    return [[random.random() for _ in range(shape[1])] for _ in range(shape[0])]

def concatenate(arrays, axis):
    if axis == 1:
        return [row1 + row2 for row1, row2 in zip(arrays[0], arrays[1])]
    return arrays

def sort(arr, axis):
    return sorted(arr, key=lambda x: x[axis])

def cumsum(arr, axis):
    if axis == 1:
        return [[sum(arr[i][:j+1]) for j in range(len(arr[i]))] for i in range(len(arr))]
    return arr

def searchsorted(arr, values, side):
    indices = []
    for row in arr:
        idx = []
        for v in values:
            if side == 'right':
                idx.append(next((i for i, x in enumerate(row) if x > v), len(row)-1))
            else:
                idx.append(next((i for i, x in enumerate(row) if x >= v), len(row)-1))
        indices.append(idx)
    return indices

def clamp(arr, min_val, max_val):
    return [[max(min_val, min(max_val, x)) for x in row] for row in arr]

def gather(arr, axis, indices):
    if axis == 1:
        return [[row[i] for i in idx] for row, idx in zip(arr, indices)]
    return arr

def ones_like(arr):
    return [[1.0 for _ in row] for row in arr]

def exp(arr):
    import math
    return [[math.exp(x) for x in row] for row in arr]

def sum(arr, axis):
    if axis == 1:
        return [sum(row) for row in arr]
    return arr
```

**Edge Cases and Pitfalls:**
*   **Degenerate Input:** If $N_c=0$, importance sampling fails. Ensure $N_c \ge 1$ [S1, unofficial][S2, unofficial].
*   **Memory:** Large $N_c$ and $N_f$ combined with high-resolution rays can exceed GPU memory. Use `chunk` size to limit simultaneous evaluations [S1, unofficial][S2, unofficial].
*   **Precision:** When computing transmittance, use a small epsilon (e.g., `1e-10`) to avoid `log(0)` or `exp(-inf)` issues in dense regions [G].
*   **Boundaries:** Ensure $\delta$ for the last sample uses $t_{far}$, not a non-existent next sample [G].

## Subtopics

### VF-NeRF Rendering Pipeline
Uses a different acceleration strategy focused on view frustum culling and hierarchical grid structures rather than simple coarse-fine importance sampling. Choose when scene complexity is high and view-dependent caching is beneficial. [VF-NeRF Rendering Pipeline](3302-vf-nerf-rendering-pipeline.md)

## Operating characteristics
*   **Time Complexity:** $O(N_{rays} \cdot (N_c + N_f))$ per ray. The coarse pass is $O(N_{rays} \cdot N_c)$ and the fine pass is $O(N_{rays} \cdot (N_c + N_f))$ due to merging and re-evaluation [G].
*   **Memory Complexity:** $O(N_{rays} \cdot (N_c + N_f))$ for storing merged samples and intermediate tensors [G].
*   **Measured Performance:** With $N_c=64, N_f=128$, it provides a significant quality improvement over uniform sampling with $N=128$ while being computationally cheaper than uniform sampling with $N=256$ [S1, unofficial][S2, unofficial].
*   **Strengths:** Efficiently allocates samples to high-density regions; reduces noise in empty space; simple to implement as a drop-in replacement for uniform sampling [G].
*   **Limitations:** Requires two network evaluations (coarse and fine) or a single network evaluated twice; sorting merged samples adds overhead; may still undersample if the coarse pass misses thin structures [G].
*   **When to Use:** Standard NeRF training and rendering where quality is prioritized over raw speed [G].
*   **When Not to Use:** Real-time applications requiring strict frame budgets without coarse-fine trade-off tuning; scenes with extremely thin structures that the coarse pass might miss entirely [G].

## Background
Not given in the sources.

## Sources
[S1] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S2] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
