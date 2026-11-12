# StyleGAN2 Training Configuration

Also called: StyleGAN2 Training Setup

## Overview
A training configuration adapted from the StyleGAN2 architecture for optimizing 3D-aware radiance fields, characterized by equalized learning rates, minibatch standard deviation in the discriminator, exponential moving average (EMA) of generator weights, mixed-precision arithmetic, and sparse R1 regularization. It distinguishes itself from standard StyleGAN2 by omitting path regularization and style-mixing to reduce computational overhead in the context of volume rendering [S1].

## How it works
**Inputs and Outputs**
- **Inputs:** Generator weights, Discriminator weights, minibatch of images, latent vectors.
- **Outputs:** Updated Generator and Discriminator weights (EMA updated generator weights are used for evaluation).

**Data Structures**
- **Generator/Discriminator Parameters:** Standard convolutional and dense layer weights.
- **EMA State:** Separate copy of generator weights, updated per step.
- **Minibatch Statistics:** Standard deviation calculated over the batch dimension for the discriminator's final layer.

**Parameters**
- **R1 Regularization Strength ($\gamma$):** 1.0 [S1].
- **R1 Regularization Frequency:** Once every 4 minibatches [S1].
- **Learning Rate:** Equalized across all trainable parameters [S1].
- **Precision:** Mixed-precision (FP16/FP32) for both Generator and Discriminator [S1].

**Steps**
1. **Equalized Learning Rates:** Scale the learning rate for each trainable parameter in the Generator and Discriminator inversely proportional to the magnitude of the parameter's update or weight, ensuring stable training across layers of varying scales [S1].
2. **Discriminator Architecture:**
   - Apply a minibatch standard deviation layer at the end of the discriminator. This computes the standard deviation of feature maps across the batch dimension and concatenates this statistic channel-wise to the feature map before the final pooling/linear layer [S1].
3. **Regularization (R1):**
   - Calculate the R1 gradient penalty on the discriminator's output with respect to the input images.
   - Apply this penalty only once every 4 minibatches to reduce computational cost [S1].
   - The penalty term is $\gamma \cdot \frac{1}{2} \|\nabla_{\hat{x}} D(\hat{x})\|^2$, where $\hat{x}$ is a random sample from the real data distribution [S1].
4. **Exponential Moving Average (EMA):**
   - After each optimizer step, update the generator's shadow weights ($\theta_{EMA}$) using the current weights ($\theta$):
     $$ \theta_{EMA} \leftarrow \alpha \theta_{EMA} + (1 - \alpha) \theta $$
     where $\alpha$ is the decay rate (typically close to 1, e.g., 0.9999, though exact value not specified in sources) [S1].
5. **Mixed Precision:**
   - Perform forward and backward passes for both Generator and Discriminator using mixed-precision arithmetic to accelerate training and reduce memory usage [S1].
6. **Exclusions:**
   - Do **not** apply path regularization [S1].
   - Do **not** apply style-mixing [S1].

**Edge Cases and Pitfalls**
- **Small Datasets:** For small datasets like AFHQ, standard training may fail. The recommended pitfall avoidance is to fine-tune a generator pretrained on FFHQ. Specifically, use a truncation psi ($\psi$) of 0.7 for sampling and initialize with a generator that has $R_I = 128$ and $R_G = 32$ (before pruning) [S1].
- **Rendering Skip:** While not part of the GAN training loop itself, the associated volume rendering skips voxels with $\sigma_i < 10^{-10}$ and stops early if $T_i < 10^{-7}$, which affects the gradient flow into the generator [S1].

## Subtopics

### Conditional Discriminator with Skip Connections for Progressive Growing
A variant of the discriminator used to facilitate progressive growing. It uses conditional input and implements a skip architecture that adds a downsampled version of the input image to the input of each layer, as introduced in StyleGAN2 [S1].

## Operating characteristics
- **Time Complexity:** Reduced compared to full StyleGAN2 due to the omission of path regularization and style-mixing, and the application of R1 regularization only every 4 minibatches [S1].
- **Memory Complexity:** Reduced due to mixed-precision training and sparse R1 regularization [S1].
- **Strengths:** Efficient training for 3D-aware synthesis; stable convergence via equalized learning rates and EMA; robustness via minibatch std dev [S1].
- **Limitations:** Requires careful handling for small datasets (e.g., AFHQ) via fine-tuning from FFHQ [S1].
- **When to Use:** When training 3D-aware generative models (like VoxGRAF) where computational efficiency is critical and style-mixing/path regularization are not beneficial for the specific radiance field representation [S1].

## Background
Originating from the StyleGAN2 architecture [22], this configuration was adapted for 3D-aware image synthesis with sparse voxel grids (VoxGRAF) [S1]. The specific modifications (removing path regularization/style-mixing, sparse R1) were introduced to optimize training for volume rendering tasks, leveraging custom CUDA kernels and efficient sampling strategies [S1]. The use of equalized learning rates and minibatch standard deviation follows the original StyleGAN2 contributions [19, 22].

## Sources
[S1] VoxGRAF: Fast 3D-Aware Image Synthesis with Sparse Voxel Grids, https://doi.org/10.52202/068431-2464, fetched 2026-10-26
