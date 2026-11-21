# Adversarial Training (GAN)

Also called: Discriminator Training, GAN Loss

## Overview
Adversarial Training introduces a discriminator network trained in parallel with the radiance field generator to classify rendered images as ground truth or predicted [S1]. It distinguishes itself from standard differentiable rendering pipelines by adding a game-theoretic loss term that encourages photorealism and high-frequency detail, addressing the blurriness often associated with pure pixel-wise reconstruction losses [S1].

## How it works
**Inputs and Outputs:**
*   **Generator Input ($z$):** The latent representation or low-resolution input passed through the radiance field network (e.g., super-resolution network) to produce a predicted image $G(z)$ [S1].
*   **Discriminator Input:** Pairs of images consisting of real ground truth images $x$ and generated fake images $G(z)$ [S1].
*   **Outputs:** The generator outputs a rendered image; the discriminator outputs a scalar score $D(\cdot) \in [0, 1]$ indicating the probability that the input is real [S1].

**Data Structures:**
*   **Discriminator Network ($D$):** A neural network (typically a CNN in image-based contexts, or adapted for volumetric features) that takes an image as input and outputs a single scalar value [S1].
*   **Generator Network ($G$):** The existing radiance field model (e.g., the super-resolution network described in the parent article) [S1].

**Formulas:**
The discriminator loss $L_{GAN,D}$ minimizes the error in distinguishing real from fake data [S1]:
$$
L_{GAN,D} = - \log(D(x)) - \log(1 - D(G(z)))
$$
where $x$ is the ground truth image and $G(z)$ is the generated image [S1].

The generator loss $L_{GAN,G}$ minimizes the discriminator's ability to detect fakes, effectively maximizing the probability that the discriminator classifies generated images as real [S1]:
$$
L_{GAN,G} = - \log(D(G(z)))
$$

**Parameters:**
*   $\lambda_G$: Weight for the adversarial generator loss in the total objective function [S1].
*   $\lambda_{M}, \lambda_{AO}, \lambda_{N}$, etc.: Weights for other auxiliary losses (e.g., L1 losses on masks, albedo, normals) which are summed with the GAN loss [S1].

**Steps:**
1.  **Forward Pass (Generator):** Render the image $G(z)$ using the radiance field network from input $z$ [S1].
2.  **Discriminator Update:**
    a. Compute $D(x)$ for ground truth images $x$ [S1].
    b. Compute $D(G(z))$ for generated images $G(z)$ [S1].
    c. Calculate $L_{GAN,D}$ [S1].
    d. Update discriminator weights to minimize $L_{GAN,D}$ [S1].
3.  **Generator Update:**
    a. Compute $D(G(z))$ using the current discriminator [S1].
    b. Calculate $L_{GAN,G}$ [S1].
    c. Update generator weights to minimize $L_{GAN,G}$ (often combined with other reconstruction losses) [S1].

**Edge Cases and Pitfalls:**
*   **Training Instability:** GAN training is known to be unstable; the discriminator may overpower the generator, leading to mode collapse or vanishing gradients [S1].
*   **Loss Balance:** The relative weights ($\lambda$) of the adversarial loss versus reconstruction losses (like L1) must be carefully tuned [S1].

## Subtopics

### [Lighting Prediction Network Training](19342-lighting-prediction-network-training.md)
A training strategy for predicting lighting conditions in street scenes that addresses depth rendering ambiguity and encourages photorealism through adversarial supervision. It distinguishes itself from standard differentiable rendering pipelines by combining a multi-term hybrid loss (including sky regression, reconstruction, and regularization) with a PatchGAN discriminator, trained in a two-stage process (pre-training followed by joint adversarial training) to stabilize convergence.

## Operating characteristics
*   **Complexity:** Adds the computational cost of the discriminator network to each training step. The discriminator typically requires fewer parameters than the generator but adds significant overhead to the backward pass if not carefully sized [S1].
*   **Performance:** Improves perceptual quality and high-frequency details compared to MSE/L1-only training [S1].
*   **Strengths:** Produces sharper, more realistic textures; reduces blurriness [S1].
*   **Limitations:** Harder to train; prone to instability; requires careful hyperparameter tuning for loss weights [S1].
*   **When to use:** When photorealism and high-frequency detail are prioritized over strict pixel-wise accuracy [S1].

## Background
Originated from Generative Adversarial Networks (GANs) introduced by Goodfellow et al. [S1]. In the context of volumetric rendering and super-resolution, it was employed to train a discriminator network in parallel with a super-resolution generator to classify images as ground truth or predicted, using the discriminator's output as a loss term for the generator [S1].

## Sources
[S1] Volumetric Isosurface Rendering with Deep Learning-Based Super ..., https://ieeexplore.ieee.org/abstract/document/8918030, fetched 2026-10-31
