# Fully Convolutional Discriminator

Also called: Discriminator Network (Learned Self-Supervision), Learned Self-Supervision via GAN Discriminator

## Overview
A fully convolutional discriminator used for learned self-supervision in single-view volume reconstruction to constrain degrees of freedom and prevent smearing or streak artifacts [S1]. It differs from sibling methods by evaluating randomly sampled viewpoints to provide image-space constraints coupled over time via global transport, rather than relying on fixed multi-view inputs or patch-based scaling [S1].

## How it works
**Inputs:**
1.  A small dataset of example motions (real samples) [S1].
2.  Rendered fake samples ($\hat{I}_f$) from the current reconstruction [S1].
3.  A history of previously rendered fake samples [S1].

**Outputs:**
1.  Discriminator loss gradients used to update the reconstruction network [S1].
2.  Discriminator weights $\Theta_D$ [S1].

**Data Structures:**
*   **Discriminator Weights ($\Theta_D$):** Parameters for 14 convolutional layers.
*   **History Buffer:** Stores previously rendered fake samples for additional training data [S1].

**Architecture Layout:**
The discriminator $D$ is a fully convolutional network with 14 layers. All layers use a filter size of $4 \times 4$. The sequence of (output channels, stride) pairs is:
1.  $(8, 1)$
2.  $(16, 2)$
3.  $(16, 1)$
4.  $(24, 2)$
5.  $(24, 1)$
6.  $(32, 2)$
7.  $(32, 1)$
8.  $(32, 1)$
9.  $(64, 2)$
10. $(64, 1)$
11. $(64, 1)$
12. $(16, 1)$
13. $(4, 1)$
14. $(1, 1)$

Each layer is followed by a Leaky ReLU (LReLU) activation with a leak factor of 0.2 [S1]. Padding is "SAME", but implemented by mirroring the input at the boundaries instead of zero-padding to avoid artifacts [S1]. L2 regularization is applied to the weights $\Theta_D$ [S1].

**Training Procedure:**
1.  **Sample Selection:** Select 8 real samples from the example motion dataset and 4 rendered fake samples from the current reconstruction [S1].
2.  **History Augmentation:** When training the discriminator, include 4 additional fake samples from the history buffer [S1].
3.  **Data Augmentation:** Apply random crop, scale, rotation, intensity scaling, and gamma scaling to the input samples [S1].
4.  **Forward Pass:** Compute discriminator outputs for real and fake samples.
5.  **Loss Calculation:** Use the relativistic GAN 'average' formulation with the least-squares variant [S1]. This variant allows different resolutions and batch sizes for real and fake samples [S1].
6.  **Update:** Update $\Theta_D$ using the computed loss and L2 regularization [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_discriminator(real_samples, fake_samples, history_fake_samples, discriminator_weights, lr):
    """
    Updates the discriminator weights using real samples, current fake samples,
    and historical fake samples.
    """
    # Combine samples: 8 real, 4 current fake, 4 historical fake
    # Note: Source specifies batch size of 8 real and 4 fake for update,
    # plus 4 additional from history.
    all_real = real_samples[:8]
    all_fake = fake_samples[:4] + history_fake_samples[:4]
    
    # Apply data augmentation
    all_real = apply_augmentation(all_real) # random crop, scale, rotation, intensity, gamma
    all_fake = apply_augmentation(all_fake)
    
    # Forward pass through fully convolutional network
    # Architecture: 14 conv layers, 4x4 filters, strides as defined in layout
    # Activation: LReLU(0.2) after each layer
    # Padding: Mirrored SAME padding
    logits_real = discriminator_forward(all_real, discriminator_weights)
    logits_fake = discriminator_forward(all_fake, discriminator_weights)
    
    # Relativistic Average Least-Squares GAN Loss
    # L_D = E[ (D(x) - mean(D(x_fake)))^2 ] + E[ (1 - D(x_fake) + mean(D(x_real)))^2 ]
    # Note: Source cites relativistic 'average' formulation with least-squares variant.
    mean_real = mean(logits_real)
    mean_fake = mean(logits_fake)
    
    loss_real = mean((logits_real - mean_fake) ** 2)
    loss_fake = mean((1 - logits_fake + mean_real) ** 2)
    
    # L2 Regularization on weights
    l2_reg = sum(w ** 2 for w in discriminator_weights)
    
    total_loss = loss_real + loss_fake + lambda_l2 * l2_reg
    
    # Backpropagate and update weights
    gradients = compute_gradients(total_loss, discriminator_weights)
    discriminator_weights = discriminator_weights - lr * gradients
    
    return discriminator_weights
```

**Parameters:**
*   **Batch Size:** 8 real samples, 4 rendered fake samples, 4 historical fake samples [S1].
*   **Leak Factor:** 0.2 [S1].
*   **Filter Size:** $4 \times 4$ for all layers [S1].

**Edge Cases and Pitfalls:**
*   **Pre-trained Reuse:** Reusing a pre-trained discriminator yields inferior results. It must be re-trained alongside each new reconstruction because the discriminator adapts to provide feedback for different aspects of the solution over time [S1].
*   **Padding:** Zero-padding should be avoided; mirrored padding is required to prevent specific artifacts [S1].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** The full method (including this discriminator) measures an average reconstruction time of ~30 minutes per frame using a single GPU [S1].
*   **Strengths:** Allows working with very sparse observations, even a single viewpoint [S1]. Provides image-space constraints coupled over time [S1].
*   **Limitations:** Requires a small dataset of example motions [S1]. Assumes self-similarity in fluid flow [S1]. Dissipation effects are not modeled, leading to errors in divergence constraints [S1].

## Background
Originated in "Global Transport for Fluid Reconstruction with Learned Self-Supervision" (2021) by researchers from TU Berlin (tum-pbs) [S1]. The method is part of a pipeline for recovering end-to-end motion of fluid sequences from single input views, combining physical transport constraints with differentiable volumetric rendering and learned self-supervision [S1].

## Sources
[S1] Global Transport for Fluid Reconstruction with Learned Self-Supervision, https://doi.org/10.1109/cvpr46437.2021.00168, fetched 2026-10-30
