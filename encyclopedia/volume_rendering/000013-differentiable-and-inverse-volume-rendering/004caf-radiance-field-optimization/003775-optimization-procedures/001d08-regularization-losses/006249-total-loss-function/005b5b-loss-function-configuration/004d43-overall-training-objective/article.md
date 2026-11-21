# Overall Training Objective

Also called: Total Loss Function, Composite Loss

## Overview
The Overall Training Objective defines the composite loss function used to optimize radiance fields, balancing pixel-wise fidelity, structural similarity, perceptual quality, and adversarial constraints. It differs from single-metric approaches by combining multiple perceptual and pixel-wise metrics to supervise optimization, addressing the problem of generating high-fidelity, visually consistent rendered images. It excludes explicit physical priors like visibility or normal consistency unless specified by a variant.

## How it works
The method minimizes a weighted sum of loss components during the training of a neural radiance field. The optimization is performed using the Adam optimizer.

**Inputs:**
- Rendered image patches or full images.
- Ground truth images.
- Text descriptions (positive and negative samples) for adversarial/perceptual guidance.
- Hyperparameters: $\lambda_g, \lambda_l, \lambda_p, \lambda_r$.

**Outputs:**
- Updated radiance field parameters.

**Data Structures:**
- Loss weights: $\lambda_g$ (adversarial), $\lambda_l$ (local contrastive), $\lambda_p$ (perceptual), $\lambda_r$ (reconstruction).
- Text embeddings: Positive text descriptions and negative text descriptions (e.g., from a collected dataset).

**Parameters:**
- $\lambda_g = 0.2$
- $\lambda_l = 0.1$
- $\lambda_p = 2.0$
- $\lambda_r = 0.1$
- Patch size: $1/10$ of the original input image dimensions.
- Negative samples: ~200 text descriptions manually collected (e.g., "Zombie", "Tolkien elf").

**Steps:**
1. **Reconstruction Stage:**
   - Sample 192 points per ray.
   - Compute the reconstruction loss ($\mathcal{L}_r$) between rendered and ground truth pixels.
   - Compute the perceptual loss ($\mathcal{L}_p$) using feature differences.
   - Compute the local contrastive loss ($\mathcal{L}_l$) using patches of size $1/10$ of the input.
   - Compute the adversarial loss ($\mathcal{L}_g$) using positive and negative text embeddings.
   - Total Loss: $\mathcal{L} = \lambda_g \mathcal{L}_g + \lambda_l \mathcal{L}_l + \lambda_p \mathcal{L}_p + \lambda_r \mathcal{L}_r$.
   - Update model parameters using Adam with learning rate $0.0005$ for 6 epochs.

2. **Stylization Stage:**
   - Use the same loss formulation.
   - Update model parameters using Adam with learning rate $0.001$ for 4 epochs.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Hyperparameters
lambda_g = 0.2
lambda_l = 0.1
lambda_p = 2.0
lambda_r = 0.1
lr_recon = 0.0005
lr_styl = 0.001
epochs_recon = 6
epochs_styl = 4
points_per_ray = 192
patch_ratio = 0.1

# Data
positive_texts = [...] # e.g., "Fauvism painting"
negative_texts = [...] # ~200 texts from Pinterest

def compute_total_loss(rendered_image, ground_truth, positive_emb, negative_emb, model):
    # Reconstruction loss (e.g., L1 or MSE)
    loss_r = reconstruction_loss(rendered_image, ground_truth)
    
    # Perceptual loss (e.g., LPIPS or feature distance)
    loss_p = perceptual_loss(rendered_image, ground_truth)
    
    # Local contrastive loss
    # Extract patches of size 1/10 of original image
    patch_size = (ground_truth.width * patch_ratio, ground_truth.height * patch_ratio)
    loss_l = local_contrastive_loss(rendered_image, ground_truth, patch_size)
    
    # Adversarial loss (conditional)
    # Uses text embeddings to guide style
    loss_g = adversarial_loss(rendered_image, positive_emb, negative_emb, model)
    
    total_loss = (lambda_g * loss_g + 
                  lambda_l * loss_l + 
                  lambda_p * loss_p + 
                  lambda_r * loss_r)
    return total_loss

def train_stage(model, data_loader, stage):
    optimizer = Adam(model.parameters(), lr=lr_recon if stage == 'recon' else lr_styl)
    epochs = epochs_recon if stage == 'recon' else epochs_styl
    
    for epoch in range(epochs):
        for batch in data_loader:
            rendered = model.render(batch.cameras, batch.points)
            loss = compute_total_loss(rendered, batch.gt_image, 
                                      batch.positive_text_emb, 
                                      batch.negative_text_emb, 
                                      model)
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
```

## Subtopics

### No Conditional Adversarial Objective
Deactivates the conditional adversarial loss ($\mathcal{L}_g$) and retrains the model. This results in incomplete renderings with clear artifacts, demonstrating the importance of the adversarial component for style transfer [S2].

## Operating characteristics
- **Time Complexity:** Depends on the number of epochs and points per ray. Reconstruction stage uses 6 epochs with 192 points/ray; stylization uses 4 epochs.
- **Memory Complexity:** Reduced by processing at the patch level (1/10 of original input) for local contrastive loss.
- **Strengths:** Balances local pixel fidelity with global structural integrity and perceptual quality.
- **Limitations:** Requires manual collection of negative text samples. Adversarial loss is critical for style transfer; removing it degrades quality significantly.
- **When to use:** When high-fidelity stylization of radiance fields is required using text guidance.
- **When not to use:** When computational resources are extremely limited or when style transfer is not needed (use standard reconstruction loss).

## Background
The method is based on NeRF-Art [S1], which introduces text-driven neural radiance fields stylization. It builds on VolSDF [S1] as the base NeRF model. The approach addresses the limitations of single-metric losses in generating visually consistent stylized images. The "No conditional adversarial objective" variant is discussed in Pix2NeRF [S2] to highlight the role of adversarial training in conditional GANs for NeRF translation.

## Sources
[S1] NeRF-Art: Text-Driven Neural Radiance Fields Stylization, https://doi.org/10.1109/tvcg.2023.3283400, fetched 2026-10-26
[S2] Pix2NeRF: Unsupervised Conditional $\pi$-GAN for Single Image to Neural Radiance Fields Translation, https://doi.org/10.1109/cvpr52688.2022.00395, fetched 2026-10-29
