# Diffusion Model Training for Triplane Features
Also called: DDPM Training for Triplane Features

## Overview
This method trains a Denoising Diffusion Probabilistic Model (DDPM) to generate 3D-aware triplane feature representations. It solves the problem of synthesizing coherent, high-quality 3D geometry and appearance by learning the distribution of triplane features extracted from 3D datasets. It differs from sibling methods by focusing specifically on the generative modeling of the triplane latent space itself, rather than operating in pixel space (like Latent Diffusion Models for 3D) or point cloud space (like Point Upsampling Diffusion).

## How it works
**Inputs and Outputs:**
*   **Input:** A dataset of 3D shapes, where each shape is represented by a triplane feature tensor. The triplane consists of three 2D feature planes (typically $xy$, $xz$, $yz$ projections).
*   **Output:** A trained DDPM capable of sampling new triplane feature tensors from noise.

**Data Structures:**
*   **Triplane Feature Tensor:** A 3D tensor representing the three orthogonal planes. The specific channel dimensions and spatial resolution depend on the feature extractor used (e.g., [1]), but the diffusion model treats the flattened or channel-wise normalized features as the data distribution.
*   **Normalization State:** Statistics (mean and standard deviation) per feature channel, computed from the training set.

**Preprocessing (Normalization):**
The raw triplane features are theoretically unbounded and may contain outliers. To stabilize training:
1.  Compute the mean $\mu_c$ and standard deviation $\sigma_c$ for each feature channel $c$ across the training dataset.
2.  Normalize each channel: $x'_{c} = \frac{x_c - \mu_c}{\sigma_c}$.
3.  Clip the normalized values to the range $[-S, S]$, where $S = 16$.
4.  Scale the clipped values to the range $[-1, 1]$.

**Training Procedure:**
The model is trained using the standard DDPM objective, minimizing the prediction of noise added to the triplane features.

1.  **Initialization:** Initialize the UNet-based diffusion model.
2.  **Sampling:** For each training step $t$:
    a. Sample a random batch of triplane feature tensors $X_0$ from the dataset.
    b. Apply the normalization procedure described above to $X_0$.
    c. Sample a random timestep $t$ uniformly from $[1, T]$.
    d. Sample noise $\epsilon \sim \mathcal{N}(0, I)$.
    e. Compute the noisy triplane features $X_t$ using the forward diffusion process:
       $$X_t = \sqrt{\bar{\alpha}_t} X_0 + \sqrt{1 - \bar{\alpha}_t} \epsilon$$
       where $\bar{\alpha}_t$ is the cumulative product of noise schedule coefficients.
    f. Pass $X_t$ and $t$ through the UNet to predict the noise $\epsilon_\theta$.
    g. Compute the loss:
       $$L = \mathbb{E}_{t, X_0, \epsilon} [\| \epsilon - \epsilon_\theta(X_t, t) \|^2]$$
    h. Update model parameters using gradient descent.

**Parameters:**
*   **Batch Size:** 128 [S1].
*   **Hardware:** 8 NVIDIA A6000 GPUs [S1].
*   **Learning Rate:**
    *   Cars dataset: $1 \times 10^{-4}$ [S1].
    *   Chairs and Planes datasets: $3 \times 10^{-5}$ [S1].
*   **Training Steps:**
    *   Cars: 400,000 steps [S1].
    *   Chairs: 200,000 steps [S1].
    *   Planes: 200,000 steps [S1].
*   **Pretraining (Cars):** The cars model is pretrained on a subset of the cars data for 160,000 steps before continuing on the full dataset for the remaining 240,000 steps [S1].
*   **Hyperparameters:** DDPM hyperparameters (noise schedule, etc.) are identical to the class-specific LSUN model in [1] [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader

# Assume DiffusionModel is a UNet architecture defined elsewhere
# Assume TriplaneDataset returns normalized triplane tensors in [-1, 1]

def train_diffusion_for_triplanes(model, train_loader, optimizer, num_steps, lr, device):
    model.train()
    scheduler = None # Use standard cosine or linear schedule if specified in [1]
    
    for step in range(num_steps):
        # 1. Load batch
        triplanes = next(iter(train_loader)) # Batch size 128
        triplanes = triplanes.to(device)
        
        # 2. Sample timestep t
        t = torch.randint(1, model.num_timesteps, (triplanes.shape[0],), device=device)
        
        # 3. Sample noise
        noise = torch.randn_like(triplanes)
        
        # 4. Add noise to get noisy triplanes
        # Assumes model has a forward_diffusion method or uses standard DDPM formula
        noisy_triplanes = model.q_sample(triplanes, t, noise=noise)
        
        # 5. Predict noise
        predicted_noise = model(noisy_triplanes, t)
        
        # 6. Compute loss
        loss = nn.functional.mse_loss(predicted_noise, noise)
        
        # 7. Backpropagate
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Logging and checkpointing would occur here
```

**Edge Cases and Pitfalls:**
*   **Outliers:** Raw triplane features can have unbounded values. Failure to clip to $S=16$ standard deviations and scale to $[-1, 1]$ can skew the distribution and destabilize training [S1].
*   **Instability:** Using a high learning rate (e.g., $1 \times 10^{-4}$) on datasets like chairs or planes can cause training instability. A lower learning rate ($3 \times 10^{-5}$) is required for these classes [S1].
*   **Pretraining:** For complex datasets like cars, pretraining on a subset is necessary to prevent mode collapse or poor convergence [S1].

## Subtopics

### [Triplane Feature Synthesis](18449-triplane-feature-synthesis.md)
Generates expressive 3D features using three orthogonal 2D planes with extra synthesis blocks for disentanglement. It builds on this method by using the generated triplane features as the initial 3D-aware representation.

### [Unbiased Score Distillation (USD) Rectification](21250-unbiased-score-distillation-usd-rectification.md)
Corrects unconditional noise bias in diffusion models for novel view synthesis. It addresses inconsistencies arising from fine-tuned models retaining base model noise prediction characteristics.

## Operating characteristics
*   **Time Complexity:** Training time scales with the number of steps and batch size. Cars model took ~6 days for 400k steps; chairs and planes took ~3 days each for 200k steps on 8 A6000 GPUs [S1].
*   **Memory Complexity:** Depends on the triplane resolution and channel depth, plus the memory required for the UNet and gradients. Batch size of 128 requires significant GPU memory [S1].
*   **Strengths:** Produces high-quality, diverse 3D-aware feature representations. The normalization strategy ensures stable training despite unbounded feature spaces.
*   **Limitations:** Requires significant computational resources (8 GPUs, multiple days). Performance is sensitive to learning rate choices, varying by dataset complexity.
*   **When to use:** When the goal is to generate novel 3D shapes in the form of triplane features for downstream rendering or editing.
*   **When not to use:** When real-time inference is critical without pre-generation, or when computational resources are limited.

## Background
This method is based on the work presented in "3D Neural Field Generation Using Triplane Diffusion" [S1]. It adapts the successful DDPM framework, originally developed for image generation (e.g., LSUN models [1]), to the domain of 3D triplane features. The approach leverages the ability of diffusion models to model complex distributions, applying it to the latent space of 3D representations.

## Sources
[S1] 3D Neural Field Generation Using Triplane Diffusion, https://doi.org/10.1109/cvpr52729.2023.02000, fetched 2026-10-26
