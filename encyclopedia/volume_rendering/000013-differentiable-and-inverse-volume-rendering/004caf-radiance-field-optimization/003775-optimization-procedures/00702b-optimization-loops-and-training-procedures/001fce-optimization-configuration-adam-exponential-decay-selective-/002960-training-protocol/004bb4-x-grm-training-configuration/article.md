# X-GRM Training Configuration

Also called: X-GRM Implementation Details

## Overview
X-GRM is a large Gaussian reconstruction model designed for sparse-view X-ray to Computed Tomography (CT) reconstruction [S1, unofficial]. This configuration defines the specific optimization protocol, including the AdamW optimizer settings, learning rate scheduling, and variable-view learning strategy, to train the model on synthetic radiographic projections [S1, unofficial]. It distinguishes itself from other training protocols by employing a variable-number-of-views strategy during training to enhance robustness against sparse input views, combined with bFloat16 precision and gradient checkpointing for memory efficiency on large-scale GPU clusters [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** Synthetic radiographic projections. The dataset consists of 50 high-definition views (256x256 resolution) distributed uniformly throughout the complete angular spectrum $[0^\circ, 360^\circ]$ [S1, unofficial].
*   **Output:** Reconstructed 3D volumetric data (CT volume) [S1, unofficial].

**Data Structures and Model Architecture**
The model consists of two Vision Transformer (ViT) components [S1, unofficial]:
1.  **Encoder ViT ($F_{enc}$):** Uses ViT-B/16 architecture. Initialized from DINO pre-trained weights [S1, unofficial].
2.  **Fusion ViT ($F_{fuse}$):** Uses ViT-B/16 architecture. Initialized from scratch. Configuration: 16x16 patch size, 12 layers, 12 heads, embedding dimension 768, MLP ratio 4.0 [S1, unofficial].

**Training Parameters**
*   **Optimizer:** AdamW [S1, unofficial].
*   **Batch Size:** 8 [S1, unofficial].
*   **Epochs:** 100 [S1, unofficial].
*   **Learning Rate Schedule:** Cosine annealing [S1, unofficial].
    *   Initial Learning Rate: $1 \times 10^{-4}$ [S1, unofficial].
    *   Final Learning Rate: $1 \times 10^{-6}$ [S1, unofficial].
*   **Optimizer Hyperparameters:** $\beta_1 = 0.9$, $\beta_2 = 0.95$ [S1, unofficial].
*   **Precision:** bFloat16 [S1, unofficial].
*   **Memory Optimization:** Gradient checkpointing enabled [S1, unofficial].
*   **Hardware:** 4 NVIDIA A100 (40G) GPUs for training [S1, unofficial].

**Algorithm Steps**
1.  **Data Preparation:** Generate or load synthetic radiographic projections. The source data includes 50 views with noise integration combining Gaussian and Poisson distributions to replicate quantum photon effects and Compton scatter phenomena [S1, unofficial].
2.  **Variable-View Selection:** For each training step, randomly select the number of input views $N$ from the set $\{6, 8, 10\}$ [S1, unofficial].
3.  **View Sampling:** Evenly sample $N$ perspectives from the 50 available projections [S1, unofficial].
4.  **Forward Pass:** Process the selected views through the Encoder ViT and Fusion ViT to generate the reconstruction [S1, unofficial].
5.  **Loss Computation:** Calculate the loss based on the reconstructed volume and the ground truth [S1, unofficial].
6.  **Backward Pass:** Compute gradients using bFloat16 precision with gradient checkpointing to reduce memory usage [S1, unofficial].
7.  **Update:** Update model parameters using the AdamW optimizer with the current learning rate determined by the cosine annealing schedule [S1, unofficial].
8.  **Repeat:** Iterate for 100 epochs [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# X-GRM Training Configuration
# Based on [S1, unofficial]

def train_xgrm():
    # Hyperparameters
    num_epochs = 100
    batch_size = 8
    initial_lr = 1e-4
    final_lr = 1e-6
    beta1 = 0.9
    beta2 = 0.95
    available_views = 50  # Total available projections
    view_options = [6, 8, 10]
    
    # Model Initialization
    encoder_vit = ViT_B_16(weights='DINO_PRETRAINED')
    fusion_vit = ViT_B_16(
        patch_size=16,
        layers=12,
        heads=12,
        embed_dim=768,
        mlp_ratio=4.0,
        init='scratch'
    )
    
    # Optimizer
    optimizer = AdamW(
        params=chain(encoder_vit.parameters(), fusion_vit.parameters()),
        lr=initial_lr,
        betas=(beta1, beta2)
    )
    
    # Learning Rate Scheduler (Cosine Annealing)
    scheduler = CosineAnnealingLR(
        optimizer=optimizer,
        T_max=num_epochs,
        eta_min=final_lr
    )
    
    # Training Loop
    for epoch in range(num_epochs):
        for batch in data_loader:
            # Variable-view learning strategy
            num_views = random.choice(view_options)
            
            # Evenly sample views from the 50 available projections
            # Note: Implementation detail for 'evenly sampled' is not explicitly 
            # defined in sources, assumed to be deterministic spacing or 
            # pre-defined subsets [G].
            selected_indices = evenly_sample_indices(available_views, num_views)
            input_views = batch['projections'][:, selected_indices, :, :]
            
            # Forward pass
            # encoder_vit and fusion_vit process input_views
            reconstructed_volume = forward_pass(
                input_views, 
                encoder_vit, 
                fusion_vit
            )
            
            # Compute loss (e.g., L1/L2 or perceptual loss in 3D space)
            # Source mentions PSNR calculated in 3D volumetric space [S1, unofficial]
            loss = compute_loss(reconstructed_volume, batch['ground_truth'])
            
            # Backward pass with gradient checkpointing
            loss.backward()
            
            # Optimizer step
            optimizer.step()
            optimizer.zero_grad()
            
            # Update learning rate
            scheduler.step()
```

## Subtopics

### SinGRAF Training Configuration
SinGRAF uses a tri-plane representation with StyleGAN2 generator modulated by a noise vector $z \sim \mathbb{R}^{128}$ [S2]. The tri-plane spatial resolution is 256x256 with feature channel count $C=32$ [S2]. Rendering involves bilinear interpolation of features from the three planes followed by concatenation, processed by a decoder MLP (two shared linear layers, one density layer, two RGB layers, 64 hidden units with leakyReLU) to obtain color and density [S2]. Volume rendering uses 96 samples per ray without importance sampling [S2]. Training is performed on 64x64 patches, but a patch scale of $s=0.25$ allows effective 256x256 resolution rendering without 2D upsampling [S2]. Regularization balancing parameters are $\lambda_1 = 0.5$ and $\lambda_2 = 50$ [S2]. The discriminator is a StyleGAN2 architecture with additional concatenation of patch scale [S2].

## Operating characteristics
*   **Inference Time:** 0.928 seconds on one A100 GPU for 10-view input [S1, unofficial].
*   **Performance (10-View Input):** PSNR of 29.21 and SSIM of 0.886 [S1, unofficial].
*   **Performance (6-View Input):** PSNR of 28.39 and SSIM of 0.873 [S1, unofficial].
*   **Performance (8-View Input):** PSNR of 28.86 and SSIM of 0.879 [S1, unofficial].
*   **Memory Efficiency:** Utilizes bFloat16 precision and gradient checkpointing to improve GPU memory and computational efficiency [S1, unofficial].
*   **Scalability:** Trained on 4 NVIDIA A100 (40G) GPUs [S1, unofficial].

## Background
X-GRM was introduced to address the challenge of reconstructing high-quality CT volumes from sparse-view X-ray projections [S1, unofficial]. The method leverages large Gaussian reconstruction models and Vision Transformers [S1, unofficial]. The training configuration was designed to handle synthetic radiographic data with clinical authenticity enhancements, including noise integration to replicate quantum photon effects and Compton scatter [S1, unofficial].

## Sources
[S1] X-GRM: Large Gaussian Reconstruction Model for Sparse-view X-rays to Computed Tomography, https://doi.org/10.48550/arXiv.2505.15235, fetched 2026-10-27 (unofficial)
[S2] SinGRAF: Learning a 3D Generative Radiance Field for a Single Scene, https://doi.org/10.1109/cvpr52729.2023.00822, fetched 2026-10-29
