# BARF (Bundle-Adjusting Neural Radiance Fields)
Also called: Bundle-Adjusting NeRF
## Overview
BARF reconstructs a Neural Radiance Field (NeRF) and refines inaccurate camera poses given multi-view images with pose errors [S1, unofficial]. It differs from standard NeRF optimization by using a coarse-to-fine refinement strategy that gradually increases the high-frequency part of positional encoding [S1, unofficial].
## How it works
**Inputs and outputs:**
- Inputs: Multi-view images $\{I_i\}$, initial camera poses $\{P_i\}$ (with errors), and a NeRF model $\mathcal{F}_\theta$ [S1, unofficial].
- Outputs: Refined camera poses $\{\hat{P}_i\}$ and optimized NeRF parameters $\hat{\theta}$ [S1, unofficial].

**Data structures:**
- **NeRF Model $\mathcal{F}_\theta$**: A multi-layer perceptron (MLP) mapping 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to volume density $\sigma$ and color $\mathbf{c}$ [G].
- **Positional Encoding $\Gamma$**: Standard Fourier feature mapping used to embed $\mathbf{x}$ and $\mathbf{d}$ [G].
- **Optimization State**: Parameters $\theta$ for the MLP and pose parameters (rotation $\mathbf{R}$, translation $\mathbf{t}$) for each camera [G].

**Method:**
The core innovation is a coarse-to-fine refinement strategy that gradually introduces high-frequency information into the positional encoding [S1, unofficial]. This allows the optimizer to first converge on low-frequency structures (which are robust to pose errors) before refining high-frequency details (which require accurate poses) [G].

1.  **Initialization**: Initialize the NeRF parameters $\theta$ and camera poses $P_i$ using the provided (potentially inaccurate) initial poses [S1, unofficial].
2.  **Coarse-to-Fine Schedule**: Define a schedule for the number of frequency bands $L(t)$ used in the positional encoding $\Gamma$ at training step $t$ [S1, unofficial].
3.  **Optimization Loop**: For each training step $t$:
    a. Set the positional encoding frequency limit $L(t)$ based on the schedule [S1, unofficial].
    b. Sample rays through the scene using the current camera poses $P_i$ [G].
    c. Render colors $\hat{C}$ using the NeRF $\mathcal{F}_\theta$ with positional encoding $\Gamma_{L(t)}$ [G].
    d. Compute the loss $\mathcal{L}$ between rendered colors $\hat{C}$ and ground truth images $I_i$ (typically Mean Squared Error) [G].
    e. Perform gradient descent to update both NeRF parameters $\theta$ and camera pose parameters [G].
    f. Increment $t$ and update $L(t)$ if the schedule dictates [S1, unofficial].

**Formulas:**
- Positional Encoding with frequency limit $L$:
  $$ \Gamma_L(\mathbf{x}) = [\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x})] $$
- Rendering equation (standard volume rendering):
  $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i \left( 1 - e^{-\sigma_i \delta_i} \right) \mathbf{c}_i, \quad T_i = e^{-\sum_{j=1}^{i-1} \sigma_j \delta_j} $$
- Loss function:
  $$ \mathcal{L} = \frac{1}{N} \sum_{i=1}^N \| \hat{C}_i - I_i \|^2 $$

**Parameters:**
- Initial frequency bands: Not given in the sources. [G]
- Final frequency bands: Not given in the sources. [G]
- Learning rate: Not given in the sources. [G]
- Optimization algorithm: Not given in the sources. [G]

**Edge cases and pitfalls:**
- **Precision**: The gradual introduction of frequencies prevents the optimizer from being misled by high-frequency noise in the presence of pose errors [S1, unofficial].
- **Boundaries**: The schedule must be smooth enough to allow convergence at each frequency level [G].
- **Degenerate input**: If initial poses are extremely poor, even the coarse stages may fail to converge [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def barf_optimization(images, initial_poses, nerf_mlp, schedule_fn, num_steps):
    # Initialize learnable parameters
    theta = list(nerf_mlp.parameters())
    poses = [p.clone().requires_grad_(True) for p in initial_poses]
    optimizer = torch.optim.Adam(theta + poses, lr=1e-4) # [G]

    for t in range(num_steps):
        L_t = schedule_fn(t)
        optimizer.zero_grad()

        rendered_images = []
        for i, gt_img in enumerate(images):
            # [G] Standard ray sampling through camera frustum
            rays_o, rays_d = sample_rays_through_poses(poses[i], gt_img.shape)
            # Apply positional encoding with limited frequency bands
            encoded_rays = apply_positional_encoding(rays_o, rays_d, L=L_t)
            # [G] Standard differentiable volume rendering
            color = volume_render(nerf_mlp, encoded_rays)
            rendered_images.append(color)

        loss = compute_mse_loss(rendered_images, images)
        loss.backward()
        optimizer.step()

    return poses, theta

def apply_positional_encoding(rays_o, rays_d, L):
    # Standard Fourier features up to frequency band L
    encoded = []
    for i in range(L):
        freq = 2**i * torch.pi
        encoded.extend([torch.sin(freq * rays_o), torch.cos(freq * rays_o),
                        torch.sin(freq * rays_d), torch.cos(freq * rays_d)])
    return torch.cat(encoded, dim=-1)

def compute_mse_loss(rendered, ground_truth):
    return torch.mean((torch.stack(rendered) - torch.stack(ground_truth)) ** 2)
```
## Subtopics

### [Iterative Alignment Strategy](6957-iterative-alignment-strategy.md)
The Iterative Alignment Strategy is a post-processing and data-generation technique used to mitigate reconstruction artifacts in Neural Radiance Fields (NeRF) caused by inaccurate camera poses and non-rigid scene motion [G]. It addresses the "domain gap" where NeRF-rendered views are blurry due to misalignment, violating the sharp-pair assumption of standard optical flow estimators [G]. Unlike direct alignment methods that optimize a loss function during training, this strategy iteratively warps ground truth images using optical flow to create a better-aligned training set, leveraging the sharpening effect of improved alignment to refine flow estimates in subsequent rounds [G].

### [Global Pose and Scene Refinement](17192-global-pose-and-scene-refinement.md)
This method performs the final optimization step in the LU-NeRF pipeline to achieve high-fidelity novel view synthesis by jointly refining camera poses and the neural radiance field (NeRF) representation [G]. It solves the problem of residual errors in camera poses and scene geometry that remain after the initial synchronization stage [G]. Unlike the Iterative Alignment Strategy, which focuses on sequential alignment, this method leverages the global consistency provided by synchronization to apply a comprehensive bundle adjustment-style refinement [G]. It sets itself apart by using the BARF optimization procedure as its core engine, treating the synchronized poses as a sufficiently accurate initialization to converge to the ground truth [G].

## Operating characteristics
- **Time complexity**: Not given in the sources. [G]
- **Memory complexity**: Not given in the sources. [G]
- **Strengths**: Robust to initial pose errors by decoupling frequency optimization [S1, unofficial].
- **Limitations**: Requires a coarse-to-fine schedule [S1, unofficial].
- **When to use**: When camera poses are inaccurate or sparse [S1, unofficial].

## Background
BARF (Bundle-Adjusting Neural Radiance Fields) was introduced in ICCV 2021 [S1, unofficial]. It addresses the issue of NeRF's sensitivity to camera pose accuracy, which is a common problem in real-world scenarios where poses are estimated from SfM pipelines and may contain errors [S1, unofficial]. The method was developed to improve the robustness of NeRF reconstruction under pose uncertainty [S1, unofficial].

## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
