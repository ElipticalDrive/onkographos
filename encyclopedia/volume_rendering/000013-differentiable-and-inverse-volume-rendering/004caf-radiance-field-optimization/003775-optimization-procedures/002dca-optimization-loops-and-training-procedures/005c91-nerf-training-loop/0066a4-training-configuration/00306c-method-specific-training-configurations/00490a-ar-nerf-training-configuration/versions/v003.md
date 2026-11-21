# AR-NeRF Training Configuration

Also called: AR-NeRF Implementation Details, AR-NeRF architecture and training configuration

## Overview
AR-NeRF [S1] is a method for unsupervised learning of depth and defocus effects from natural images by integrating aperture rendering into a Neural Radiance Field (NeRF) framework. It solves the problem of generating novel views with realistic depth-of-field effects without requiring explicit depth or camera aperture labels. It distinguishes itself from standard NeRF variants by explicitly modeling the camera aperture to simulate defocus blur, allowing for continuous control over focus distance and aperture size during inference.

## How it works
AR-NeRF builds on the generative NeRF framework of pi-GAN [S1]. The method trains a synthesis network to predict radiance fields conditioned on a latent code, while a discriminator ensures photorealism. The training configuration is specific to the datasets used (FFHQ, Oxford Flowers, CUB-200-2011) and involves specific architectural choices to balance performance and memory.

### Inputs and Outputs
*   **Inputs:** Training images (unlabeled natural images), random latent codes $\mathbf{z}$ sampled from a standard normal distribution.
*   **Outputs:** Rendered images $\hat{I}$ with simulated defocus effects, optimized network parameters.

### Data Structures and Layouts

**Mapping Network**
*   **Architecture:** Multi-Layer Perceptron (MLP).
*   **Layers:** 3 hidden layers.
*   **Units:** 256 units per layer.
*   **Activation:** Leaky Rectified Linear Unit (LReLU) with negative slope 0.2 [S1].
*   **Latent Code Dimension:** 256 [S1].
*   **Purpose:** Propagates information from the latent code to each layer of the synthesis network, similar to StyleGAN [S1].

**Foreground Synthesis Network**
*   **Architecture:** MLP with FiLM (Feature-wise Linear Modulation) [S1] and SIREN (Sinusoidal Representation Networks) [S1] hidden layers.
*   **Layers:** 8 hidden layers.
*   **Units:** 128 units per layer.
*   **Note:** The original pi-GAN used 256 units per layer. AR-NeRF reduces this to 128 to allow a larger number of points per ray, which improves performance without significantly affecting output quality [S1].

**Background Synthesis Network**
*   **Architecture:** MLP with FiLM-SIREN hidden layers.
*   **Layers:** 8 hidden layers.
*   **Units:** 64 units per layer.
*   **Rationale:** Fewer parameters are used under the assumption that the background is simpler than the foreground [S1].

**Discriminator**
*   **Architecture:** Convolutional network with residual blocks [S1].
*   **Variants:**
    *   **FFHQ:** Includes CoordConv layers [S1].
    *   **Oxford Flowers / CUB-200-2011:** Excludes CoordConv layers. Preliminary experiments showed CoordConv layers had negative effects on these datasets, likely because flowers and birds are not strictly aligned by facial landmarks like faces in FFHQ [S1].

### Training Parameters and Optimization

**Optimizer**
*   **Algorithm:** Adam [S1].
*   **Learning Rate:** 0.00006 [S1].
*   **Iterations:** 200,000 [S1].

**Loss Functions and Regularization**
*   **GAN Objective:** Non-saturating GAN loss [S1].
*   **R1 Regularization:** Applied to the real gradient penalty. Weight $\lambda_{R1} = 0.2$ [S1].
*   **Identity Regularization:** Used to keep identity consistent across different viewpoints. Weight $\lambda_{id} = 15$ [S1].

### Implementation Steps

1.  **Dataset Preparation:**
    *   Load training images.
    *   **FFHQ:** 70,000 face images [S1].
    *   **Oxford Flowers:** 8,189 images, 102 categories [S1].
    *   **CUB-200-2011:** 11,788 images, 200 bird species [S1].

2.  **Network Initialization:**
    *   Initialize Mapping Network (3x256 MLP, LReLU).
    *   Initialize Foreground Synthesis Network (8x128 FiLM-SIREN MLP).
    *   Initialize Background Synthesis Network (8x64 FiLM-SIREN MLP).
    *   Initialize Discriminator (CoordConv + ResBlocks for FFHQ; ResBlocks only for Flowers/CUB) [S1].

3.  **Training Loop (per iteration):**
    *   Sample a batch of real images $I_{real}$.
    *   Sample random latent codes $\mathbf{z}$.
    *   Generate images $I_{gen}$ using the synthesis networks conditioned on $\mathbf{z}$ and aperture parameters (if applicable for the specific rendering mode) [S1].
    *   Compute GAN loss (non-saturating) [S1].
    *   Compute R1 regularization loss on real images with weight 0.2 [S1].
    *   Compute Identity regularization loss with weight 15 [S1].
    *   Update all network parameters using Adam optimizer with learning rate 0.00006 [S1].

4.  **Inference/Rendering:**
    *   To generate images with specific defocus, vary the aperture size $s$ and focus distance $f$ [S1].
    *   For higher resolution outputs (e.g., 128x128), increase the density of input points along the rays [S1].

### Edge Cases and Pitfalls
*   **CoordConv Usage:** Using CoordConv layers on datasets with unaligned objects (flowers, birds) degrades performance [S1].
*   **Ray Sampling:** Using stratified sampling is critical. Models with the same number of rays but uniform sampling perform worse than those with stratified sampling [S1].
*   **Resolution vs. Points:** Increasing image resolution requires increasing the number of points per ray to maintain quality, which increases processing time and memory usage [S1].

## Subtopics

### AR-NeRF-R
AR-NeRF-R is a defocus renderer trained on images generated by AR-NeRF. It allows for continuous and intuitive manipulation of defocus strength and focus distance given an input image [S1].

*   **Definition:** A conditional U-Net [S1] that takes an image and auxiliary parameters to output a defocused version.
*   **Architecture:** U-Net with two auxiliary scalar inputs: aperture size $s$ and focus distance $f$. These are injected as conditioning signals [S1].
*   **Training Data:** 128x128 images generated by AR-NeRF (using increased point density to support higher resolution) from the Oxford Flowers dataset [S1].
*   **Evaluation:** Tested on the iPhone2DSLR Flower dataset [S1].
*   **Comparison:** Unlike AR-GAN-DR, where defocus parameters are uniquely determined by the input image, AR-NeRF-R allows interactive, continuous adjustment of $s$ and $f$ via auxiliary inputs [S1].

## Operating characteristics
*   **Memory:** The reduction of synthesis network units from 256 to 128 allows for a larger number of points per ray, improving performance but requiring careful memory management as point density increases with resolution [S1].
*   **Performance:** AR-NeRF achieves lower FID and KID scores compared to baseline GANs (AR-GAN, AR-GAN++, RGBD-GAN) on Oxford Flowers, CUB-200-2011, and FFHQ datasets [S1].
*   **Scalability:** Processing time and memory increase with the number of rays and points per ray. Higher resolutions require more points [S1].

## Background
AR-NeRF [S1] is based on pi-GAN [S1], a state-of-the-art generative variant of NeRF. The method was developed to address the limitations of previous generative NeRF models in handling defocus effects. It was applied to three datasets: Oxford Flowers, CUB-200-2011, and FFHQ [S1]. The authors note that the original pi-GAN was not applied to these specific datasets, so configurations were carefully tuned [S1].

## Sources
[S1] AR-NeRF: Unsupervised Learning of Depth and Defocus Effects from Natural Images with Aperture Rendering Neural Radiance Fields, https://arxiv.org/abs/2206.06100v1, fetched 2026-10-26
