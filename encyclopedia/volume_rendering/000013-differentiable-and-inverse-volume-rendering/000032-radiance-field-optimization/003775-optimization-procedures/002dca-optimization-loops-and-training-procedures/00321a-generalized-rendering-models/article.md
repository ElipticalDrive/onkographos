# Generalized Rendering Models

Also called: Optimized Rendering Models for Clinical Data

## Overview
Generalized Rendering Models are deep neural architectures trained to optimize volume rendering parameters end-to-end, enabling consistent visual output across subjects with high inter-patient variations in clinical data [S1, unofficial]. Unlike manual transfer functions or simple lookup tables, these models learn complex visual feature mappings (such as colorization and view-dependent shading) directly from image-space data, bypassing the need for explicit intensity-based feature engineering [S1, unofficial].

## How it works
The method involves training specific neural network architectures (Lookup, VNet variants, DVRNet, RenderNet) to minimize the difference between rendered images and target images generated from clinical CT volumes [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** 3D volumetric data (e.g., abdominal angiographic CT volumes, resampled to isotropic resolution $192 \times 192 \times 192$) [S1, unofficial].
*   **Output:** 2D rendered images (resolution $200 \times 200$) for fixed viewing directions [S1, unofficial].
*   **Targets:** Ground truth images generated via a shader-based Differentiable Volume Rendering (DVR) implementation [S1, unofficial].

**Data Generation:**
Two distinct datasets are created from the volumetric data to test different rendering capabilities [S1, unofficial]:
1.  **Kidney Dataset:** Uses 3D label volumes (arteries and kidneys) to generate targets with distinct colors and boosted opacity for these structures [S1, unofficial].
2.  **Shading Dataset:** Does not use labels. Instead, it uses an ambient and diffuse shading model with secondary shadow rays. The light source is fixed at the top-left of the camera (view-dependent illumination) [S1, unofficial].

**Architectures:**
1.  **Lookup Table (Baseline):** A simple parameterized transfer function [S1, unofficial].
2.  **VNet Variants:** 3D convolutional encoders followed by Multi-Layer Perceptrons (MLPs) for transfer functions [S1, unofficial]. Variants include `VNet4-4`, `VNetL16-4` (latent color space), and `VNetL16-17` (latent color space with decoder) [S1, unofficial].
3.  **DVRNet:** A multiscale architecture shifting 3D upconvolutions to 2D upconvolutions in the decoder, allowing reasoning in image space [S1, unofficial].
4.  **RenderNet:** Transforms the volume to view space before applying 3D convolutions to better capture view-dependent effects [S1, unofficial].

**Training Procedure:**
1.  **Data Split:** Data is split by patient: 15 patients for training, 6 for validation, 6 for testing [S1, unofficial]. This results in 240 training images and 96 validation/test images per dataset [S1, unofficial].
2.  **Loss Function:** Sum of Structural Similarity Index (SSIM) and Mean Squared Error (MSE) [S1, unofficial].
3.  **Optimizer:** Adam optimizer with $\beta_1 = 0.9$, $\beta_2 = 0.99$ [S1, unofficial].
4.  **Annealing:** Step size annealing is applied with parameters $s_l = 0.1$, $s_h = 1.0$ for all models except RenderNet [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader

# Define the loss function: SSIM + MSE
def ssim_mse_loss(rendered, target):
    # Implementation of SSIM and MSE
    # Note: SSIM is used as part of the loss, not just an independent metric
    return ssim(rendered, target) + mse(rendered, target)

# Define the model architecture (e.g., DVRNet or VNet variant)
# The model takes 3D volume as input and outputs 2D image
model = DVRNet() 
# or model = VNetL16_17()
# or model = RenderNet()

# Optimizer setup
optimizer = optim.Adam(model.parameters(), lr=learning_rate, betas=(0.9, 0.99))

# Data loading
# Training set: 15 patients, 16 views each, 200x200 resolution
# Target generation: 
# - Kidney: Label-based color/opacity boosting
# - Shading: Ambient/diffuse shading, light top-left
train_loader = DataLoader(clinical_ct_dataset, batch_size=batch_size, shuffle=True)

# Training loop
for epoch in range(num_epochs):
    for volume_batch, target_batch in train_loader:
        optimizer.zero_grad()
        
        # Forward pass: Render volume to image
        # The rendering operator is differentiable
        rendered_batch = model(volume_batch)
        
        # Compute loss
        loss = ssim_mse_loss(rendered_batch, target_batch)
        
        # Backward pass
        loss.backward()
        
        # Update weights
        optimizer.step()
        
        # Step size annealing (if not RenderNet)
        # sl = 0.1, sh = 1.0
        adjust_learning_rate(optimizer, epoch, sl=0.1, sh=1.0)

# Evaluation
# Split: 6 patients for validation, 6 for testing
# Metrics: LPIPS, FID, SSIM
```

**Edge Cases and Pitfalls:**
*   **Intensity Ambiguity:** Lookup tables fail to discriminate structures (bone, vessel, kidney) purely by intensity, leading to incorrect colorization [S1, unofficial].
*   **View-Dependence:** Models restricted to 3D volume encoding (VNet*) without image-space reasoning (like DVRNet or RenderNet) struggle with view-dependent illumination, often learning only average luminance [S1, unofficial].
*   **Optimization Difficulty:** Adding parameters (e.g., VNetL16-17 vs VNet16-4) can make optimization harder for certain tasks, potentially reducing performance on Shaded data [S1, unofficial].
*   **Resolution:** Models are trained on low-resolution data ($200 \times 200$) but can generate high-resolution outputs [S1, unofficial].

## Operating characteristics
*   **Training Time:**
    *   Lookup TF: ~20 minutes total [S1, unofficial].
    *   DVRNet: Requires a quarter of the training time of VNet* variants [S1, unofficial].
    *   VNet* Variants: Dominated by the deep encoder part; e.g., VNet4-4 takes ~8h 32m, VNetL16-17 takes ~9h 46m on the Bonsai3-H dataset (Table 2 in [S1, unofficial]).
*   **Performance:**
    *   **Kidney Data:** VNet* variants (especially with latent color space) perform better than DVRNet, which struggles with complex 3D features [S1, unofficial].
    *   **Shading Data:** DVRNet excels due to 2D convolutional upsampling allowing image-space reasoning for view-dependent illumination [S1, unofficial]. RenderNet captures more illumination details by transforming volume to view space before 3D convolutions [S1, unofficial].
*   **Metrics:** Evaluated using LPIPS (lower is better), FID (lower is better), and SSIM (higher is better) [S1, unofficial].
*   **Strengths:** End-to-end learning of visual features without manual TF design; handles inter-patient variations; can learn view-dependent effects [S1, unofficial].
*   **Limitations:** Lookup tables are limited to intensity-based features; VNet* variants may be restricted to ambient occlusion-like features; RenderNet can produce blurry results [S1, unofficial].

## Background
Generalized Rendering Models were introduced in the context of Deep Direct Volume Rendering (DeepDVR) to address the difficulty of manually designing robust transfer functions for clinical data with high inter-patient variations [S1, unofficial]. The work demonstrates that deep architectures can learn meaningful rendering features purely from image-space data, outperforming traditional manual transfer functions in tasks like colorization and shading [S1, unofficial].

## Sources
[S1] Deep Direct Volume Rendering: Learning Visual Feature Mappings From Exemplary Images, https://doi.org/10.48550/arxiv.2106.05429, fetched 2026-10-26 (unofficial)
