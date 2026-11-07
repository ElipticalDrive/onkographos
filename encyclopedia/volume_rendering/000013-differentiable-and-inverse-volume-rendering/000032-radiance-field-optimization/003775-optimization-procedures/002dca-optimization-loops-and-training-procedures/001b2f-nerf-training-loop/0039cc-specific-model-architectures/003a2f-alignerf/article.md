# AligNeRF

Also called: Alignment-Aware Neural Radiance Fields

## Overview
AligNeRF is a training strategy and architectural modification for Neural Radiance Fields (NeRF) designed to improve high-frequency detail recovery and robustness to misalignment caused by moving objects or camera calibration errors in high-resolution settings [S1]. It distinguishes itself from standard NeRF variants by combining a convolution-augmented architecture with a staged, alignment-aware fine-tuning procedure that utilizes patch-based sampling and optical flow estimation, rather than relying solely on random ray sampling [S1, S2].

## How it works
AligNeRF builds on [Specific Model Architectures](8585-specific-model-architectures.md) by modifying the baseline (specifically mip-NeRF 360) to support local neighborhood information and alignment correction.

### Inputs and Outputs
*   **Inputs:** 3D spatial coordinates $\mathbf{x}$, viewing directions $\mathbf{d}$, and estimated optical flow fields between source images [S1].
*   **Outputs:** Rendered color $\mathbf{c}$ and volume density $\sigma$ for each pixel, derived from patch-based rendering [S1].

### Data Structures
The method modifies the standard NeRF output layer and introduces a post-processing convolutional network.
1.  **Feature Space Output:** The final MLP layer outputs $N$ channels (feature space) instead of 3 (RGB) [S1].
2.  **Convolutional Network:** A 3-layer CNN with ReLU activations and $3 \times 3$ kernels processes the rendered feature maps [S1].
3.  **Padding:** "Reflectance" padding is used for CNN inputs to prevent checkerboard artifacts; padding values are removed from outputs [S1].

### Steps
1.  **Staged Training:**
    *   **Stage 1 (Pre-training):** Train the model using standard random ray sampling and RGB-space numerical integration [S1].
    *   **Stage 2 (Fine-tuning):** Switch to the alignment-aware strategy [S1].

2.  **Patch-Based Sampling:**
    *   Instead of random rays, sample a batch of rays corresponding to a local image region (patch) [S1].
    *   Standard patch size is $32 \times 32$ pixels [S1].
    *   During fine-tuning, sample a larger ground truth patch $G$ (e.g., $48 \times 48$ pixels) [S1].

3.  **Feature-Space Rendering:**
    *   Compute the volumetric rendering integral in the $N$-dimensional feature space rather than RGB space [S1].
    *   This yields a rendered feature patch $R$ [S1].

4.  **Convolutional Refinement:**
    *   Apply the 3-layer CNN to the rendered feature patch $R$ to incorporate neighborhood information [S1].
    *   Use a feed-forward perceptron layer to map the CNN output back to RGB space [S1].

5.  **Alignment-Aware Loss Calculation:**
    *   Estimate optical flow to handle moving objects [S1].
    *   Search for the best-matching subpatch $G_i$ within the larger ground truth patch $G$ that aligns with the rendered patch $R$ [S1].
    *   Compute the loss using a combination of pixel distance and coordinate regularization [S1]:
        $$ L_{PM}(G, R) = \min_{i=1,...,N} \left( D(G_i, R) + \lambda \cdot D_{coord}(G_i, R) \right) $$
        where:
        *   $D(G_i, R) = \|G_i - R\|_2$ is the Euclidean distance between the subpatch and rendered patch.
        *   $D_{coord}(G_i, R) = \sqrt{\Delta x^2 + \Delta y^2}$ is the coordinate offset penalty, where $\Delta x$ and $\Delta y$ are the horizontal and vertical offsets between $G_i$ and $R$.
        *   $\lambda$ is a regularization weight, empirically set to $0.01$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def alignerf_training(model, images, optical_flows, config):
    """
    Staged training for AligNeRF.
    config.patch_size: int, default 32
    config.gt_patch_size: int, default 48
    config.lambda_reg: float, default 0.01
    config.feature_channels: int, N
    """
    
    # Stage 1: Normal Pre-training
    for epoch in range(pretrain_epochs):
        rays = sample_random_rays(images)
        # Standard NeRF rendering in RGB space
        colors, densities = model(rays)
        loss = mse_loss(colors, rays.gt_colors)
        optimize(loss)

    # Stage 2: Alignment-Aware Fine-tuning
    for epoch in range(finetune_epochs):
        # 1. Patch-based sampling
        # Sample a batch of rays forming a 32x32 patch
        patch_rays = sample_patch_rays(images, size=config.patch_size)
        
        # 2. Feature-space rendering
        # MLP outputs N-channel features
        feature_features, densities = model(patch_rays)
        
        # Numerical integration in feature space
        rendered_features = volumetric_render_integral(feature_features, densities)
        
        # 3. Convolutional refinement
        # Apply 3-layer CNN with reflectance padding
        refined_features = conv_network(rendered_features)
        
        # Perceptron to RGB
        rendered_patch = perceptron_layer(refined_features)
        
        # 4. Alignment-aware loss
        # Sample larger GT patch (48x48)
        gt_patch = extract_gt_patch(images, size=config.gt_patch_size)
        
        # Estimate optical flow to warp GT or align coordinates
        # Note: Exact warping logic depends on optical flow implementation [S1]
        # Here we assume optical flow provides alignment offsets
        
        best_loss = infinity
        best_offset = (0, 0)
        
        # Search over all possible subpatches Gi in G
        for i in range(gt_patch.width - config.patch_size + 1):
            for j in range(gt_patch.height - config.patch_size + 1):
                subpatch = gt_patch[i:i+config.patch_size, j:j+config.patch_size]
                
                # Pixel distance
                d_pixel = l2_norm(subpatch - rendered_patch)
                
                # Coordinate regularization
                # Delta x/y are offsets of this subpatch relative to rendered patch center/origin
                delta_x = i - center_x
                delta_y = j - center_y
                d_coord = sqrt(delta_x**2 + delta_y**2)
                
                total_loss = d_pixel + config.lambda_reg * d_coord
                
                if total_loss < best_loss:
                    best_loss = total_loss
                    best_offset = (i, j)
        
        optimize(best_loss)
```

### Parameters
*   **Patch Size:** $32 \times 32$ pixels [S1].
*   **Ground Truth Patch Size:** $48 \times 48$ pixels (during fine-tuning) [S1].
*   **Regularization Weight ($\lambda$):** $0.01$ [S1].
*   **CNN Kernels:** $3 \times 3$ [S1].
*   **Feature Channels ($N$):** Larger than 3 (exact value not specified in sources, but implied to be a hyperparameter) [S1].

### Edge Cases and Pitfalls
*   **Checkerboard Artifacts:** Mitigated by using "reflectance" padding in the CNN and removing padding values from outputs [S1].
*   **Multiview Inconsistency:** The shallow CNN does not perform upsampling, which helps avoid multiview inconsistencies [S1].
*   **Blurry Renderings:** If the rendered patch $R$ is too blurry, it may match multiple subpatches $G_i$ equally well. The coordinate regularization term $D_{coord}$ penalizes large offsets to resolve this ambiguity [S1].

## Operating characteristics
*   **Performance:** On outdoor datasets, AligNeRF achieves a PSNR more than 3 points higher than standard NeRF and better than the best mip-NeRF results [S2].
*   **Efficiency:** The method does not significantly increase training time despite the additional convolutional and alignment steps [S2].
*   **Scalability:** The approach is designed to be plug-and-play for both point-sampling and frustum-based NeRF variants [S1].
*   **Strengths:** Improves high-frequency detail recovery and handles misalignment from moving objects [S1, S2].
*   **Limitations:** Relies on accurate optical flow estimation for moving objects [S1].

## Background
AligNeRF was introduced by Yifan Jiang, Peter Hedman, Ben Mildenhall, Dejia Xu, Jonathan T. Barron, Zhangyang Wang, and Tianfan Xue from the University of Texas at Austin, Google Research, and The Chinese University of Hong Kong [S1]. It was presented at CVPR 2023, addressing limitations of NeRF in high-resolution settings and scenes with moving objects [S1].

## Sources
[S1] AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training, https://doi.org/10.1109/cvpr52729.2023.00013, fetched 2026-10-26
[S2] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
