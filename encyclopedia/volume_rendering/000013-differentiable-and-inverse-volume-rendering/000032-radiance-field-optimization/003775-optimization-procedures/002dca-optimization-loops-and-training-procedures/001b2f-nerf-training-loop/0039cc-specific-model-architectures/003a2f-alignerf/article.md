# AligNeRF

Also called: Alignment-Aware Training for NeRF

## Overview
AligNeRF is a training strategy and architectural modification for Neural Radiance Fields that addresses rendering artifacts caused by misalignment from moving objects or camera calibration errors, while improving high-frequency detail recovery in high-resolution settings [S1, S2]. It sets itself apart from standard NeRF variants by combining a convolution-augmented architecture with a staged, alignment-aware fine-tuning procedure that explicitly handles spatial misalignment during the optimization loop [S1, S2].

## How it works
AligNeRF builds on [Specific Model Architectures](8585-specific-model-architectures.md) by modifying the input sampling, network topology, and loss function. The method consists of two phases: a "normal" pre-training stage followed by an alignment-aware fine-tuning stage [S1].

### Inputs and Outputs
*   **Inputs:** Camera poses, intrinsics, and a set of input images. During fine-tuning, the method also utilizes estimated optical flow to handle moving objects [S1].
*   **Outputs:** Rendered images (RGB) and volume density $\sigma$ for novel view synthesis [S1].

### Data Structures and Network Topology
The architecture modifies the standard MLP-based NeRF [S1]:
1.  **MLP Feature Head:** The Multi-Layer Perceptron $F_\Theta$ takes 3D position $\mathbf{x}$ and viewing direction $\mathbf{d}$ (after positional encoding) and outputs a feature vector $\mathbf{f} \in \mathbb{R}^N$ and volume density $\sigma$. The number of output channels $N$ is larger than 3 (the standard RGB dimension) [S1].
2.  **Volumetric Rendering:** Numerical integration is performed along the ray in the feature space $\mathbf{f}$ rather than RGB space [S1].
3.  **Convolutional Post-Processing:** A 3-layer convolutional network follows the volumetric rendering function. It uses 3×3 kernels and ReLU activations [S1].
    *   **Padding:** "Reflectance" padding is applied to the inputs of each convolution to prevent checkerboard artifacts. Padding values are removed from the outputs [S1].
4.  **RGB Projection:** A feed-forward perceptron layer at the end of the CNN converts the feature space representation back to RGB space [S1].

### Numbered Steps

**Phase 1: Normal Pre-training**
1.  Initialize the MLP $F_\Theta$ and CNN parameters.
2.  Sample rays randomly from the input images.
3.  Render the scene using the standard NeRF volumetric rendering equation in RGB space (or feature space if configured, but standard pre-training typically aligns with baseline NeRF [S1]).
4.  Optimize parameters using standard photometric loss (e.g., MSE) against ground truth images [S1].

**Phase 2: Alignment-Aware Fine-Tuning**
1.  **Patch-Based Sampling:** Switch from random ray sampling to patch-based sampling. Gather a local image region (patch) for each iteration [S1].
    *   *Parameter:* Rendered patch size $R$ is $32 \times 32$ pixels [S1].
2.  **Feature Space Rendering:** Render the patch $R$ using the MLP to produce feature vectors $\mathbf{f}$, integrate them volumetrically, and pass the result through the CNN and RGB projection layer [S1].
3.  **Ground Truth Patch Selection:** Sample a larger ground truth patch $G$ (size $48 \times 48$ pixels) centered on the corresponding location in the ground truth image [S1].
4.  **Alignment Search:** Search over every possible subpatch $G_i$ within $G$ to find the best match with the rendered patch $R$. This step corrects inaccuracies in alignment caused by moving objects or calibration errors [S1].
5.  **Loss Calculation:** Compute the loss $L_{PM}$ using the best-matching subpatch $G_i$ and a regularization term based on the Euclidean distance (offset) between $G_i$ and $R$ [S1]:
    $$ L_{PM}(G, R) = \min_{i=1,...,N} \left( D(G_i, R) + \lambda \cdot D_{coord}(G_i, R) \right) $$
    Where:
    *   $D(G_i, R) = \| G_i - R \|_2$ is the $L_2$ distance between the subpatch and rendered patch.
    *   $D_{coord}(G_i, R) = \sqrt{\Delta x^2 + \Delta y^2}$ is the Euclidean distance of the horizontal/vertical offset $(\Delta x, \Delta y)$ between $G_i$ and $R$.
    *   $\lambda$ is a regularization weight, empirically set to 0.01 [S1].
6.  **Optimization:** Update network parameters using $L_{PM}$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def alignerf_training(input_images, camera_poses, num_iterations):
    """
    AligNeRF training loop.
    input_images: List of ground truth images.
    camera_poses: Camera poses for each image.
    num_iterations: Total training steps.
    """
    # Initialize MLP and CNN
    # MLP outputs feature vector f of size N and density sigma
    # CNN has 3 layers, 3x3 kernels, ReLU, reflectance padding
    model = AligNeRFModel(N=64) # N is larger than 3, specific value not given in sources [S1]
    
    # Phase 1: Normal Pre-training
    for iteration in range(num_iterations // 2):
        # Random ray sampling
        rays = sample_random_rays(input_images, camera_poses)
        
        # Standard NeRF rendering (RGB space integration)
        rendered_images = render_nerf(model, rays)
        
        # Standard photometric loss
        loss = mse_loss(rendered_images, ground_truth_images)
        optimize(model, loss)

    # Phase 2: Alignment-Aware Fine-Tuning
    # Estimate optical flow for moving objects [S1]
    optical_flow = estimate_optical_flow(input_images)
    
    patch_size_rendered = 32
    patch_size_gt = 48
    lambda_reg = 0.01
    
    for iteration in range(num_iterations // 2, num_iterations):
        # Patch-based sampling
        rays, gt_patches = sample_patch_rays(input_images, camera_poses, patch_size_gt)
        
        # Render patches in feature space, then CNN, then RGB
        rendered_patches = render_patch(model, rays) # Returns RGB patches of size 32x32
        
        # Alignment-aware loss
        total_loss = 0
        for R, G in zip(rendered_patches, gt_patches):
            # Search for best subpatch Gi in G
            best_loss = infinity
            best_offset = (0, 0)
            
            # Iterate over all possible subpatches Gi of G
            # G is 48x48, R is 32x32. Subpatches are 32x32.
            for i in range(G.height - patch_size_rendered + 1):
                for j in range(G.width - patch_size_rendered + 1):
                    Gi = G[i:i+patch_size_rendered, j:j+patch_size_rendered]
                    
                    # D(Gi, R)
                    dist_rgb = l2_norm(Gi - R)
                    
                    # D_coord(Gi, R)
                    # Offset is relative to the center of G or R? 
                    # Sources say "horizontal/vertical offset between Gi and R".
                    # Assuming offset from the center of the expected alignment.
                    # Let's assume (0,0) is the top-left of the expected match.
                    # In practice, this is the displacement from the center of G to the center of Gi.
                    # Or simply the pixel coordinates (i, j) if G is aligned with R initially.
                    # The source defines D_coord = sqrt(dx^2 + dy^2).
                    # We assume dx, dy are the offsets of Gi relative to the center of G.
                    dx = i + patch_size_rendered/2 - G.width/2
                    dy = j + patch_size_rendered/2 - G.height/2
                    dist_coord = sqrt(dx**2 + dy**2)
                    
                    current_loss = dist_rgb + lambda_reg * dist_coord
                    if current_loss < best_loss:
                        best_loss = current_loss
                        best_offset = (dx, dy)
            
            total_loss += best_loss
        
        optimize(model, total_loss)
```

### Parameters
*   **Patch Size (Rendered):** $32 \times 32$ pixels [S1].
*   **Patch Size (Ground Truth):** $48 \times 48$ pixels [S1].
*   **Regularization Weight ($\lambda$):** 0.01 [S1].
*   **MLP Output Channels ($N$):** Larger than 3. Exact value not specified in sources [S1].

### Edge Cases and Pitfalls
*   **Checkerboard Artifacts:** Mitigated by using "reflectance" padding in the CNN and removing padding values in outputs [S1].
*   **Multiview Inconsistency:** The shallow CNN does not perform upsampling, which helps avoid multiview inconsistency [S1].
*   **Moving Objects:** The alignment-aware search explicitly handles misalignment caused by moving objects, which standard NeRF fails to do [S1, S2].

## Operating characteristics
*   **Time Complexity:** The alignment-aware fine-tuning stage involves searching over all possible subpatches $G_i$ for each rendered patch. For a $48 \times 48$ ground truth patch and $32 \times 32$ rendered patch, there are $(48-32+1)^2 = 256$ subpatches to evaluate per pixel/patch. This increases the computational cost of the fine-tuning phase compared to standard NeRF [S1].
*   **Memory Complexity:** The method requires storing the larger ground truth patch $G$ and intermediate feature maps. The CNN adds minimal memory overhead compared to the MLP [S1].
*   **Performance:** On outdoor datasets, AligNeRF achieves a PSNR more than 3 points higher than standard NeRF and better than the best mip-NeRF results [S2]. It improves high-frequency detail recovery without significantly increasing training time compared to mip-NeRF [S1, S2].
*   **Strengths:** Effective for high-resolution images and scenes with moving objects or calibration errors. Easy to plug into existing NeRF-like models [S1, S2].
*   **Limitations:** The patch-based search increases per-iteration cost during fine-tuning. The method assumes that misalignment can be corrected by spatial shifting within the patch [S1].

## Background
AligNeRF was introduced by Yifan Jiang, Peter Hedman, Ben Mildenhall, Dejia Xu, Jonathan T. Barron, Zhangyang Wang, and Tianfan Xue from the University of Texas at Austin, Google Research, and The Chinese University of Hong Kong [S1]. It was published in CVPR 2023 (arXiv:2211.09682v1, Nov 2022) [S1]. The method addresses limitations of training NeRF on high-resolution images and in the presence of moving objects, building on the mip-NeRF 360 baseline [S1].

## Sources
[S1] AligNeRF: High-Fidelity Neural Radiance Fields via Alignment-Aware Training, https://doi.org/10.1109/cvpr52729.2023.00013, fetched 2026-10-26
[S2] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
