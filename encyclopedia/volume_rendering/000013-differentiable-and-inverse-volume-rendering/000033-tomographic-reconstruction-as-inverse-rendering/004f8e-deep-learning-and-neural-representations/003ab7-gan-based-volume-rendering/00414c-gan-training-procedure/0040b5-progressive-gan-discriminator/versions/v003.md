# Progressive GAN Discriminator

Also called: Progressive Growing Discriminator

## Overview
A convolutional discriminator that grows progressively in resolution to stabilize training and allow larger batch sizes at lower resolutions [S1]. It solves the instability inherent in training high-resolution GANs by starting with coarse shapes and gradually introducing fine details [S1]. Unlike standard ProgressiveGAN where the generator architecture grows, this variant keeps the generator fixed, increasing resolution by sampling rays more densely from the same implicit representation [S1].

## How it works
**Inputs and Outputs:**
*   Input: Real or generated images at the current training resolution [S1].
*   Output: A scalar score indicating the probability that the input is real [S1].

**Data Structures:**
*   The discriminator is a convolutional neural network that grows progressively in resolution [S1].
*   Exact layouts, fields, sizes, and bit packing are not given in the sources.

**Steps:**
1.  **Initialization:** Begin training at a low resolution (e.g., $32 \times 32$) with a large batch size [S1].
2.  **Progressive Growth:** Double the resolution twice during training, up to $128 \times 128$ [S1].
3.  **Layer Addition:** Add new layers to the discriminator to handle higher resolutions and discriminate fine details as resolution increases [S1].
4.  **Final Rendering:** Final results are rendered by sampling $512 \times 512$ pixels, though training occurs at lower resolutions [S1].

**Formulas:**
*   Not given in the sources.

**Parameters:**
*   Initial Resolution: $32 \times 32$ [S1].
*   Final Training Resolution: $128 \times 128$ [S1].
*   Final Rendering Resolution: $512 \times 512$ [S1].

**Edge Cases and Pitfalls:**
*   Not given in the sources.

## Subtopics

### [Discriminator and Decoder](16691-discriminator-and-decoder.md)
The Discriminator and Decoder constitute the adversarial training components for differentiable volume rendering, specifically within Progressive GAN frameworks. The Discriminator evaluates the realism of rendered RGB-D samples against real data using residual blocks and minibatch standard deviation, while the Decoder reconstructs high-resolution feature maps from low-resolution discriminator features. This setup stabilizes training by providing gradient signals that enforce photometric and geometric consistency. It differs from sibling methods like [Progressive Patch Scaling](17543-progressive-patch-scaling.md) by operating on full-resolution feature maps via transposed convolutions rather than patch-based scaling, and from [Fully Convolutional Discriminator](21644-fully-convolutional-discriminator.md) by explicitly incorporating a separate decoder stage for feature upsampling.

### [Progressive Patch Scaling](17543-progressive-patch-scaling.md)
Progressive Patch Scaling is a training strategy for 3D GAN discriminators that operates on image patches of varying scales rather than full-resolution images, allowing the model to learn multi-scale features progressively. It solves the instability and diversity issues inherent in training 3D GANs from limited 2D observations by gradually reducing the scale of image patches used for training, starting from large sub-regions to smaller ones. Unlike methods that apply super-resolution modules to rendered values, this approach directly discriminates RGB color images or patches, relying on the progressive reduction of patch scale to stabilize training and promote diversity.

### [Fully Convolutional Discriminator](21644-fully-convolutional-discriminator.md)
A fully convolutional discriminator used for learned self-supervision in single-view volume reconstruction to constrain degrees of freedom and prevent smearing or streak artifacts. It differs from sibling methods by evaluating randomly sampled viewpoints to provide image-space constraints coupled over time via global transport, rather than relying on fixed multi-view inputs or patch-based scaling.

### Fully Convolutional Discriminator with Mirrored Padding
All layers are followed by LReLU activations with a leak of 0.2 [S2]. Data augmentation applies random crop, scale, rotation, intensity, and gamma scale [S2]. The architecture consists of 14 convolution layers with filter sizes and strides: (8,1), (16,2), (16,1), (24,2), (24,1), (32,2), (32,1), (32,1), (64,2), (64,1), (64,1), (16,1), (4,1), (1,1), all using 4×4 filters [S2]. For padding, it uses SAME padding but applies mirrored input instead of zero padding to avoid issues described in [31] [S2].

## Operating characteristics
*   **Stability:** Allows larger batch sizes at the beginning of training, which helped to stabilize and speed training [S1].
*   **Resolution Scaling:** Training resolution is limited to $128 \times 128$ in standard experiments, while final rendering uses $512 \times 512$ [S1].
*   **Generator Independence:** The generator architecture does not grow; resolution is increased by sampling rays more densely from the same implicit representation [S1].
*   **Time and Memory Complexity:** Not given in the sources.

## Background
Originated from ProgressiveGAN [25] (cited in [S1]), this method was adapted for 3D-aware image synthesis in pi-GAN [S1]. It addresses the difficulty of training GANs at high resolutions by progressively introducing detail [S1].

## Sources
[S1] pi-GAN: Periodic Implicit Generative Adversarial Networks for 3D-Aware Image Synthesis, https://doi.org/10.1109/cvpr46437.2021.00574, fetched 2026-10-26
[S2] Global Transport for Fluid Reconstruction with Learned Self-Supervision, https://doi.org/10.1109/cvpr46437.2021.00168, fetched 2026-10-30
