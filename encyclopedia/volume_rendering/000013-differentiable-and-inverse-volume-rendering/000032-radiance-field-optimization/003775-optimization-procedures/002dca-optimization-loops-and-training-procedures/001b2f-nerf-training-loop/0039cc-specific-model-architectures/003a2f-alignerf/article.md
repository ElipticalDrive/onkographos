# AligNeRF

Also called: AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training

## Overview
AligNeRF is a training strategy and architectural augmentation for Neural Radiance Fields (NeRF) designed to address misalignment artifacts caused by moving objects or camera calibration errors, while improving high-frequency detail recovery in high-resolution settings [S1, S2]. It sets itself apart from standard NeRF and its siblings by combining a convolution-augmented MLP architecture with a staged, alignment-aware fine-tuning procedure that utilizes patch-based sampling and optical flow estimation [S1, S2]. Unlike methods that solely modify the network structure, AligNeRF explicitly handles temporal and spatial misalignment during the optimization loop [S1].

## How it works
AligNeRF builds on [Specific Model Architectures](8585-specific-model-architectures.md) by modifying the baseline (specifically mip-NeRF 360) with convolutional layers and a specialized training loop.

### Inputs and Outputs
*   **Inputs:** A set of calibrated images $\{I_k\}$ with known camera poses. Ground truth patches are extracted from these images.
*   **Outputs:** Rendered color images $\mathbf{C}(\mathbf{r})$ for novel viewpoints.

### Data Structures
*   **MLP Network ($F_\Theta$):** Maps 3D coordinates $\mathbf{x}$ and viewing directions $\mathbf{d}$ to a feature vector $\mathbf{f} \in \mathbb{R}^N$ and volume density $\sigma$. The final layer output channels are changed from 3 (RGB) to $N$ (feature space) [S1].
*   **Convolutional Network ($C$):** A 3-layer CNN with ReLU activations and $3 \times 3$ kernels [S1].
*   **Perceptron Layer ($P$):** A feed-forward layer mapping feature space to RGB space [S1].
*   **Optical Flow:** Estimated optical flow between frames is used to warp ground truth patches [S1].

### Numbered Steps

1.  **Pre-training (Normal Stage):** Train the baseline NeRF model (e.g., mip-NeRF 360) using standard random ray sampling and RGB-space rendering for a fixed duration [S1].
2.  **Architecture Augmentation:**
    *   Switch from random ray sampling to **patch-based sampling**. Gather a local image region (e.g., $32 \times 32$ pixels) for each iteration [S1].
    *   Modify the MLP to output an $N$-dimensional feature vector $\mathbf{f}$ instead of RGB [S1].
    *   Perform numerical integration along the ray in the feature space to compute a rendered feature patch $\mathbf{F}_{render}$ [S1].
    *   Apply the 3-layer CNN to $\mathbf{F}_{render}$. Use "reflectance" padding for CNN inputs and remove padding values in outputs to prevent checkerboard artifacts [S1].
    *   Pass the CNN output through the Perceptron Layer to produce the final rendered RGB patch $\mathbf{R}$ [S1].
3.  **Alignment-Aware Fine-Tuning:**
    *   Estimate optical flow between the current view and neighboring views to handle moving objects [S1].
    *   Sample a larger ground truth patch $\mathbf{G}$ (e.g., $48 \times 48$ pixels) [S1].
    *   Search over every possible subpatch $\mathbf{G}_i$ within $\mathbf{G}$ to find the best match with the rendered patch $\mathbf{R}$ [S1].
    *   Compute the loss using the minimum distance and a regularization term based on coordinate offset [S1].

### Formulas

**Volume Rendering in Feature Space:**
The rendered feature vector $\mathbf{f}_i$ for sample $i$ is accumulated similarly to color, but in $N$-dimensional space:
$$
\mathbf{F}_{render} = \sum_{i=1}^{K} T_i \alpha_i \mathbf{f}_i, \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i), \quad T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)
$$
where $\mathbf{f}_i = F_\Theta(\mathbf{x}_i, \mathbf{d})$ is the feature output [S1].

**Alignment-Aware Loss:**
The loss function $L_{PM}$ minimizes the distance between the rendered patch $\mathbf{R}$ and the best-matching subpatch $\mathbf{G}_i$ from the larger ground truth patch $\mathbf{G}$:
$$
L_{PM}(\mathbf{G}, \mathbf{R}) = \min_{i=1,\dots,M} \left( \|\mathbf{G}_i - \mathbf{R}\|_2^2 + \lambda \cdot D_{coord}(\mathbf{G}_i, \mathbf{R}) \right)
$$
where:
*   $\|\mathbf{G}_i - \mathbf{R}\|_2^2$ is the Euclidean distance (L2 norm) between the subpatch and rendered patch [S1].
*   $D_{coord}(\mathbf{G}_i, \mathbf{R}) = \sqrt{\Delta x^2 + \Delta y^2}$ is the coordinate offset penalty, where $\Delta x$ and $\Delta y$ are the horizontal and vertical offsets between $\mathbf{G}_i$ and $\mathbf{R}$ [S1].
*   $\lambda$ is a regularization weight, empirically set to $0.01$ [S1].
*   $M$ is the number of possible subpatches in $\mathbf{G}$ [S1].

### Parameters
*   **Patch Size (Rendered):** $32 \times 32$ pixels [S1].
*   **Patch Size (Ground Truth Search):** $48 \times 48$ pixels [S1].
*   **Regularization Weight ($\lambda$):** $0.01$ [S1].
*   **CNN Kernels:** $3 \times 3$ [S1].
*   **Feature Dimension ($N$):** Not given in the sources.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def alignerf_training_loop(images, poses, num_iterations):
    # 1. Initialize baseline model (e.g., mip-NeRF 360)
    # MLP outputs feature vector f of size N and density sigma
    model = initialize_mipnerf360()
    
    # 2. Pre-training stage
    for iteration in range(pre_train_steps):
        rays = sample_random_rays(images, poses)
        # Standard rendering in RGB space for pre-training
        colors = render_rgb(model, rays)
        loss = mse_loss(colors, ground_truth_pixels)
        update_model(model, loss)
        
    # 3. Architecture Augmentation
    # Change MLP last layer to output N-dimensional feature vector
    model.change_output_to_feature_space(N)
    
    # Add CNN and Perceptron layers
    cnn = ConvNet(layers=3, kernel_size=3, activation='relu')
    perceptron = Perceptron(input_dim=cnn.output_dim, output_dim=3)
    
    # 4. Alignment-Aware Fine-Tuning stage
    for iteration in range(fine_tune_steps):
        # Patch-based sampling
        patch_coords = sample_patch_centers(images, size=32)
        
        rendered_patches = []
        ground_truth_patches = []
        
        for coord in patch_coords:
            # Gather rays for the 32x32 patch
            patch_rays = gather_rays_for_patch(coord, poses)
            
            # Render in feature space
            features = render_feature_space(model, patch_rays)
            
            # Apply CNN with reflectance padding
            # Reflectance padding: pad with edge values
            padded_features = reflectance_pad(features)
            cnn_features = cnn(padded_features)
            # Remove padding values from output
            cropped_cnn_features = remove_padding(cnn_features)
            
            # Convert to RGB
            rendered_patch = perceptron(cropped_cnn_features)
            rendered_patches.append(rendered_patch)
            
            # Get corresponding ground truth patch (48x48)
            gt_patch_large = extract_patch(images, coord, size=48)
            ground_truth_patches.append(gt_patch_large)
            
        # Alignment-aware loss computation
        total_loss = 0
        for R, G in zip(rendered_patches, ground_truth_patches):
            # Search for best matching subpatch Gi in G
            best_dist = infinity
            best_offset = (0, 0)
            
            # Iterate over all possible subpatches Gi of G that match R's size
            for i in range(number_of_subpatches):
                Gi = extract_subpatch(G, index=i)
                
                # L2 distance
                dist_l2 = l2_norm(Gi - R)
                
                # Coordinate offset penalty
                dx, dy = get_offset(Gi, R)
                dist_coord = sqrt(dx**2 + dy**2)
                
                # Combined distance
                combined_dist = dist_l2 + 0.01 * dist_coord
                
                if combined_dist < best_dist:
                    best_dist = combined_dist
                    best_offset = (dx, dy)
            
            total_loss += best_dist
            
        update_model(model, total_loss)
```

### Edge Cases and Pitfalls
*   **Checkerboard Artifacts:** Using standard padding in CNNs can cause checkerboard artifacts. The method specifies using "reflectance" padding and removing padding values in outputs to prevent this [S1].
*   **Blurry Renderings:** If the NeRF renders a very blurry patch, it may match many subpatches equally well. The coordinate offset regularization term $D_{coord}$ penalizes this ambiguity [S1].
*   **Moving Objects:** Standard NeRF assumes static scenes. Misalignment from moving objects degrades performance. The alignment-aware strategy corrects this by warping ground truth patches using estimated optical flow [S1, S2].

## Operating characteristics
*   **Time Complexity:** The alignment-aware fine-tuning stage involves searching over all possible subpatches, which increases computational cost per iteration compared to standard NeRF. However, the CNN is shallow and does not perform upsampling, limiting the overhead [S1].
*   **Memory Complexity:** The feature space dimension $N$ is larger than 3, increasing memory usage for the MLP outputs. The patch-based sampling requires storing larger ground truth patches ($48 \times 48$) [S1].
*   **Performance:** On outdoor datasets, AligNeRF achieves a PSNR more than 3 points higher than standard NeRF and better than the best mip-NeRF result [S2].
*   **Strengths:** Improves high-frequency detail recovery and handles misalignment from moving objects or calibration errors [S1, S2].
*   **Limitations:** Requires optical flow estimation. The search over subpatches adds complexity to the training loop [S1].

## Background
AligNeRF was introduced by Yifan Jiang, Peter Hedman, Ben Mildenhall, Dejia Xu, Jonathan T. Barron, Zhangyang Wang, and Tianfan Xue from the University of Texas at Austin, Google Research, and The Chinese University of Hong Kong [S1]. It was published in CVPR 2023 (arXiv:2211.09682v1, Nov 2022) [S1]. The method addresses limitations of training NeRF on high-resolution images and dynamic scenes by combining MLP with convolutional layers and proposing alignment-aware strategies [S2].

## Sources
[S1] AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training, https://doi.org/10.1109/cvpr52729.2023.00013, fetched 2026-10-26
[S2] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
