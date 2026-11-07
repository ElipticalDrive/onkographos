# Latent-NeRF

Also called: Latent Diffusion NeRF

## Overview
Latent-NeRF is a method for optimizing Neural Radiance Fields (NeRF) by integrating pre-trained latent diffusion models, enabling high-quality novel view synthesis and text-guided editing without requiring dense 3D supervision [S1]. It distinguishes itself from standard NeRF training by operating in a compact latent space rather than RGB color space, using score distillation from a stable diffusion model to guide the optimization of volume density and latent features [S1].

## How it works
**Inputs:**
*   A pre-trained autoencoder (encoder $\mathcal{E}$ and decoder $\mathcal{D}$) that maps between pixel space and a compact latent space.
*   A pre-trained stable diffusion model (UNet) for image denoising.
*   Text prompts and/or shape guidance for conditional generation.
*   Camera parameters for ray generation.

**Outputs:**
*   A NeRF representation (MLP weights) that outputs volume density $\sigma$ and four pseudo-color channels in the latent space.
*   Rendered images in the latent space, which are decoded to pixel space.

**Data Structures:**
*   **NeRF MLP:** Takes 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ as input.
    *   Output: Volume density $\sigma$ (scalar) and a latent feature vector $\mathbf{z}_{feat} \in \mathbb{R}^4$. The four channels correspond to the latent features used by the diffusion model [S1].
*   **Autoencoder:**
    *   Encoder $\mathcal{E}$: Maps real images to latent space.
    *   Decoder $\mathcal{D}$: Maps latent features back to image space.

**Algorithm Steps:**
1.  **Ray Sampling:** For a given camera pose, generate rays and sample points $\mathbf{x}_i$ along each ray.
2.  **NeRF Forward Pass:** Query the NeRF MLP at each point $\mathbf{x}_i$ with direction $\mathbf{d}$ to obtain density $\sigma_i$ and latent feature $\mathbf{z}_i \in \mathbb{R}^4$.
3.  **Volume Rendering in Latent Space:** Compute the rendered latent feature map $\mathbf{Z}$ for the image using the standard volume rendering integral, but replacing color $\mathbf{c}$ with the latent feature $\mathbf{z}$:
    $$
    \mathbf{Z}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{z}_i
    $$
    where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$.
4.  **Noise Addition:** Add Gaussian noise $\epsilon$ to the rendered latent feature map $\mathbf{Z}$ to create a noisy latent representation $\mathbf{Z}_t$ at timestep $t$:
    $$
    \mathbf{Z}_t = \mathbf{Z} + \epsilon, \quad \epsilon \sim \mathcal{N}(0, \mathbf{I})
    $$
5.  **Diffusion Denoising:** Pass $\mathbf{Z}_t$ into the pre-trained stable diffusion UNet along with the timestep $t$ and conditioning text/shape embeddings. The UNet predicts the noise $\hat{\epsilon}$ or the clean latent $\hat{\mathbf{Z}}_0$.
6.  **Score Distillation Loss:** Compute the loss by comparing the predicted noise from the diffusion model with the actual added noise (or using the predicted clean latent to compute a reconstruction loss in latent space). This loss is backpropagated through the diffusion model (frozen) to the NeRF parameters [S1].
7.  **Optimization:** Update the NeRF MLP weights using gradient descent to minimize the diffusion loss.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def latent_nerf_training_step(nerf_mlp, autoencoder, diffusion_model, camera_pose, text_cond):
    # 1. Generate rays and sample points
    rays = generate_rays(camera_pose)
    points, directions = sample_points_along_rays(rays, num_samples=128)
    
    # 2. Query NeRF for density and latent features
    # nerf_mlp outputs (sigma, z_feat) where z_feat is shape [N, 4]
    sigma, z_feat = nerf_mlp(points, directions)
    
    # 3. Volume rendering in latent space
    # Compute transmittance and weights
    deltas = compute_deltas(points)
    alpha = 1.0 - torch.exp(-sigma * deltas)
    T = torch.cumprod(torch.exp(-sigma * deltas), dim=0)
    T = torch.cat([torch.ones_like(T[:, :1]), T[:, :-1]], dim=1)
    
    # Render latent feature map Z
    Z = torch.sum(T * alpha * z_feat, dim=0) # Shape: [4, H, W]
    
    # 4. Add noise to latent feature map
    t = random.randint(0, diffusion_model.num_timesteps)
    noise = torch.randn_like(Z)
    Z_noisy = add_noise_to_latent(Z, t, noise, diffusion_model.alphas_cumprod)
    
    # 5. Diffusion model prediction
    # Condition on text and shape if available
    pred_noise = diffusion_model.unet(Z_noisy, t, text_cond).sample
    
    # 6. Compute Loss (Score Distillation)
    # The loss is typically the MSE between predicted noise and actual noise
    loss = torch.mean((pred_noise - noise) ** 2)
    
    # 7. Backpropagate to NeRF
    loss.backward()
    optimizer.step()
    
    return loss
```

**Parameters:**
*   Number of latent channels: 4 [S1].
*   Diffusion model: Stable Diffusion [S1].
*   Guidance: Text and/or shape [S1].

**Edge Cases:**
*   The method relies on the compatibility between the NeRF's output latent space and the diffusion model's input latent space. The autoencoder must be pre-trained on data similar to the target domain [S1].
*   Precision issues may arise if the latent features are not properly normalized before being passed to the diffusion model.

## Subtopics

### [Points2NeRF](9389-points2nerf.md)
Points2NeRF is a method for generating Neural Radiance Field (NeRF) representations directly from 3D point clouds using a hypernetwork-based auto-encoder architecture. It solves the problem of reconstructing high-quality radiance fields from sparse 3D input data by learning a mapping from point clouds to NeRF network weights, rather than optimizing a NeRF for each object individually. Unlike standard NeRF training which optimizes per-object parameters, Points2NeRF trains a single hypernetwork to produce weights for any input point cloud, enabling the generation of various NeRF functions at test time.

## Operating characteristics
*   **Time Complexity:** Training is slower than standard NeRF due to the iterative denoising process and the need to backpropagate through the diffusion model (or use score distillation approximations).
*   **Memory Complexity:** Requires storing the pre-trained diffusion model and autoencoder in memory alongside the NeRF.
*   **Strengths:** Enables text-guided editing and generation of NeRFs from single or sparse views without explicit 3D supervision [S1]. Produces high-quality textures and details by leveraging the prior knowledge of the diffusion model [S1].
*   **Limitations:** Computationally expensive. Requires a pre-trained diffusion model and autoencoder. Quality depends on the alignment between the NeRF's latent representation and the diffusion model's latent space.
*   **When to use:** When text-guided control or generation from sparse data is required, and computational resources are sufficient for diffusion-based optimization.
*   **When not to use:** When fast training is required or when dense multi-view images are available (standard NeRF or Instant-NGP may be more efficient).

## Background
Latent-NeRF was introduced as an extension of NeRF to improve image rendering quality by using pre-trained models [S1]. It applies score distillation to a latent diffusion model and the compact latent space of a pre-trained autoencoder [S1]. It is noted as the first approach that allows coloring of the grid under the guidance of a pre-trained diffusion model, achieving impressive results [S1]. The method is part of a broader trend of integrating generative models (like GANs and Diffusion Models) with NeRFs to enhance synthesis capabilities [S1].

## Sources
[S1] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
