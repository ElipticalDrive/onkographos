# ENeRF with Guided Sampling Training

Also called: ENeRF Depth-Guided Training, ENeRF Guided Sampling

## Overview
ENeRF with Guided Sampling Training is a training procedure that integrates pixel-region and depth-boundary-guided ray sampling into the Efficient Neural Radiance Field (ENeRF) framework [S1]. It solves the problem of high GPU memory consumption and inefficient sampling in generalizable NeRF models by using fewer rays per iteration while maintaining or improving rendering performance. It sets itself apart from standard ENeRF by actively guiding sampling locations using adapted monocular depth priors, thereby mitigating shape-radiance ambiguity more effectively than uniform or random sampling strategies [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A pre-trained ENeRF model (generalized from the DTU dataset), input images, and estimated depth maps for the input views [S1].
*   **Outputs:** Optimized radiance field parameters and rendered images for novel views [S1].

**Data Structures:**
*   **Ray Sampling Set:** A subset of camera rays selected based on pixel region importance and depth boundary locations [S1].
*   **Sampling Points:** Points along the selected rays where the neural network is evaluated. The default configuration uses 8 points for the coarse stage and 2 points for the fine stage in a 2-stage NeRF setup [S1, S2].

**Training Steps:**
1.  **Initialization:** Load the ENeRF model pre-trained on the DTU dataset [S1].
2.  **Ray Selection:** Sample a reduced number of rays ($N_r$) per iteration. The selection is guided by:
    *   **Pixel Region:** Prioritizing regions with high gradient or error [S1].
    *   **Depth Boundary:** Prioritizing rays passing through depth discontinuities derived from monocular depth estimates [S1].
3.  **Point Sampling:** For each selected ray, sample points along the ray. The number of points is typically 8 for the coarse network and 2 for the fine network [S1, S2].
4.  **Rendering:** Render the color $\hat{C}_i$ for each ray using volume rendering [S1].
5.  **Loss Calculation:** Compute the total loss $\mathcal{L}$ using the following components:
    *   **MSE Loss:**
        $$ \mathcal{L}_{mse} = \frac{1}{N_r} \sum_{i=1}^{N_r} \| \hat{C}_i - C_i \|^2 $$
        where $N_r$ is the number of sampled rays, $\hat{C}_i$ is the rendered color, and $C_i$ is the ground-truth color [S1].
    *   **Perceptual Loss:**
        $$ \mathcal{L}_{perc} = \frac{1}{N_i} \sum_{i=1}^{N_i} \| \Phi(\hat{I}_i) - \Phi(I_i) \| $$
        where $N_i$ is the number of image blocks, $\Phi$ is the perceptual function defined in the original ENeRF paper, $\hat{I}_i$ is the rendered image block, and $I_i$ is the ground-truth image block [S1].
    *   **Total Loss:**
        $$ \mathcal{L} = \mathcal{L}_{mse} + \lambda' \mathcal{L}_{perc} $$
        where $\lambda'$ is the weight for perceptual loss [S1].
6.  **Optimization:** Update the ENeRF model parameters using gradient descent on $\mathcal{L}$ [S1].
7.  **Fine-tuning:** After pre-training on DTU, fine-tune the model on the target novel scenes for a few epochs [S1].

**Parameters:**
*   $\lambda'$: Weight for perceptual loss. Default value: 0.01 [S1].
*   Sampling Points: 8 (coarse) and 2 (fine) for standard settings; 16 (coarse) and 4 (fine) for large-baseline adaptations [S1, S2].

**Edge Cases and Pitfalls:**
*   **Depth Estimation Quality:** In large baseline settings with sparse view overlap, monocular depth estimation quality degrades, which negatively affects the NeRF rendering process [S2].
*   **Memory Usage:** While the method reduces GPU memory usage compared to standard ENeRF by sampling fewer rays, insufficient sampling points in low-quality depth scenarios can lead to poor results [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_enefr_guided_sampling(model, images, depths, num_epochs, lambda_prime=0.01):
    """
    Trains ENeRF with guided sampling.
    
    Args:
        model: Pre-trained ENeRF model.
        images: List of input images.
        depths: List of estimated depth maps.
        num_epochs: Number of fine-tuning epochs.
        lambda_prime: Weight for perceptual loss.
    """
    optimizer = Adam(model.parameters())
    
    for epoch in range(num_epochs):
        for image, depth in zip(images, depths):
            # 1. Guided Ray Sampling
            # Select fewer rays based on pixel region and depth boundary
            rays = guided_ray_sampling(image, depth)
            N_r = len(rays)
            
            # 2. Forward Pass
            # Sample points along rays (8 coarse, 2 fine)
            # Render colors
            rendered_colors = []
            for ray in rays:
                points = sample_points_along_ray(ray, num_coarse=8, num_fine=2)
                color = model.render(points)
                rendered_colors.append(color)
            
            rendered_colors = torch.stack(rendered_colors)
            
            # 3. Loss Calculation
            # MSE Loss
            gt_colors = get_gt_colors(rays, image)
            loss_mse = torch.mean((rendered_colors - gt_colors) ** 2)
            
            # Perceptual Loss
            # Render full images from rays or sample blocks
            rendered_images = render_images_from_rays(model, rays)
            gt_images = crop_images_from_gt(image, rays)
            
            # Apply perceptual function Phi (e.g., VGG features)
            loss_perc = torch.mean(torch.abs(Phi(rendered_images) - Phi(gt_images)))
            
            # Total Loss
            loss = loss_mse + lambda_prime * loss_perc
            
            # 4. Backward Pass
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
```

## Subtopics

### ENeRF Large-Baseline Adaptation
An enhanced ENeRF baseline configuration designed to tolerate depth estimation errors caused by sparse view overlap in large baseline settings [S2]. It sets itself apart from the standard ENeRF guided sampling by doubling the number of sampling points on each ray from 8 and 2 to 16 and 4 for 2-stage NeRFs [S2]. Choose this variant when input views have small overlap, leading to lower depth estimation quality that negatively affects the rendering process [S2].

## Operating characteristics
*   **Time and Memory Complexity:** The method reduces GPU memory usage compared to standard ENeRF by sampling fewer rays per iteration [S1]. Time complexity is reduced per iteration due to fewer rays, but may require more iterations if depth guidance is noisy [S1].
*   **Measured Performance:** Achieves improved performance on the DTU dataset (pre-training) and fine-tuned on novel scenes [S1]. Tested on Real Forward-facing, NeRF Synthetic, ZJU-MoCap, and digital human datasets [S1].
*   **Scaling:** Performance degrades in large baseline settings if sampling density is not increased [S2].
*   **Strengths:** Lower GPU memory footprint, improved performance with fewer rays, effective generalization to novel scenes [S1].
*   **Limitations:** Relies on the quality of monocular depth estimates; performance drops if depth estimation is poor (e.g., large baselines) [S1, S2].
*   **When to Use:** When GPU memory is constrained and high-quality monocular depth estimates are available.
*   **When Not to Use:** In large baseline settings with sparse view overlap unless sampling density is increased [S2].

## Background
ENeRF with Guided Sampling Training was introduced in the context of efficient radiance field reconstruction [S1]. It builds on the ENeRF framework, which focuses on rapid generalization for novel scenes [S1]. The method was developed to address the inefficiency of standard ray sampling in NeRFs, particularly in generalizable settings [S1]. It has been used in experiments on the DTU dataset for pre-training and fine-tuned on various other datasets including Real Forward-facing, NeRF Synthetic, ZJU-MoCap, and digital human datasets [S1]. The large-baseline adaptation was proposed in the MuRF paper to handle sparse view scenarios [S2].

## Sources
[S1] Efficient ray sampling for radiance fields reconstruction, https://doi.org/10.1016/j.cag.2023.11.005, fetched 2026-10-27
[S2] MuRF: Multi-Baseline Radiance Fields, https://doi.org/10.1109/cvpr52733.2024.01894, fetched 2026-10-27
