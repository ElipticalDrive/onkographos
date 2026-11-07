# AligNeRF

Also called: AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training

## Overview
AligNeRF is an alignment-aware training strategy for Neural Radiance Fields (NeRF) that addresses rendering artifacts caused by misalignment from moving objects or camera calibration errors, while improving high-frequency detail recovery in high-resolution settings [S1, S2]. It distinguishes itself from standard NeRF variants by combining a convolution-augmented architecture with a staged training procedure that switches from random ray sampling to patch-based sampling, explicitly optimizing for spatial alignment between rendered and ground truth patches [S1].

## How it works
AligNeRF builds on the mip-NeRF 360 baseline [S1]. The method modifies the standard NeRF pipeline through architectural changes (convolutional post-processing) and a two-stage training loop (pre-training followed by alignment-aware fine-tuning) [S1].

### Inputs and Outputs
*   **Inputs:** Camera poses, intrinsics, and a set of input images.
*   **Outputs:** Rendered images for novel viewpoints.

### Data Structures
*   **MLP Network ($F_\Theta$):** Maps 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ to a feature vector $\mathbf{f} \in \mathbb{R}^N$ and volume density $\sigma$. The final layer output channels are changed from 3 (RGB) to $N$ [S1].
*   **Convolutional Network ($C$):** A 3-layer CNN with ReLU activations and $3 \times 3$ kernels [S1].
*   **Perceptron Layer ($P$):** A feed-forward layer mapping the CNN output from feature space to RGB space [S1].
*   **Patches:**
    *   Rendered patch $R$: Size $32 \times 32$ pixels [S1].
    *   Ground truth patch $G$: Size $48 \times 48$ pixels [S1].
    *   Subpatches $G_i$: $32 \times 32$ sub-regions extracted from $G$ [S1].

### Numbered Steps

1.  **Pre-training Stage:** Train the standard NeRF model (using mip-NeRF 360 as baseline) using random ray sampling and standard RGB rendering loss [S1].
2.  **Architecture Modification:**
    a. Switch from random ray sampling to **patch-based sampling**. Gather a $32 \times 32$ local image region during each iteration [S1].
    b. Modify the MLP output head to output $N$ channels (feature space) instead of 3 (RGB) [S1].
    c. Perform numerical integration along the ray in the feature space to produce a feature map, rather than in RGB space [S1].
    d. Apply the 3-layer CNN to the rendered feature map. Use "reflectance" padding for CNN inputs and remove padding values in outputs to prevent checkerboard artifacts [S1].
    e. Apply the perceptron layer to convert the CNN output to RGB [S1].
3.  **Alignment-Aware Fine-tuning Stage:**
    a. During each iteration, sample a larger ground truth patch $G$ of size $48 \times 48$ [S1].
    b. Render a patch $R$ of size $32 \times 32$ [S1].
    c. Search over all possible $32 \times 32$ subpatches $G_i$ within $G$ to find the best match with $R$ [S1].
    d. Compute the loss using the alignment-aware metric (see Formulas) [S1].

### Formulas
The standard volume rendering equation is applied in feature space:
$$ \mathbf{f} = \sum_{i=1}^{N} T_i \alpha_i \mathbf{f}_i, \quad T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right), \quad \alpha_i = 1 - \exp(-\sigma_i \delta_i) $$
where $\mathbf{f}_i$ is the feature vector at sample $i$ [S1].

The alignment-aware loss function $L_{PM}$ is defined as:
$$ L_{PM}(G, R) = \min_{i=1,...,N} \left( D(G_i, R) + \lambda \cdot D_{coord}(G_i, R) \right) $$
where:
*   $D(G_i, R) = \| G_i - R \|_2$ is the Euclidean distance between the subpatch $G_i$ and rendered patch $R$ [S1].
*   $D_{coord}(G_i, R) = \sqrt{\Delta x^2 + \Delta y^2}$ is the coordinate distance, where $\Delta x$ and $\Delta y$ are the horizontal and vertical offsets between $G_i$ and $R$ [S1].
*   $\lambda$ is a regularization parameter, empirically set to $0.01$ [S1].

### Parameters
*   **Patch Size (Rendered):** $32 \times 32$ pixels [S1].
*   **Patch Size (Ground Truth):** $48 \times 48$ pixels [S1].
*   **CNN Kernel Size:** $3 \times 3$ [S1].
*   **Regularization Weight ($\lambda$):** $0.01$ [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def alignerf_training(input_images, camera_poses, num_iterations):
    # Stage 1: Pre-training
    model = init_mlp_mipnerf360()
    for iteration in range(pre_train_iterations):
        rays = sample_random_rays(camera_poses)
        color_rgb, sigma = model.forward(rays)
        loss = mse_loss(color_rgb, ground_truth_rgb)
        model.update(loss)

    # Stage 2: Alignment-Aware Fine-tuning
    # Modify model output to N channels
    model.output_channels = N
    cnn = init_conv_network(layers=3, kernel_size=3)
    perceptron = init_perceptron()
    
    for iteration in range(fine_tune_iterations):
        # Patch-based sampling
        patch_coords = sample_patch_centers(camera_poses, patch_size=32)
        
        # Render feature map
        rays = rays_from_patch(patch_coords)
        feature_map, sigma = model.forward(rays)
        
        # Convolutional post-processing
        # Reflectance padding applied internally or explicitly
        conv_output = cnn.forward(feature_map, padding='reflect')
        # Remove padding values
        conv_output = remove_padding(conv_output)
        
        # Convert to RGB
        rendered_patch_R = perceptron.forward(conv_output)
        
        # Ground truth patch G (48x48)
        ground_truth_patch_G = extract_patch(input_images, patch_coords, size=48)
        
        # Alignment-aware loss
        best_loss = infinity
        best_offset = (0, 0)
        
        # Search over all 32x32 subpatches in 48x48 G
        # Number of subpatches = (48-32+1)^2 = 256
        for i in range(256):
            subpatch_Gi = extract_subpatch(ground_truth_patch_G, index=i, size=32)
            
            # Euclidean distance
            dist_color = l2_norm(subpatch_Gi - rendered_patch_R)
            
            # Coordinate distance (offset between subpatch center and R center)
            offset_x, offset_y = get_offset(i, patch_size=32, full_size=48)
            dist_coord = sqrt(offset_x**2 + offset_y**2)
            
            # Combined loss
            current_loss = dist_color + 0.01 * dist_coord
            
            if current_loss < best_loss:
                best_loss = current_loss
                best_offset = (offset_x, offset_y)
        
        model.update(best_loss)
        cnn.update(best_loss)
        perceptron.update(best_loss)
```

### Edge Cases and Pitfalls
*   **Checkerboard Artifacts:** Standard padding in CNNs can cause checkerboard artifacts. AligNeRF uses "reflectance" padding and explicitly removes padding values from the output to prevent this [S1].
*   **Blurry Renderings:** If the NeRF renders a very blurry patch $R$, it may match multiple subpatches $G_i$ equally well. The coordinate distance term $D_{coord}$ acts as a regularization penalty to resolve this ambiguity [S1].
*   **Multiview Inconsistency:** Because the CNN is shallow and performs no upsampling, multiview inconsistency is not observed [S1].

## Operating characteristics
*   **Performance:** On outdoor datasets, AligNeRF achieves a PSNR more than 3 points higher than standard NeRF and better than the best mip-NeRF result [S2].
*   **Training Time:** The method does not significantly increase training time compared to the baseline [S2].
*   **Complexity:** The addition of a shallow 3-layer CNN and the search over subpatches adds computational overhead per iteration, but the shallow nature of the CNN keeps it manageable [S1].
*   **Strengths:** Improves high-frequency detail recovery; handles misalignment from moving objects; easy to plug into existing NeRF-like models (point-sampling or frustum-based) [S1, S2].
*   **Limitations:** Requires patch-based sampling instead of random ray sampling; relies on the assumption that local neighborhood information is beneficial [S1].

## Background
AligNeRF was introduced by Yifan Jiang, Peter Hedman, Ben Mildenhall, Dejia Xu, Jonathan T. Barron, Zhangyang Wang, and Tianfan Xue from the University of Texas at Austin, Google Research, and The Chinese University of Hong Kong [S1]. It was published in CVPR 2023 (arXiv:2211.09682v1, Nov 2022) [S1]. The method addresses limitations of training NeRF on high-resolution images and misalignment issues [S2].

## Sources
[S1] AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training, https://doi.org/10.1109/cvpr52729.2023.00013, fetched 2026-10-26
[S2] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
