# Masked Completion via GAN Inversion

Also called: GAN Inversion for 3D Completion

## Overview
Masked Completion via GAN Inversion is a conditional sampling technique that completes a 3D radiance field by inverting a 2D discriminator-trained generator, using photometric error on non-masked regions to guide the optimization [S1]. It sets itself apart from sibling methods like [Nerfiller](14835-nerfiller.md) and [Reference-guided Controllable Inpainting of Neural Radiance Fields](15289-reference-guided-controllable-inpainting-of-neural-radiance-.md) by operating in the latent space of a pre-trained GAN generator (specifically EG3D) rather than optimizing a neural network's weights or using diffusion-based volumetric gradients directly [S1].

## How it works
**Inputs:**
*   A pre-trained 3D-aware GAN generator (e.g., EG3D) trained with 2D discriminator guidance [S1].
*   A target image with an associated 3D input mask indicating the region to be completed [S1].
*   A random initial latent code $w_{init}$ [S1].

**Outputs:**
*   An optimized latent code $w^*$ that, when passed through the generator, produces a 3D radiance field consistent with the non-masked regions of the input image [S1].

**Data Structures:**
*   **Latent Code ($w$):** The optimization variable. The specific dimensionality depends on the underlying GAN architecture (e.g., StyleGAN2/EG3D intermediate latent space) [S1].
*   **3D Input Mask:** A binary mask defining the region of interest in the 3D scene, projected onto 2D views [S1].

**Parameters:**
*   **Iterations:** 200 iterations per masked sample [S1].
*   **Views per Iteration:** 8 views [S1].
*   **Optimizer:** Adam [S1].
*   **Learning Rate:** $10^{-2}$ [S1].
*   **L2 Regularization Weight:** $5 \times 10^{-2}$ applied to the latent code [S1].
*   **Resolution:** Rendering is performed at $128 \times 128$ resolution [S1].

**Steps:**
1.  **Initialization:** Start from a random initial latent code $w$ [S1].
2.  **Iteration Loop:** Repeat for 200 iterations:
    a.  **Rendering:** Render the current synthesized sample from 8 different views using the current latent code $w$ [S1].
    b.  **Mask Projection:** Project the 3D input mask onto the synthesized views using the predicted depths from the rendering [S1].
    c.  **Error Computation:** Compute the photometric error between the rendered non-masked regions and the corresponding regions in the input images [S1].
    d.  **Optimization:** Update the latent code $w$ using the Adam optimizer with a learning rate of $10^{-2}$ and an L2 regularization term on the code weighted by $5 \times 10^{-2}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def masked_completion_via_gan_inversion(generator, target_image, input_mask_3d, num_iterations=200, num_views=8):
    """
    Performs 3D masked completion by inverting a GAN generator.
    
    Args:
        generator: Pre-trained 3D-aware GAN generator (e.g., EG3D).
        target_image: The input image with the region to be completed.
        input_mask_3d: 3D input mask defining the completion region.
        num_iterations: Number of optimization iterations (default 200).
        num_views: Number of views to render per iteration (default 8).
        
    Returns:
        w_opt: Optimized latent code.
    """
    # Initialize with a random latent code
    w = random_latent_code(generator)
    
    # Optimizer setup
    optimizer = Adam(params=[w], lr=1e-2)
    l2_weight = 5e-2
    
    for i in range(num_iterations):
        optimizer.zero_grad()
        
        # Render the current synthesized sample from 8 views
        rendered_images = []
        predicted_depths = []
        for view_idx in range(num_views):
            # Render using the generator and current latent code w
            # This returns an image and its associated depth map
            img, depth = generator.render(w, view=view_idx)
            rendered_images.append(img)
            predicted_depths.append(depth)
            
        # Project the 3D input mask onto the synthesized views using predicted depths
        # This creates 2D masks for each view indicating which pixels are in the 'unknown' region
        view_masks = project_3d_mask_to_views(input_mask_3d, predicted_depths, camera_poses)
        
        # Compute photometric error on non-masked regions
        total_error = 0.0
        for j in range(num_views):
            rendered_img = rendered_images[j]
            # Get the non-masked region of the target image for this view
            # Note: The target image must be available for all 8 views or projected back
            # Assuming target_images is a list of images for the 8 views
            target_img = target_images[j]
            
            # Create inverse mask for non-masked regions
            non_masked_region = 1.0 - view_masks[j]
            
            # Compute photometric error only on non-masked regions
            # Using Mean Squared Error (MSE) as a standard photometric loss
            error = torch.mean((rendered_img * non_masked_region - target_img * non_masked_region) ** 2)
            total_error += error
            
        # Add L2 regularization on the latent code
        l2_loss = l2_weight * torch.sum(w ** 2)
        loss = total_error + l2_loss
        
        # Backpropagate and update
        loss.backward()
        optimizer.step()
        
    return w
```

**Edge Cases and Pitfalls:**
*   **Depth Prediction Accuracy:** The method relies on predicted depths to project the 3D mask. Inaccurate depth predictions can lead to incorrect mask projection, causing the photometric error to be computed on the wrong regions [S1].
*   **Generator Limitations:** Since the method inverts a 2D discriminator-trained generator, the quality of the completion is bounded by the generator's ability to represent the object class and the consistency of the 3D representation [S1].
*   **Local Minima:** Optimization in latent space can get stuck in local minima, especially with complex shapes or insufficient views [S1].

## Operating characteristics
*   **Time Complexity:** The method requires 200 iterations per sample, with each iteration involving rendering from 8 views [S1].
*   **Memory Complexity:** Memory usage is dominated by the storage of the latent code and the intermediate rendered images and depths for the 8 views per iteration [S1].
*   **Measured Performance:** On an NVIDIA RTX 2080 TI, the method achieves a run time of 48.6 seconds per sample for 1000 iterations (extrapolated from the 200-iteration description, implying ~9.7s for 200 iterations) [S1].
*   **Rendering Speed:** Once synthesized, the explicit representation enables rendering at $128 \times 128$ resolution with over 380 FPS [S1].
*   **Strengths:** Leverages the strong generative priors of pre-trained GANs, allowing for plausible completions even with limited input data [S1].
*   **Limitations:** Dependent on the quality and diversity of the pre-trained GAN generator; may struggle with object classes not well-represented in the training data [S1].

## Background
Masked Completion via GAN Inversion is described in the context of DiffRF, which uses a rendering-guided 3D radiance field diffusion model [S1]. The specific technique of using GAN inversion for 3D masked completion is attributed to the use of the EG3D generator, which is trained via 2D discriminator guidance [S1]. This approach allows for conditional sampling in 3D by optimizing the latent code to match 2D image observations [S1].

## Sources
[S1] DiffRF: Rendering-Guided 3D Radiance Field Diffusion, https://doi.org/10.1109/cvpr52729.2023.00421, fetched 2026-10-26
