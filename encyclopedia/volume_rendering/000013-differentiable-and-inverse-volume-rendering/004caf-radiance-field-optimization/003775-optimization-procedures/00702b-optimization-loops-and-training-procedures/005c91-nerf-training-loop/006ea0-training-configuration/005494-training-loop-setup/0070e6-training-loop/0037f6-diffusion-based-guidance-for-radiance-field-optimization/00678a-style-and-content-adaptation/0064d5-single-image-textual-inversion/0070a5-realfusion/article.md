# RealFusion

Also called: RealFusion optimization, RealFusion training procedure, single-image 3D reconstruction via diffusion prior

## Overview
RealFusion reconstructs a full 360° photographic 3D model of an arbitrary object from a single input image without 3D supervision or object category assumptions [S3]. It solves the ill-posed nature of single-view reconstruction by combining a NeRF photometric reconstruction loss with a Score Distillation Sampling (SDS) prior derived from a pre-trained 2D diffusion model [S3]. Unlike DreamFusion, which relies on generic text prompts and suffers from mode collapse under high guidance weights, RealFusion uses single-image textual inversion to generate a specific prompt embedding that conditions the diffusion model to "dream up" plausible novel views of the exact input object [S3]. It builds on [Single-Image Textual Inversion](19221-single-image-textual-inversion.md) to substitute for missing multi-view data [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   A single input image $I_0$ of an object [S3].
*   An off-the-shelf image matting model to produce an object mask $M$ for $I_0$ [S3].
*   A pre-trained 2D diffusion model $\Phi$ (specifically Stable Diffusion trained on LAION) [S3].
*   A base radiance field architecture (InstantNGP) [S3].

**Outputs:**
*   A trained neural radiance field (NeRF) representing the 3D geometry and appearance of the object [S3].

### Data Structures
The radiance field is implemented using InstantNGP [S3].
*   **Feature Grids:** A set of feature grids $\{G_i\}_{i=1}^L$ at multiple resolutions [S3].
*   **Resolution:** Resolutions are chosen as a geometric progression between the coarsest and finest levels [S3].
*   **Configuration:** The implementation uses $L=16$ resolution levels, a feature dimension of 2, and a maximum resolution of 2048 [S3].
*   **Training State:** Feature grids are trained simultaneously, but optimization is restricted to lower-resolution grids during the first half of training (coarse-to-fine) [S3].

### Steps

#### 1. Single-Image Textual Inversion (Prompt Engineering)
Before 3D optimization, a specific text prompt is engineered to condition the diffusion model [S3].
1.  Generate random augmentations $g(I_0)$ of the input image $I_0$, where $g \in \mathcal{G}$ [S3].
2.  Treat these augmentations as a mini-dataset $\mathcal{D}_0 = \{g(I_0)\}$ [S3].
3.  Introduce a new token `hei` into the vocabulary of the diffusion model's text encoder [S3].
4.  Define the prompt template as "An image of a hei" [S3].
5.  Optimize the embedding $e(I_0)$ corresponding to `hei` by minimizing the diffusion loss $L_{diff}(\Phi(\cdot; e(I_0)))$ with respect to $e(I_0)$, while freezing all other text embeddings and model parameters [S3].
    *   *Note:* This step substitutes for alternative views by capturing object-specific details in the embedding [S1].

#### 2. Radiance Field Optimization
The radiance field parameters $\sigma$ (density) and $c$ (color) are optimized using a combined loss function [S3]. The training proceeds in a coarse-to-fine manner [S3].

**Camera Setup:**
*   **Reconstruction Camera:** Fixed at a position looking at the origin on a sphere of radius 1.8, at an angle of 15° above the plane [S3].
*   **Novel View Camera:** Randomly sampled at each optimization step [S3].

**Loss Components:**
1.  **Reconstruction Loss ($L_{rec}$):** Minimizes L2 error between the input image and the rendered image from the fixed reconstruction camera [S3].
2.  **Mask Loss ($L_{rec,mask}$):** Constrains rendered opacities $O$ at the fixed viewpoint to match the object mask $M$ [S3].
    $$ L_{rec,mask} = ||O - M||^2 $$
    where $O = R(\cdot; \sigma, c, \pi_0) \in \mathbb{R}^{H \times W}$ is the rendered opacity from the fixed reference viewpoint $\pi_0$ [S3].
3.  **SDS Prior Loss ($L_{SDS}$):** Applied to the randomly sampled novel view to enforce 3D consistency using the diffusion prior [S3].
    $$ L_{SDS}(\sigma, c; \pi, e, t) = \mathbb{E}_{t,\epsilon} \left[ w(t) (\Phi(\alpha_t I + \sigma_t \epsilon; t, e) - \epsilon) \cdot \nabla_{(\sigma,c)} I \right] $$
    where $I = R(\cdot; \sigma, c, \pi)$ is the image rendered from the novel viewpoint $\pi$, $e$ is the optimized prompt embedding, $t$ is the noise level, $\epsilon$ is the predicted noise, and $w(t)$ is a weighting function [S3].
4.  **Normal Regularization ($L_{normals}$):** Encourages smooth surface normals by comparing rendered normals to a blurred version of themselves [S3].
    $$ L_{normals} = ||N - \text{stopgrad}(\text{blur}(N, k))||^2 $$
    where $N \in \mathbb{R}^{H \times W \times 3}$ are the aggregated normals along the ray, $\text{blur}(\cdot, k)$ is a Gaussian blur with kernel size $k$, and $\text{stopgrad}$ stops gradient flow through the blur operation [S3].

**Total Loss:**
$$ \nabla_{\sigma,c} L = \nabla L_{SDS} + \lambda_{normals} \cdot \nabla L_{normals} + \lambda_{image} \cdot \nabla L_{image} + \lambda_{mask} \cdot \nabla L_{mask} $$
where $L_{image}$ corresponds to $L_{rec}$ [S3].

**Coarse-to-Fine Schedule:**
*   **First Half:** Optimize only lower-resolution feature grids $\{G_i\}_{i=1}^{L/2}$ [S3].
*   **Second Half:** Optimize all feature grids $\{G_i\}_{i=1}^L$ [S3].

**Shading Strategy:**
*   **Warmup:** Initial period uses albedo-only shading [S3].
*   **Main Training:** Stochastically applies diffuse and textureless shading throughout optimization [S3].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def realfusion_train(I0, mask_model, diffusion_model, instantngp_model, total_steps=10000):
    # 1. Single-Image Textual Inversion
    # Generate random augmentations of the input image
    augmentations = [apply_random_augmentation(I0) for _ in range(10)]
    mini_dataset = augmentations

    # Initialize new token embedding 'hei'
    hei_embedding = np.random.randn(diffusion_model.text_encoder.dim)

    # Optimize embedding to match input image via diffusion loss
    # Freeze diffusion model parameters and other text embeddings
    lr_ti = 1e-3
    for step in range(1000):
        loss_diff = 0.0
        for img_aug in mini_dataset:
            t = np.random.randint(0, diffusion_model.num_timesteps)
            eps = np.random.randn(*img_aug.shape)
            noisy_img = diffusion_model.alphas[t]**0.5 * img_aug + (1 - diffusion_model.alphas[t])**0.5 * eps
            pred_eps = diffusion_model(noisy_img, t, prompt_template="An image of a <e>", embedding=hei_embedding)
            loss_diff += np.mean((pred_eps - eps)**2)
        hei_embedding -= lr_ti * np.gradient(loss_diff, hei_embedding)

    # Construct prompt
    prompt = "An image of a hei"

    # 2. Radiance Field Optimization
    # Setup cameras
    reconstruction_camera = {"position": (0, 0, 1.8), "angle_deg": 15.0}
    novel_view_camera = {"position": sample_random_sphere_point(), "angle_deg": 15.0}

    # Hyperparameters
    lambda_normals = 0.5
    lambda_image = 5.0
    lambda_mask = 0.5
    k = 9  # Gaussian blur kernel size
    half_steps = total_steps // 2
    warmup_steps = 1000

    for step in range(total_steps):
        # Coarse-to-fine grid selection
        if step < half_steps:
            active_grids = instantngp_model.grids[:len(instantngp_model.grids)//2]
        else:
            active_grids = instantngp_model.grids

        # Render from fixed reconstruction camera
        render_rec = instantngp_model.render(reconstruction_camera, active_grids)
        rendered_image_rec = render_rec["rgb"]
        rendered_opacity_rec = render_rec["opacity"]

        # Compute Reconstruction Losses
        # L_rec: L2 error between input image and rendered image
        loss_image = np.mean((rendered_image_rec - I0)**2)

        # L_rec,mask: L2 error between rendered opacity and object mask
        mask = mask_model(I0)
        loss_mask = np.mean((rendered_opacity_rec - mask)**2)

        # Render from random novel view camera
        render_novel = instantngp_model.render(novel_view_camera, active_grids)
        rendered_image_novel = render_novel["rgb"]
        rendered_normals_novel = render_novel["normals"]

        # Compute SDS Loss on novel view
        # Sample noise level t and noise epsilon
        t = np.random.randint(0, diffusion_model.num_timesteps)
        epsilon = np.random.randn(*rendered_image_novel.shape)

        # Add noise to rendered novel view
        alpha_t = diffusion_model.alphas[t]**0.5
        sigma_t = (1 - diffusion_model.alphas[t])**0.5
        noisy_image = alpha_t * rendered_image_novel + sigma_t * epsilon

        # Get predicted noise from diffusion model
        predicted_noise = diffusion_model(noisy_image, t, prompt)

        # Compute SDS gradient estimate
        # w(t) is a weighting function, typically dependent on t
        w_t = compute_weighting(t)
        grad_sds = w_t * (predicted_noise - epsilon) * np.gradient(rendered_image_novel, instantngp_model.params)
        # Apply gradient to radiance field
        instantngp_model.apply_gradient(grad_sds)

        # Compute Normal Regularization Loss
        # Blur normals and stop gradient
        blurred_normals = gaussian_blur(rendered_normals_novel, k)
        stopped_blurred_normals = np.stop_gradient(blurred_normals)
        loss_normals = np.mean((rendered_normals_novel - stopped_blurred_normals)**2)
        # Apply gradient for normal regularization
        grad_normals = np.gradient(loss_normals, instantngp_model.params)
        instantngp_model.apply_gradient(lambda_normals * grad_normals)

        # Compute gradients for reconstruction losses
        grad_image = np.gradient(loss_image, instantngp_model.params)
        grad_mask = np.gradient(loss_mask, instantngp_model.params)
        instantngp_model.apply_gradient(lambda_image * grad_image + lambda_mask * grad_mask)

        # Stochastic Shading
        if step > warmup_steps:
            apply_stochastic_shading(instantngp_model)
```

### Parameters
*   **$\lambda_{image}$:** 5.0 [S3].
*   **$\lambda_{mask}$:** 0.5 [S3].
*   **$\lambda_{normals}$:** 0.5 [S3].
*   **$k$ (blur kernel):** 9 [S3].
*   **Diffusion Model:** Stable Diffusion trained on LAION [S3].
*   **InstantNGP Config:** 16 resolution levels, feature dimension 2, max resolution 2048 [S3].
*   **Camera Radius:** 1.8 [S3].
*   **Camera Angle:** 15° above the plane [S3].

### Edge Cases and Pitfalls
*   **Over-fitting:** Optimizing only with the reconstruction loss on a single image leads to zero loss but no sensible 3D model [S3]. Solved by the SDS prior [S3].
*   **Limited Diversity (Mode Collapse):** High classifier-free guidance weights (e.g., 100) in DreamFusion produce only generic objects. RealFusion solves this by using the specific `hei` embedding from textual inversion, allowing reconstruction of any specific object [S3].
*   **Surface Irregularities:** InstantNGP optimization can produce small surface irregularities. Solved by coarse-to-fine training [S3].
*   **Noisy Surfaces:** Low-level artifacts can appear. Solved by 2D normal vector regularization [S3].
*   **Janus Problem:** The model may copy the front view to the back of the object. This is a known failure mode [S3].
*   **Convergence Failure:** The model may produce a semi-transparent scene lacking well-defined geometry. This is a known failure mode [S3].

## Operating characteristics
*   **Time Complexity:** Training takes hours rather than days, due to the use of InstantNGP [S3].
*   **Memory Complexity:** Depends on the InstantNGP grid configuration (16 levels, dim 2, max res 2048) [S3].
*   **Strengths:**
    *   Reconstructs arbitrary objects from a single image without category assumptions [S3].
    *   Produces full 360° photographic models [S3].
    *   Significantly higher quality than CLIP priors [S3].
    *   2D normal regularization reduces variance and yields superior results compared to 3D regularization [S3].
*   **Limitations:**
    *   Susceptible to the Janus problem (front-back texture copying) [S3].
    *   Can fail to converge, resulting in semi-transparent scenes [S3].
    *   Requires an off-the-shelf matting model for the mask [S3].
*   **When to use:** Single-image 3D reconstruction of arbitrary objects in the wild or on benchmark datasets [S3].
*   **When not to use:** When multi-view images are available (other methods may be more robust) or when strict geometric accuracy without texture artifacts is required (due to Janus/convergence risks) [S3].

## Background
RealFusion was introduced in February 2023 [S1]. It builds upon the DreamFusion method [S3], which generates 3D models from text descriptions. RealFusion adapts this by conditioning on a single image using textual inversion [S1]. It uses Stable Diffusion as the prior and InstantNGP as the radiance field backbone [S1]. The method was evaluated on the CO3D dataset against Shelf-Supervised Mesh Prediction [S3].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
[S3] RealFusion 360° Reconstruction of Any Object from a Single Image, https://doi.org/10.1109/cvpr52729.2023.00816, fetched 2026-10-30
