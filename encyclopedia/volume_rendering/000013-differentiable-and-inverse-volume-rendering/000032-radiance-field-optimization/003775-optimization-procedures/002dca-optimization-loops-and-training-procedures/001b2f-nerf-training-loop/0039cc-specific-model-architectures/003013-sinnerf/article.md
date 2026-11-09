# SinNeRF

Also called: Sinusoidal NeRF

## Overview
SinNeRF addresses the problem of training Neural Radiance Fields (NeRF) from a single input image, where the lack of multi-view constraints typically leads to degenerate solutions and poor novel view synthesis. It sets itself apart from standard NeRF by incorporating a depth map prior and adversarial/perceptual losses to constrain unobserved views, distinguishing it from sibling methods like [AdaNeRF](6195-adanerf.md) which focuses on adaptive sampling, or [ReconFusion](9112-nerdi.md) which uses diffusion priors.

## How it works
SinNeRF builds on the standard NeRF radiance field representation [Parent](8585-specific-model-architectures.md). The method optimizes the radiance field $f_\theta(\mathbf{x}, \mathbf{d})$ using a single reference image and a depth map of that reference view.

**Inputs and Outputs:**
*   **Input:** A single reference RGB image $\mathbf{I}_{ref}$ and its corresponding depth map $\mathbf{D}_{ref}$.
*   **Output:** A trained radiance field capable of synthesizing novel views.

**Data Structures:**
The architecture follows the original NeRF paper [S2]. The network takes positional encoding of coordinates and view direction as input. No specific bit-packing or custom data structures are defined beyond standard tensor operations for image patches.

**Training Procedure:**
1.  **Patch Sampling:** Randomly sample two patches of rays: one from the reference view and one from a randomly sampled unseen view (synthetic or interpolated).
    *   Patch sizes are dataset-specific: $64 \times 64$ for NeRF synthetic (Blender), $84 \times 63$ for LLFF, and $70 \times 56$ for DTU [S2].
    *   A stride schedule controls sampling density, starting at 6 and reducing by 2 every 10k iterations [S2].
2.  **Rendering:** Render the sampled patches to generate synthetic RGB images and depth maps.
3.  **Preprocessing:** Resize rendered patches to $224 \times 224$ resolution to match the input requirements of the DINO-ViT perceptual network [S2].
4.  **Loss Computation:**
    *   **Adversarial Loss:** A discriminator evaluates the realism of the rendered patches against real image patches.
    *   **Perceptual Loss:** Features extracted from the DINO-ViT network are compared between the rendered patches and the reference image to enforce semantic consistency [S1, unofficial][S2].
    *   **Depth Constraint:** The depth map of the reference view is used to constrain the unobserved views via the adversarial and perceptual losses [S1, unofficial].
5.  **Optimization:** Update network parameters using the RAdam optimizer [S2].

**Parameters:**
*   **Optimizer:** RAdam [S2].
*   **Learning Rate:** Initial $1e-3$ for the MLP; discriminator LR is 20% of the MLP's LR [S2].
*   **LR Decay:** Halved every 10k iterations [S2].
*   **Loss Weights:** $\lambda_1, \lambda_2, \lambda_3$ initialized to 8, 0.1, and 0 respectively. $\lambda_2$ decreases linearly to 0, while $\lambda_3$ increases linearly to 0.1 during training [S2].
*   **Hardware:** NVIDIA RTX A6000 GPU [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_sinnerf(reference_image, reference_depth, num_iterations=100000):
    # Initialize NeRF MLP and Discriminator
    mlp = init_nerf_mlp()
    discriminator = init_discriminator()
    dino_vit = init_dino_vit() # Pre-trained
    
    optimizer = RAdam(mlp.parameters(), lr=1e-3)
    disc_optimizer = RAdam(discriminator.parameters(), lr=1e-3 * 0.2)
    
    lambda1 = 8.0
    lambda2 = 0.1
    lambda3 = 0.0
    stride = 6
    
    for iteration in range(num_iterations):
        # Update loss weights linearly
        progress = iteration / num_iterations
        lambda2 = 0.1 * (1 - progress)
        lambda3 = 0.1 * progress
        
        # Update stride schedule
        if iteration % 10000 == 0 and iteration > 0:
            stride = max(1, stride - 2)
            
        # Sample patches
        # Patch sizes: 64x64 (Blender), 84x63 (LLFF), 70x56 (DTU)
        ref_patch = sample_patch(reference_image, stride=stride)
        unseen_patch = sample_unseen_view(mlp, stride=stride)
        
        # Resize for DINO-ViT
        ref_patch_resized = resize(ref_patch, 224, 224)
        unseen_patch_resized = resize(unseen_patch, 224, 224)
        
        # Compute losses
        # Adversarial loss
        disc_loss = compute_adversarial_loss(discriminator, ref_patch, unseen_patch)
        
        # Perceptual loss using DINO-ViT
        perceptual_loss = compute_perceptual_loss(dino_vit, ref_patch_resized, unseen_patch_resized)
        
        # Depth constraint loss (using reference depth to constrain unseen)
        depth_loss = compute_depth_constraint_loss(reference_depth, unseen_patch)
        
        # Total loss
        total_loss = lambda1 * disc_loss + lambda2 * perceptual_loss + lambda3 * depth_loss
        
        # Backpropagate
        optimizer.zero_grad()
        disc_optimizer.zero_grad()
        
        total_loss.backward()
        optimizer.step()
        disc_optimizer.step()
        
        # Decay learning rate every 10k iterations
        if iteration % 10000 == 0 and iteration > 0:
            adjust_lr(optimizer, factor=0.5)
            adjust_lr(disc_optimizer, factor=0.5)
```

## Subtopics

### SinNeRF Implementation Configuration
This variant refers to the specific hyperparameter and hardware setup described in the original paper. It is characterized by the use of RAdam optimizer, specific patch sizes for different datasets (Blender, LLFF, DTU), and the integration of DINO-ViT for perceptual loss. It is chosen when reproducible results matching the original ECCV 2022 publication are required.

## Operating characteristics
*   **Time Complexity:** Training takes several hours per scene [S2].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Evaluated on NeRF synthetic, LLFF, and DTU datasets using PSNR, SSIM, and LPIPS metrics [S2].
*   **Strengths:** Effective at generating plausible novel views from a single image by leveraging depth priors and adversarial training.
*   **Limitations:** Relies on the quality of the input depth map; performance may degrade if the depth prior is inaccurate.
*   **When to use:** Single-image 3D reconstruction tasks where multi-view inputs are unavailable.
*   **When not to use:** When multi-view images are available, standard NeRF or other multi-view methods are more robust.

## Background
SinNeRF was introduced in the paper "SinNeRF: Training Neural Radiance Fields on Complex Scenes from a Single Image" published at ECCV 2022 [S2]. It is part of the broader effort to regularize NeRF training for sparse input scenarios, following works like RegNeRF [S1, unofficial].

## Sources
[S1] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S2] SinNeRF: Training Neural Radiance Fields on Complex Scenes from a Single Image, https://doi.org/10.1007/978-3-031-20047-2_42, fetched 2026-10-26
