# 3D Gaussian Splatting Optimization Parameters

Also called: 25 Things Every Principal Must Know, 3DGS Training Hyperparameters, Densification and Opacity Reset Settings, Principal Knowledge Points

## Overview
This method defines the hyperparameters and procedural rules for optimizing a set of 3D Gaussians to reconstruct a scene from images [S1, unofficial]. It sets apart from static rendering by introducing adaptive density control (clone, split, prune, opacity reset) and periodic opacity resets, which are critical for converging to a high-fidelity radiance field [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:** Initial set of $N$ 3D Gaussians, training images with known camera poses, and optimization hyperparameters [S1, unofficial].
**Outputs:** Optimized set of Gaussians with updated parameters $(\boldsymbol{\mu}, q, s, \alpha, \mathrm{SH})$ [S1, unofficial].

### Data Structures
Each Gaussian $g_i$ stores center $\boldsymbol{\mu}_i \in \mathbb{R}^3$, orientation $q_i \in \mathbb{R}^4$ (normalized quaternion), scale $s_i \in \mathbb{R}^3$, opacity $\alpha_i \in [0, 1]$, and spherical harmonics coefficients $\mathrm{SH}_i$ [S1, unofficial]. The covariance matrix is reconstructed as $\Sigma_i = R_i \, \mathrm{diag}(s_i)^2 \, R_i^\top$, where $R_i$ is the rotation matrix derived from $q_i$ [S1, unofficial].

### Optimization Steps
The training process alternates between rendering, loss computation, and density control [S1, unofficial].
1. **Initialization**: Place initial Gaussians using Structure-from-Motion (SfM) points and set initial scales based on nearest-neighbor distances [G]. Initialize opacities to a small value [G].
2. **Rendering**: Project each Gaussian to 2D using EWA splatting, which linearizes the perspective projection at $\boldsymbol{\mu}_c$ to obtain $\Sigma_{2D}$ [S1, unofficial]. Render the image via alpha compositing of 2D Gaussian footprints [S1, unofficial].
3. **Loss Computation**: Compute photometric loss (e.g., L1 + SSIM) between rendered and ground-truth images [G].
4. **Gradient Accumulation**: Accumulate gradients for $\boldsymbol{\mu}_i$ and other parameters, tracking the magnitude $\|\nabla_{\boldsymbol{\mu}_i} \mathcal{L}\|$ [G].
5. **Adaptive Density Control**: Performed periodically during training [S1, unofficial].
   - **Clone**: For Gaussians with high gradient magnitude and low opacity, clone the Gaussian [S1, unofficial].
   - **Split**: For Gaussians with high gradient magnitude and large scale, split the Gaussian into two [S1, unofficial].
   - **Prune**: Remove Gaussians with low opacity or small scale [S1, unofficial].
   - **Opacity Reset**: Reset opacities of all Gaussians to a small value to prevent saturation and allow further densification [S1, unofficial].

### Parameters
- **Densification Gradient Threshold ($\tau$)**: Approximately $2 \times 10^{-4}$ [S1, unofficial].
- **Opacity Reset Interval ($K_{\text{reset}}$)**: Every 3000 iterations [S1, unofficial].
- **Clone/Split/Prune Thresholds**: Specific scale and opacity thresholds for clone, split, and prune operations are not given in the sources. [S1, unofficial]

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_gaussians(gaussians, images, poses, num_iterations,
                       grad_threshold=2e-4,
                       opacity_reset_interval=3000,
                       clone_opacity_thresh=0.01,
                       prune_opacity_thresh=0.005):
    # Standard operations defined inline for self-containment
    def render(gs, ps):
        # EWA splatting: linearize perspective at mu_c to get Sigma_2D, then alpha composite footprints [S1, unofficial]
        return projected_composite(gs, ps)
    def compute_loss(rendered, target):
        # Photometric loss (L1 + SSIM) [G]
        return l1_ssim(rendered, target)
    def compute_gradients(loss, gs):
        # Backpropagate through rendering graph [G]
        return autograd(loss, gs)
    def update_parameters(gs, grads):
        # SGD/Adam step [G]
        for g in gs:
            g.mu -= lr * grads[g].mu
            g.alpha -= lr * grads[g].alpha
            g.scale -= lr * grads[g].scale
            g.q -= lr * grads[g].q
            g.SH -= lr * grads[g].SH
    def reset_opacities(gs):
        # Reset all opacities to a small value [S1, unofficial]
        for g in gs:
            g.alpha = 0.01
    def clone_gaussian(g):
        # Create copy at mu with perturbation, copy q, s, alpha, SH [G]
        new = copy(g)
        new.mu += uniform_noise(scale=0.01)
        gaussians.append(new)
    def split_gaussian(g):
        # Create two copies with reduced scale (e.g., /1.6) [G]
        new1 = copy(g); new2 = copy(g)
        new1.scale /= 1.6; new2.scale /= 1.6
        gaussians.extend([new1, new2])
    def prune_gaussian(g):
        # Mark for removal from list [G]
        g.to_remove = True

    for iter in range(num_iterations):
        rendered_image = render(gaussians, poses)
        loss = compute_loss(rendered_image, images[iter % len(images)])
        gradients = compute_gradients(loss, gaussians)
        update_parameters(gaussians, gradients)

        if iter % 100 == 0:
            to_clone, to_split, to_prune = [], [], []
            for g in gaussians:
                grad_mag = norm(gradients[g].mu)
                if grad_mag > grad_threshold and g.alpha < clone_opacity_thresh:
                    to_clone.append(g)
                if grad_mag > grad_threshold and g.scale > threshold_scale:
                    to_split.append(g)
                if g.alpha < prune_opacity_thresh:
                    to_prune.append(g)
            for g in to_clone: clone_gaussian(g)
            for g in to_split: split_gaussian(g)
            for g in to_prune: prune_gaussian(g)
            gaussians = [g for g in gaussians if not g.to_remove]

        if iter % opacity_reset_interval == 0:
            reset_opacities(gaussians)
```

### Edge Cases and Pitfalls
- **Precision**: Using log-space for scales prevents negative values and improves optimization stability [G].
- **Boundaries**: Low thresholds cause Gaussian explosion; monitor memory usage [G].
- **Degenerate Input**: Low-texture scenes yield small gradients, causing insufficient densification; adjust thresholds or add priors [G].
- **Popping Artifacts**: Arise from per-tile center-depth sort; use StopThePop, GOF, or exact pixel sort to fix them [S1, unofficial].

## Subtopics

### Mip-Splatting
Fixes scale-aliasing via 3D smoothing + 2D Mip filter. Use when high-frequency details cause aliasing artifacts. [S1, unofficial]

### 2DGS
Replaces 3D ellipsoids with oriented disks for better surface reconstruction. Use when geometry/mesh extraction is the primary goal. [S1, unofficial]

### Scaffold-GS / Octree-GS
Uses voxel anchors + tiny MLP for compression and far-view extrapolation. Use when model size is a constraint. [S1, unofficial]

### Hierarchical 3DGS
Provides Level-of-Detail (LoD) from city-scale to centimeter close-ups. Use for large-scale scenes. [S1, unofficial]

## Operating characteristics
- **Time Complexity**: Rendering is $O(N)$ where $N$ is the number of Gaussians. Densification adds sparse overhead [G].
- **Memory Complexity**: $O(N)$ per Gaussian. Vanilla 3DGS uses 1–3 M Gaussians for room-scale scenes, occupying 200–800 MB on disk [S1, unofficial].
- **Performance**: Real-time rendering is achievable with tile-based rasterization. Optimization speed scales with $N$ and iteration count [G].
- **Strengths**: High-quality novel view synthesis, real-time rendering, explicit representation [G].
- **Limitations**: Large model size, potential for popping artifacts due to per-tile center-depth sort (mitigated by StopThePop, GOF, exact pixel sort) [S1, unofficial].
- **When to Use**: Photorealistic novel view synthesis, room-scale scenes with plenty of views [S1, unofficial].
- **When Not to Use**: When surface mesh quality is critical (use 2DGS), or when model size is extremely constrained (use LightGaussian, SOG, EAGLES, RDO-Gaussian) [S1, unofficial].
- **Benchmarking**: Report compression, FPS, and memory alongside quality metrics; numbers without them are misleading [S1, unofficial].

## Background
3D Gaussian Splatting (3DGS) replaces NeRF's MLP+ray-march with explicit anisotropic Gaussians and tile rasterization [S1, unofficial]. The optimization parameters described here are based on the original implementation and its refinements. The method has become a standard for radiance-field optimization, with variants addressing compression, dynamic scenes, and geometry extraction [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
