# NeRF Volumetric Rendering (Coarse/Fine Sampling)

Also called: Vanilla NeRF, Standard NeRF, NeRF Two-Stage Sampling, Volume Rendering (NeRF), NeRF Coarse-to-Fine Rendering, Neural Radiance Fields rendering, coarse-fine volumetric rendering

## Overview

NeRF Volumetric Rendering (Coarse/Fine Sampling) is a differentiable volume rendering method that synthesizes novel views of 3D scenes by querying a neural radiance field (MLP) along camera rays and integrating the outputs using classical volume rendering techniques [S3, S14]. It solves the problem of insufficient sampling in high-opacity regions, which leads to aliasing and noise, by employing a two-stage sampling strategy: a coarse network estimates the density distribution to guide a fine network that concentrates samples where the volume contributes most to the final color [S1, unofficial][S2]. It sets itself apart from its siblings, such as [Iterative Ray Marching with Neural Volume Inference](1494-iterative-ray-marching-with-neural-volume-inference.md) which uses learned acceleration structures, and [DVGO Volume Rendering Forward Pass](2363-dvgo-volume-rendering-forward-pass.md) which uses voxel grids, by relying on a simple, differentiable stratified resampling of the coarse alpha weights without auxiliary acceleration structures [S1, unofficial][S2].

## How it works

**Inputs and Outputs:**
*   **Inputs:** A batch of rays defined by origins $\mathbf{o} \in \mathbb{R}^3$, directions $\mathbf{d} \in \mathbb{R}^3$, and near/far bounds $[t_n, t_f]$. Ground truth RGB images for loss computation.
*   **Outputs:** Rendered RGB color $\mathbf{C}$, accumulated opacity $\alpha$, and optionally depth $D$ for each ray.

**Data Structures:**
*   **Ray Batch:** Tensor of shape $[R, D]$, where $R$ is the number of rays. Columns $0:3$ are ray origins, $3:6$ are ray directions, $6:8$ are near/far bounds. If view directions are used, columns $-3:$ contain them [S8, unofficial].
*   **NeRF Model:** An MLP $F_\theta$ mapping 5D coordinates (3D position + viewing direction) to raw density $\sigma$ and color $\mathbf{c}$. Input shape $[N, 5]$, output shape $[N, 4]$ (RGB + $\sigma$) [S4, S9, unofficial].
*   **Sample Points:** Tensor `z_vals` of shape $[R, N]$ representing depth positions along rays [S9, unofficial].
*   **3D Points:** Tensor `pts` of shape $[R, N, 3]$ representing spatial coordinates [S6, unofficial].

**Parameters:**
*   $N_{coarse}$: Number of coarse samples per ray. Default: 64 [S8, S19, unofficial].
*   $N_{fine}$: Number of additional fine samples per ray. Default: 128 [S19, unofficial].
*   $N_{total} = N_{coarse} + N_{fine}$ [S19, unofficial].
*   `perturb`: Float controlling jitter. 0.0 for no jitter, 1.0 for full jitter. Default: 1.0 [S8, S19, unofficial].
*   `raw_noise_std`: Standard deviation of noise added to density predictions for regularization. Default: 0.0, Recommended: 1.0 [S9, S19, unofficial].
*   `lindisp`: Boolean. If True, sample linearly in inverse depth (disparity) rather than depth. Default: False [S8, S19, unofficial].
*   `white_bkgd`: Boolean. If True, composites result onto white background. Default: False [S9, S19, unofficial].
*   `chunk`: Number of rays processed in parallel to manage memory. Default: 32768 [S19, unofficial].

**Algorithm Steps:**

1.  **Ray Generation:** For each pixel, generate ray $r(t) = \mathbf{o} + t\mathbf{d}$ using camera pose and intrinsics [S2, S14]. Extract origins $\mathbf{o}$, directions $\mathbf{d}$, and bounds $t_n, t_f$ from the ray batch [S5, S8, unofficial].

2.  **Coarse Sampling:**
    *   Generate $N_{coarse}$ sample depths $t_{vals}$ linearly spaced between 0 and 1 [S7, unofficial].
    *   Map to physical depth $z_{vals}$:
        *   If `lindisp` is False: $z_{vals} = t_n (1 - t_{vals}) + t_f t_{vals}$ [S6, S7, unofficial].
        *   If `lindisp` is True: $z_{vals} = \frac{1}{\frac{1}{t_n}(1 - t_{vals}) + \frac{1}{t_f}t_{vals}}$ [S5, S7, unofficial].
    *   If `perturb` > 0, apply stratified sampling: compute midpoints $mids = 0.5(z_{vals}[:, 1:] + z_{vals}[:, :-1])$, define intervals $[z_{vals}[:, :1], mids]$ and $[mids, z_{vals}[:, -1:]]$, and jitter uniformly within these intervals [S6, S8, unofficial].
    *   Compute 3D points: $\mathbf{p}_i = \mathbf{o} + z_{vals,i} \mathbf{d}$ [S6, S8, unofficial].

3.  **Coarse Network Query:**
    *   Flatten points to $[-1, 3]$. If view directions are used, flatten directions to $[-1, 3]$ and concatenate with positions [S1, unofficial].
    *   Query coarse MLP $F_{\theta_{coarse}}$ to get raw output tensor of shape $[R \cdot N_{coarse}, 4]$ [S1, S9, unofficial].
    *   Reshape to $[R, N_{coarse}, 4]$. Split into raw color $\mathbf{c}_{raw}$ (first 3 channels) and raw density $\sigma_{raw}$ (4th channel) [S1, S9, unofficial].
    *   Apply activation functions: $\mathbf{c} = \text{sigmoid}(\mathbf{c}_{raw})$, $\sigma = \text{softplus}(\sigma_{raw})$ or $\sigma = \text{ReLU}(\sigma_{raw})$ [S1, S9, unofficial].
    *   Add Gaussian noise with standard deviation `raw_noise_std` to $\sigma$ for regularization [S9, S13, unofficial].

4.  **Coarse Rendering:**
    *   Compute segment distances: $\delta_i = z_{vals,i+1} - z_{vals,i}$ for $i=1 \dots N_{coarse}-1$. Append a large value (e.g., $10^{10}$) for the last segment to ensure ray termination [S9, unofficial][S11].
    *   Scale distances by ray direction norm if directions are not unit length: $\delta_i = \delta_i \|\mathbf{d}\|$ [S9, unofficial].
    *   Compute opacity: $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ [S6, S18, unofficial].
    *   Compute transmittance: $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$, with $T_1 = 1$ [S24][S22, unofficial].
    *   Compute weights: $w_i = T_i \alpha_i$ [S18, S20, unofficial].
    *   Compute coarse color: $\mathbf{C}_{coarse} = \sum_{i=1}^{N_{coarse}} w_i \mathbf{c}_i$ [S1, S18, unofficial].
    *   Compute accumulated opacity: $\alpha_{acc} = \sum_{i=1}^{N_{coarse}} w_i$ [S1, S18, unofficial].

5.  **Fine Sampling (Importance Resampling):**
    *   If $N_{fine} > 0$, use coarse weights $w_i$ (excluding the first sample if it has zero weight or is background) to define a probability density function (PDF) [S18, S23, unofficial].
    *   Draw $N_{fine}$ additional samples from this PDF using stratified sampling [S18, S23, unofficial]. This concentrates samples in high-opacity regions [S1, unofficial][S2].
    *   Concatenate coarse $z_{vals}$ and fine $z_{vals}$, then sort them to interleave fine samples among coarse ones [S18, unofficial].
    *   Detach fine samples from the computation graph to prevent gradient flow back to the coarse network during fine sampling [S18, unofficial].

6.  **Fine Network Query:**
    *   Query fine MLP $F_{\theta_{fine}}$ at the combined sample points. The fine network has the same architecture but independent weights [S18, S19, unofficial].
    *   Apply same activations (sigmoid for color, softplus/ReLU for density) and noise addition as coarse stage [S1, S9, unofficial].

7.  **Fine Rendering:**
    *   Recompute $\delta_i, \alpha_i, T_i, w_i$ for the combined sorted samples [S18, unofficial].
    *   Compute fine color: $\mathbf{C}_{fine} = \sum_{i=1}^{N_{total}} w_i \mathbf{c}_i$ [S13, S18, unofficial].
    *   Compute accumulated opacity: $\alpha_{acc} = \sum_{i=1}^{N_{total}} w_i$ [S18, unofficial].

8.  **Background Compositing:**
    *   If `white_bkgd` is True, add $(1 - \alpha_{acc})$ to the final RGB color to simulate a white background: $\mathbf{C}_{final} = \mathbf{C}_{fine} + (1 - \alpha_{acc}) \cdot 1.0$ [S9, S19, unofficial].

9.  **Loss Computation:**
    *   Compute photometric loss (MSE) between $\mathbf{C}_{coarse}$ and ground truth RGB [S13, unofficial].
    *   If fine samples are used, compute MSE between $\mathbf{C}_{fine}$ and ground truth RGB and add to loss [S13, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def render_rays(rays_o, rays_d, z_vals, network_fn, network_query_fn, 
                perturb=1.0, raw_noise_std=0.0, white_bkgd=False, 
                lindisp=False, netchunk=1024*64):
    """
    Render rays using a single network (coarse or fine).
    rays_o: [R, 3]
    rays_d: [R, 3]
    z_vals: [R, N]
    """
    # 1. Compute 3D points
    pts = rays_o[:, None, :] + rays_d[:, None, :] * z_vals[..., None]  # [R, N, 3]
    
    # 2. Query network
    # Flatten for batched query
    pts_flat = pts.reshape(-1, 3)
    # If view directions are used, they must be passed to network_query_fn
    # Assuming network_query_fn handles viewdirs if needed
    raw = network_query_fn(pts_flat, network_fn, netchunk)  # [R*N, 4]
    
    # 3. Reshape and split
    raw = raw.reshape(list(pts.shape[:-1]) + [4])  # [R, N, 4]
    rgb = raw[..., :3]  # [R, N, 3]
    sigma = raw[..., 3]  # [R, N]
    
    # 4. Activations
    rgb = torch.sigmoid(rgb)
    sigma = F.softplus(sigma)  # or ReLU
    
    # 5. Add noise to sigma for regularization
    if raw_noise_std > 0:
        sigma = sigma + torch.randn_like(sigma) * raw_noise_std
        
    # 6. Compute distances
    dists = z_vals[..., 1:] - z_vals[..., :-1]  # [R, N-1]
    dists = torch.cat([dists, torch.Tensor([1e10]).expand(dists[..., :1].shape).to(dists.device)], dim=-1)  # [R, N]
    dists = dists * torch.norm(rays_d[..., None, :], dim=-1)  # [R, N]
    
    # 7. Compute alpha and transmittance
    alpha = 1.0 - torch.exp(-sigma * dists)  # [R, N]
    transmittance = torch.cumprod(torch.exp(-sigma * dists), dim=-1)  # [R, N]
    transmittance = torch.cat([torch.ones_like(transmittance[..., :1]), transmittance[..., :-1]], dim=-1)  # [R, N]
    
    # 8. Compute weights
    weights = alpha * transmittance  # [R, N]
    
    # 9. Compute color
    rgb_final = torch.sum(weights[..., None] * rgb, dim=-2)  # [R, 3]
    
    # 10. Background
    if white_bkgd:
        acc = torch.sum(weights, dim=-1)  # [R]
        rgb_final = rgb_final + (1.0 - acc)[..., None]
        
    return rgb_final, weights, alpha, transmittance

def render_coarse_fine(rays_o, rays_d, near, far, 
                       n_samples_coarse=64, n_samples_fine=128,
                       perturb=1.0, raw_noise_std=0.0, white_bkgd=False,
                       lindisp=False, chunk=32768):
    """
    Two-stage rendering.
    """
    R = rays_o.shape[0]
    
    # --- Coarse Stage ---
    # Generate coarse z_vals
    t_vals = torch.linspace(0., 1., n_samples_coarse).to(rays_o.device)
    if not lindisp:
        z_vals_coarse = near * (1. - t_vals) + far * t_vals
    else:
        z_vals_coarse = 1. / (1. / near * (1. - t_vals) + 1. / far * t_vals)
        
    # Stratified sampling for coarse
    if perturb > 0.:
        mids = .5 * (z_vals_coarse[..., 1:] + z_vals_coarse[..., :-1])
        upper = torch.cat([mids, z_vals_coarse[..., -1:]], -1)
        lower = torch.cat([z_vals_coarse[..., :1], mids], -1)
        t_rand = torch.rand(z_vals_coarse.shape).to(rays_o.device)
        z_vals_coarse = lower + (upper - lower) * t_rand
        
    # Query coarse network
    # Assume coarse_model and coarse_query_fn are available
    rgb_coarse, weights_coarse, alpha_coarse, transmittance_coarse = render_rays(
        rays_o, rays_d, z_vals_coarse, coarse_model, coarse_query_fn,
        perturb=0.0, raw_noise_std=raw_noise_std, white_bkgd=False, lindisp=False
    )
    
    # --- Fine Stage ---
    if n_samples_fine > 0:
        # Resample based on coarse weights
        # Exclude first sample if it's background/empty? Usually use all weights
        # PDF is proportional to weights
        # Sample finer z-values using coarse weights
        # Implementation detail: use torch.multinomial or inverse CDF sampling
        # For simplicity, assume a helper function sample_finer_z_values exists
        # which draws n_samples_fine points from the distribution defined by weights_coarse
        z_vals_fine = sample_finer_z_values(weights_coarse, n_samples_fine, near, far, lindisp)
        
        # Concatenate and sort
        z_vals = torch.cat([z_vals_coarse, z_vals_fine], dim=-1)
        z_vals, _ = torch.sort(z_vals, dim=-1)
        
        # Detach fine samples to stop gradient flow to coarse
        z_vals = z_vals.detach()
        
        # Query fine network
        rgb_fine, weights_fine, alpha_fine, transmittance_fine = render_rays(
            rays_o, rays_d, z_vals, fine_model, fine_query_fn,
            perturb=0.0, raw_noise_std=raw_noise_std, white_bkgd=False, lindisp=False
        )
    else:
        rgb_fine = rgb_coarse
        weights_fine = weights_coarse
        
    return rgb_coarse, rgb_fine, weights_coarse, weights_fine
```

**Edge Cases and Pitfalls:**
*   **Numerical Stability:** Add a small epsilon ($\epsilon = 10^{-10}$) to transmittance calculations to prevent $\log(0)$ or division by zero [S12, S19, unofficial].
*   **Last Segment:** The last segment distance must be large (e.g., $10^{10}$) to ensure the ray terminates and transmittance goes to zero [S9, unofficial][S11].
*   **View Directions:** If `use_viewdirs` is False, the network input is only 3D position, reducing view-dependent effects [S19, unofficial].
*   **Memory:** Process rays in chunks (`chunk` size) to avoid OOM errors during network queries [S11][S19, unofficial].

## Subtopics

### [NeRF Importance Sampling](1565-nerf-importance-sampling.md)
NeRF Importance Sampling is a two-stage volume rendering strategy that refines the discrete sampling distribution along a ray by drawing additional points from a probability density function (PDF) derived from the coarse stage's opacity weights. It solves the problem of insufficient sampling in high-opacity regions, which leads to aliasing and noise, by concentrating fine samples where the volume contributes most to the final color. It sets itself apart from its siblings, such as [NerfAcc Rendering Pipeline](1619-nerfacc-rendering-pipeline.md) which uses learned acceleration structures, and [VF-NeRF](3138-vf-nerf.md) which uses view-dependent feature grids, by relying on a simple, differentiable stratified resampling of the coarse alpha weights without auxiliary networks or complex data structures.

### [NerfAcc Rendering Pipeline](1619-nerfacc-rendering-pipeline.md)
The NerfAcc Rendering Pipeline is a modular, high-performance volumetric rendering method implemented as a PyTorch library. It decouples the sampling strategy from the radiance field representation, supporting both Occupancy Grids and Proposal Networks to accelerate training and inference. Unlike standard uniform sampling, it uses interval-based ray representation to support conical frustums and integrates coarse/fine sampling with importance sampling to concentrate samples on surfaces while skipping empty space. It builds on [NeRF Volumetric Rendering (Coarse/Fine Sampling)](979-nerf-volumetric-rendering-coarse-fine-sampling.md) by providing a unified API for efficient transmittance estimation and differentiable rendering.

### [VF-NeRF](3138-vf-nerf.md)
VF-NeRF is a neural volume rendering method that represents indoor scenes using learned vector fields rather than standard scalar density fields, trained from multi-view posed images and their associated monocular depth maps. It distinguishes itself from standard NeRF by modeling scene geometry and appearance through a vector field representation integrated along rays, leveraging structural priors from depth supervision.

### Voxel-based Volume Rendering
A volume rendering technique that trains a voxel representation of a radiance field, where multiple voxels contribute to a single sample during rendering [S1, unofficial]. It differs from voxelization (sparse octree) in that voxel-based rendering uses dense or semi-dense voxels where multiple voxels contribute to a single sample, whereas voxelization creates a sparse octree containing voxels only where the model determines the space is solid [S1, unofficial]. It is used as a coarse model for two-tier sampling in NeRF-like architectures to determine opaque regions in space [S1, unofficial].

## Operating characteristics

*   **Time Complexity:** $O(R \cdot N \cdot D_{MLP})$, where $R$ is rays, $N$ is samples per ray, and $D_{MLP}$ is MLP depth/width. Two-stage sampling doubles the query cost but improves convergence.
*   **Memory Complexity:** $O(R \cdot N)$ for storing sample points, densities, and gradients. Chunking reduces peak memory usage.
*   **Measured Performance:** Training takes 10-20 hours on a single GPU for 50k-200k iterations [S7, unofficial]. Rendering an image takes <1 second to 30 seconds depending on resolution and chunk size [S7, unofficial].
*   **Scaling:** Scales linearly with number of rays and samples. Does not scale well to very high resolutions without chunking or acceleration structures.
*   **Strengths:** High-quality novel view synthesis, differentiable, simple implementation, view-dependent effects.
*   **Limitations:** Slow training and inference compared to grid-based methods, requires known camera poses, sensitive to sampling density.
*   **When to Use:** High-quality offline rendering, view synthesis with known poses.
*   **When Not to Use:** Real-time applications, scenes with unknown poses, large-scale outdoor scenes without acceleration.

## Background

Originated by Mildenhall et al. in 2020 with the paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" [S3]. It introduced the use of a continuous volumetric function family modeled by an MLP to map 3D coordinates and viewing directions to volume density and RGB color [S7, S13, unofficial]. The method was trained using gradient descent on a photometric loss between rendered and ground truth images [S3, S15]. It has become a foundational technique in neural rendering, inspiring numerous variants and acceleration methods [S4, unofficial][S21].

## Sources

[S1] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
[S2] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S3] NeRF: Representing scenes as neural radiance fields for view synthesis, http://www.matthewtancik.com/nerf, fetched 2026-10-27
[S4] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
[S5] GreatDrake/reparameterized-volume-sampling, https://github.com/GreatDrake/reparameterized-volume-sampling, fetched 2026-10-26 (unofficial)
[S6] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S7] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S8] yenchenlin/nerf-pytorch, https://github.com/yenchenlin/nerf-pytorch, fetched 2026-10-26 (unofficial)
[S9] yashbhalgat/HashNeRF-pytorch, https://github.com/yashbhalgat/HashNeRF-pytorch, fetched 2026-10-26 (unofficial)
[S10] The Adaption of Recent New Concepts in Neural Radiance Fields and Their Role for High-Fidelity Volume Reconstruction in Medical Images., https://doi.org/10.3390/s24185923, fetched 2026-10-26
[S11] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S12] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S13] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S14] 3D volumetric rendering with NeRF - Keras, https://keras.io/examples/vision/nerf/, fetched 2026-10-26
[S15] Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane, https://doi.org/10.1109/tmi.2024.3388559, fetched 2026-10-26
[S16] Fast Learning Radiance Fields by Shooting Much Fewer Rays, https://doi.org/10.1109/tip.2023.3267049, fetched 2026-10-26
[S17] Jrender: An efficient differentiable rendering library based on Jittor, https://www.sciencedirect.com/science/article/pii/S1524070323000322, fetched 2026-11-01
[S18] charan2598/NeRF-PyTorch, https://github.com/charan2598/NeRF-PyTorch, fetched 2026-10-27 (unofficial)
[S19] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S20] 3DGS-101/textbook/05-alpha-compositing.md at main - GitHub, https://github.com/Chamud/3DGS-101/blob/main/textbook/05-alpha-compositing.md, fetched 2026-10-28 (unofficial)
[S21] VoxNeRF: Bridging Voxel Representation and Neural Radiance Fields for ..., https://arxiv.org/abs/2311.05289, fetched 2026-10-29
[S22] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
[S23] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S24] NeRFMeshing: Distilling Neural Radiance Fields into Geometrically-Accurate 3D Meshes, https://doi.org/10.1109/3dv62453.2024.00093, fetched 2026-10-29
[S25] TöRF: Time-of-Flight Radiance Fields for Dynamic Scene View Synthesis, https://doi.org/10.48550/arxiv.2109.15271, fetched 2026-10-29 (unofficial)
