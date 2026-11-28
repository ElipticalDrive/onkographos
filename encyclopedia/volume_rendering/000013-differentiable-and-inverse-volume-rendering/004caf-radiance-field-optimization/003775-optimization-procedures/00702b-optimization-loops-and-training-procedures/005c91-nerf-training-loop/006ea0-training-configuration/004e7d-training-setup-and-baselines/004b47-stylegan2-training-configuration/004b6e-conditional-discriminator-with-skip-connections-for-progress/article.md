# Conditional Discriminator with Skip Connections for Progressive Growing

Also called: StyleGAN2 Conditional Discriminator

## Overview
A conditional discriminator based on StyleGAN2 that uses skip connections for progressive growing [S1]. It integrates conditional inputs and adds a downsampled version of the input image to the input of each layer [S1]. It differs from alternative progressive methods like Progressive Up-sampling by requiring linear fading of newly added layers rather than inserting upsample operations [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A batch of images with conditional inputs [S1].
- Output: A scalar score indicating real vs. fake [G].

**Data Structures:**
- The discriminator follows the StyleGAN2 architecture [S1].
- Each layer receives two inputs: the standard feature map from the previous layer and a downsampled version of the original input image [S1].

**Steps:**
1. **Input Preparation:** The input image is downsampled to match the resolution of the current training stage [S1].
2. **Skip Connection Injection:** At the input of *each* layer in the discriminator, the downsampled input image is added to the feature map [S1].
3. **Conditional Processing:** The discriminator processes the combined features using the standard StyleGAN2 convolutional blocks [S1].
4. **Progressive Growing Integration:** As the training resolution increases, new layers are added and linearly interpolated (faded in) with the previous low-resolution output [S2, unofficial].
5. **Final Classification:** The output passes through a minibatch standard deviation layer and a final classification layer [S1].

**Parameters:**
- Learning rate for generator/discriminator balance ($\eta_{fg}$): 0.4 for AFHQ and FFHQ datasets; 0.1 for Carla dataset (due to smaller object fraction) [S1].
- Background learning rate ($\eta_{bg}$): 0.1 for all datasets [S1].
- Consistency loss weights ($\lambda_{fcvg}$, $\lambda_{bg}$): 0.1 for all datasets [S1].

**Edge Cases and Pitfalls:**
- **Carla Dataset:** Requires lower learning rates ($\eta_{fg} = 0.1$) because objects cover a smaller fraction of the image [S1].
- **Progressive Growing Stability:** Linear interpolation of new layers is essential when using progressive growing to prevent training instability due to random initialization of new parameters [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def discriminator_forward(image, layers):
    # Downsample input image to match current training resolution [G]
    downsampled_image = average_pool(image, target_size=image.shape[-1] // 2)

    features = image
    for layer in layers:
        # Add skip connection: downsampled input to each layer [S1]
        features = features + downsampled_image
        # Standard StyleGAN2 convolutional block processing [S1]
        features = layer(features)

    # Minibatch standard deviation layer at the end [S1]
    batch_std = features.std(dim=0, keepdim=True)
    features = concatenate([features, batch_std], dim=1)

    # Final classification layer projecting to a scalar score [G]
    output = sum(features)

    return output
```

## Subtopics

### Progressive Up-sampling
An alternative progressive training method that inserts upsample operations during training instead of growing new layers [S2, unofficial]. Unlike progressive growing, this method does not require linearly fading in layers [S2, unofficial]. It was found to work similarly in visual quality to progressive growing, but progressive growing offers better speed advantages for low-resolution images [S2, unofficial].

### Conditional Discriminator with Relative Pose Conditioning
A GAN training setup where the discriminator is conditioned on relative human pose to handle view-biased 2D generator outputs and ensure consistent texture generation [S3]. During training, the input image from the 2D generator is calibrated with PyMAF to obtain the relative human root transformation $M_{\text{human}}$ and skeleton pose [S3]. For a sampled camera view $v$ from the dataset, the actual rendering view is computed as $v_r = M_{\text{human}}^{-1} v$ [S3].

### Conditional Discriminator with Gradient Regularization
A discriminator $D$ conditioned on camera pose $\xi$ and motion feature $m$, using a non-saturating adversarial loss with gradient regularization [S4]. During training, a training image is randomly sampled with its corresponding estimated motion feature and pose to control image synthesis [S4]. The adversarial loss is defined as $L_{\text{adv}} = \mathbb{E}_{I_0 \sim p_{\text{syn}}, m \sim p_m, \xi \sim p_\xi}[f(D(I_0, \xi, m))] + \mathbb{E}_{I \sim p_{\text{real}}}[f(-D(I)) + \lambda \|\nabla_D(I)\|_2]$, where $f(u) = -\log(1 + \exp(-u))$ [S4]. The gradient regularization weight $\lambda$ is a coefficient for the penalty term $\|\nabla_D(I)\|_2$, though its exact value is not explicitly stated in the text [S4].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Progressive growing achieves better speed advantages for low-resolution images compared to Progressive Up-sampling [S2, unofficial].
- **Strengths:** Supports progressive resolution increases via skip connections and linear layer fading [S1][S2, unofficial].
- **Limitations:** Requires careful tuning of learning rates for datasets with small object fractions (e.g., Carla) [S1].

## Background
Originated from the StyleGAN2 architecture [S1]. The specific use of conditional inputs and skip connections for progressive growing was detailed in the context of 3D-aware image synthesis methods like VoxGRAF [S1] and StyleNeRF [S2, unofficial]. The approach builds on the foundational progressive growing technique by Karras et al. (2017) [S2, unofficial].

## Sources
[S1] VoxGRAF: Fast 3D-Aware Image Synthesis with Sparse Voxel Grids, https://doi.org/10.52202/068431-2464, fetched 2026-10-26
[S2] StyleNeRF: A Style-based 3D-Aware Generator for High-resolution Image Synthesis, https://doi.org/10.48550/arxiv.2110.08985, fetched 2026-10-26 (unofficial)
[S3] HumanGen: Generating Human Radiance Fields with Explicit Priors, https://doi.org/10.1109/cvpr52729.2023.01207, fetched 2026-10-29
[S4] Controllable Radiance Fields for Dynamic Face Synthesis, https://doi.org/10.1109/3dv57658.2022.00075, fetched 2026-10-29
