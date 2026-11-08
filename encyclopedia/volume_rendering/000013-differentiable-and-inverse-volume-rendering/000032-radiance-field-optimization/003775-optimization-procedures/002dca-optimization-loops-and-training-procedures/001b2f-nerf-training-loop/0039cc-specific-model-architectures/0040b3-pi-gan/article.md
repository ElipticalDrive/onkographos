# pi-GAN

Also called: Periodic Implicit Generative Adversarial Networks, π-GAN

## Overview
pi-GAN is a generative adversarial approach for unsupervised 3D representation learning from 2D images, synthesizing view-consistent images by conditioning an implicit radiance field on a noise vector [S1, unofficial]. It differs from standard 2D GANs by outputting a 5D spatio-angular radiance field rendered via volume rendering, and from other implicit GANs (like GRAF) by using a StyleGAN-inspired mapping network with Feature-wise Linear Modulation (FiLM) for conditioning and a progressive growing strategy to stabilize training [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A latent noise vector $\mathbf{z} \sim p_z$ and a camera pose $\boldsymbol{\xi}$ (defined by origin $\mathbf{o}$ and viewing direction $\mathbf{d}$).
*   **Output:** A 2D image $\mathbf{I}$ synthesized from the camera pose $\boldsymbol{\xi}$.

**Data Structures:**
The core data structure is the implicit radiance field parameterized by a SIREN (Sinusoidal Representation Network) backbone $\Phi$ and a mapping network $M$.
*   **Mapping Network ($M$):** A ReLU MLP. Input: $\mathbf{z}$. Output: A set of frequency parameters $\{\gamma_i\}$ and phase shift parameters $\{\beta_i\}$ for each layer $i$ of the SIREN.
*   **SIREN Backbone ($\Phi$):** An MLP with $n$ layers. Each layer $i$ has weights $\mathbf{W}_i \in \mathbb{R}^{N_i \times M_i}$ and biases $\mathbf{b}_i \in \mathbb{R}^{N_i}$.
*   **Density Head:** Linear layer with weights $\mathbf{W}_\sigma$ and bias $\mathbf{b}_\sigma$.
*   **Color Head:** Linear layer with weights $\mathbf{W}_c$ and bias $\mathbf{b}_c$.

**Numbered Steps:**

1.  **Latent Mapping:** The noise vector $\mathbf{z}$ is passed through the mapping network $M$ to produce the conditioning parameters $\gamma_i$ and $\beta_i$ for all layers $i=0 \dots n-1$ [S1, unofficial].
2.  **SIREN Forward Pass:** For a query 3D coordinate $\mathbf{x} = (x, y, z)$, compute the SIREN output $\Phi(\mathbf{x})$ using the conditioned parameters:
    $$
    \Phi(\mathbf{x}) = \phi_{n-1} \circ \phi_{n-2} \circ \dots \circ \phi_0(\mathbf{x})
    $$
    where each layer function is defined as:
    $$
    \phi_i(\mathbf{x}_i) = \sin(\gamma_i \cdot (\mathbf{W}_i \mathbf{x}_i + \mathbf{b}_i) + \beta_i)
    $$
    Here, $\mathbf{x}_0 = \mathbf{x}$, and $\mathbf{x}_{i+1} = \phi_i(\mathbf{x}_i)$ [S1, unofficial].
3.  **Radiance Field Query:** Compute volume density $\sigma(\mathbf{x})$ and view-dependent color $\mathbf{c}(\mathbf{x}, \mathbf{d})$:
    $$
    \sigma(\mathbf{x}) = \mathbf{W}_\sigma \Phi(\mathbf{x}) + \mathbf{b}_\sigma
    $$
    $$
    \mathbf{c}(\mathbf{x}, \mathbf{d}) = \mathbf{W}_c \phi_c \left( [\Phi(\mathbf{x}), \mathbf{d}]^T \right) + \mathbf{b}_c
    $$
    where $[\Phi(\mathbf{x}), \mathbf{d}]^T$ denotes the concatenation of the SIREN output $\Phi(\mathbf{x})$ and the viewing direction $\mathbf{d}$, and $\phi_c$ is the activation function of the color head (typically ReLU or linear, as specified by the source's general MLP definition, though the source explicitly writes $\phi_c$ in Eq. 4) [S1, unofficial].
4.  **Volume Rendering:** Cast rays from the camera origin $\mathbf{o}$ with direction $\mathbf{d}$ defined by pose $\boldsymbol{\xi}$. Sample points $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ along the ray between near bound $t_n$ and far bound $t_f$. Compute pixel color $\mathbf{C}$ using the standard volume rendering integral (as defined in the parent article) [S1, unofficial].
5.  **Adversarial Training:** The generated image is fed to a traditional convolutional discriminator. The generator $G_\theta$ is trained to minimize the adversarial loss, while the discriminator is trained to distinguish real from generated images [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pi_gan_forward(z, camera_pose):
    """
    Generates an image from latent z and camera pose.
    """
    # 1. Mapping Network: Get conditioning parameters
    # M is a ReLU MLP
    gamma, beta = mapping_network(z) 
    
    # 2. SIREN Backbone: Compute feature representation Phi(x)
    # x is a 3D coordinate (batch of points)
    x = query_points(camera_pose) # Samples along rays
    phi = x
    for i in range(num_layers):
        # Affine transform
        affine = weights[i] @ phi + biases[i]
        # Apply conditioned sine activation
        # gamma[i] and beta[i] are scalars or vectors broadcasted appropriately
        phi = sin(gamma[i] * affine + beta[i])
        
    # 3. Density Head
    sigma = weights_sigma @ phi + biases_sigma
    
    # 4. Color Head
    # Concatenate Phi(x) and viewing direction d
    d = get_view_directions(camera_pose, x)
    concat_input = concatenate(phi, d, dim=-1)
    color = weights_c @ concat_input + biases_c
    
    # 5. Volume Rendering
    # Standard differentiable volume rendering using sigma and color
    image = volume_render(x, sigma, color, camera_pose)
    
    return image

def pi_gan_training_step(real_images, z, camera_pose):
    """
    Single training step for pi-GAN.
    """
    # Generate image
    fake_image = pi_gan_forward(z, camera_pose)
    
    # Discriminator Loss
    d_real = discriminator(real_images)
    d_fake = discriminator(fake_image.detach())
    d_loss = adversarial_loss(d_real, d_fake)
    
    # Generator Loss
    d_fake_gen = discriminator(fake_image)
    g_loss = adversarial_loss(d_fake_gen)
    
    # Backpropagate and update weights
    update(discriminator, d_loss)
    update(generator, g_loss)
```

**Parameters:**
*   $\gamma_i$: Frequency parameter for the sine activation in layer $i$. Output by the mapping network. [S1, unofficial]
*   $\beta_i$: Phase shift parameter for the sine activation in layer $i$. Output by the mapping network. [S1, unofficial]
*   $\mathbf{W}_i, \mathbf{b}_i$: Weights and biases of the SIREN layers. [S1, unofficial]
*   $\mathbf{W}_\sigma, \mathbf{b}_\sigma$: Weights and biases for the density head. [S1, unofficial]
*   $\mathbf{W}_c, \mathbf{b}_c$: Weights and biases for the color head. [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Local Overfitting:** SIREN is prone to local overfitting to the last batch if sufficient coverage of the space is not maintained. This is why pi-GAN does not employ a patch-based discriminator (unlike GRAF) [S1, unofficial].
*   **Conditioning:** Concatenation-based conditioning was found less expressive than FiLM conditioning, leading to lower image quality [S1, unofficial].

## Subtopics

### SIREN Backbone
Uses periodic sine activations to represent high-frequency details. Sets it apart from ReLU-based implicit representations. Choose when high-frequency detail recovery is needed. [S1, unofficial]

### FiLM Conditioning
Maps latent noise to layer-specific frequencies and phase shifts. Sets it apart from concatenation or additive conditioning. Choose for better expressivity in generative models. [S1, unofficial]

### Progressive Growing
Training strategy starting from low resolution and increasing. Sets it apart from fixed-resolution training. Choose to stabilize 3D GAN training and accelerate convergence. [S1, unofficial]

## Operating characteristics
*   **Time Complexity:** Training is computationally expensive due to the 3D nature of the representation and volume rendering. Progressive growing helps offset this complexity [S1, unofficial].
*   **Memory Complexity:** High, as it stores a large MLP for the radiance field and requires sampling points in 3D space for every training step [S1, unofficial].
*   **Performance:** Achieves state-of-the-art results on CelebA, Cats, and CARLA datasets for 3D-aware image synthesis [S1, unofficial].
*   **Strengths:** View-consistent image synthesis; interpretable 3D proxy shapes can be extracted via marching cubes [S1, unofficial].
*   **Limitations:** Proxy shapes may not be as high quality as those from explicit single-view reconstruction methods; prone to local overfitting if batch coverage is insufficient [S1, unofficial].
*   **When to use:** Unsupervised 3D-aware image generation from 2D images.
*   **When not to use:** When explicit high-quality 3D geometry is required without post-processing, or when computational resources are severely limited.

## Background
pi-GAN was introduced in 2021 by Tewari et al. as a novel generative model for high-quality 3D-aware image synthesis [S1, unofficial]. It builds on the success of SIREN (Sinusoidal Representation Networks) and StyleGAN, adapting them for 3D implicit representations. The method addresses the lack of photorealistic 3D training data and the view-inconsistency of traditional 2D GANs [S1, unofficial]. It is used for novel view synthesis and 3D-aware image generation [S1, unofficial].

## Sources
[S1] pi-GAN: Periodic Implicit Generative Adversarial Networks for 3D-Aware Image Synthesis, https://doi.org/10.1109/cvpr46437.2021.00574, fetched 2026-10-26 (unofficial)
