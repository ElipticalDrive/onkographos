# Dual-Branch Hash Encoder NeRF for Stylization

Also called: Instant Photorealistic Neural Radiance Fields Stylization method, Two-branch Instant-NGP, Instant Neural Radiance Fields Stylization, Instant-NGP Stylization, InstantNeRF-Stylization

## Overview
A method for multi-view image stylization of 3D scenes that uses a hash table-based position encoder (Instant-NGP) split into content and style sub-branches, applying Adaptive Instance Normalization (AdaIN) to position encoding features during inference to transfer style from a target (image or scene set) to novel views without retraining [S1]. It solves the problem of transferring style from a target scene/image to a content scene while preserving geometry, using an efficient hash encoder instead of a standard MLP for position encoding [S1]. Unlike previous works where style targets are only images, this method extends the style target to 3D scene image sets [S1]. Compared to using two independent Instant-NGP networks, the dual-branch approach with shared MLP RGB preserves geometry consistency better and avoids noise [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of images of a content 3D scene; a style target, which is either a single style image or a set of images of a style 3D scene.
*   **Outputs:** Stylized novel view images of the content scene, preserving its geometry but adopting the appearance of the style target.

**Data Structures:**
1.  **Voxel Grid ($V$):** A 3D grid representing the spatial domain for style representation.
    *   Layout: $V \in \mathbb{R}^{N_V \times N_V \times N_V}$.
    *   $N_V$: Voxel resolution. Typical values: 32, 64, 128, 256. Default: 128 [S1].
2.  **Content Hash Encoder:** A hash-based position encoder mapping 3D coordinates to feature vectors for the content scene.
3.  **Style Hash Encoder:** A hash-based position encoder mapping 3D coordinates to feature vectors for the style scene.
4.  **Shared Direction Encoder:** A spherical encoder mapping view directions to feature vectors.
5.  **Shared MLP_RGB:** A multilayer perceptron taking concatenated features from position encoders and the direction encoder to output color ($C$) and density ($\sigma$).
6.  **MLP_Density:** A separate MLP (part of the content branch architecture) that outputs density based on content position encoder features [S1].

**Numbered Steps:**

**Training Process:**
1.  Initialize two position encoder branches (Content Hash Encoder, Style Hash Encoder) and one shared direction encoder (Spherical Encoder). Initialize the shared MLP_RGB and content-specific MLP_Density [S1].
2.  For each training step, sample rays for both the content target and the style target.
    *   If the style target is a single image, treat it as a 3D scene by placing 3D colored voxels representing the 2D image pixels in the center of the 3D space [S1].
    *   If the style target is an image set, treat it as a normal 3D scene [S1].
3.  Encode 3D coordinates ($x, y, z$) using both the Content Hash Encoder and the Style Hash Encoder to obtain content features ($f_c$) and style features ($f_s$) [S1].
4.  Encode view direction ($d$) using the shared Direction Encoder [S1].
5.  Concatenate features and pass through the shared MLP_RGB to predict color and density for both targets.
    *   For the content target, use content features ($f_c$) and direction features.
    *   For the style target, use style features ($f_s$) and direction features [S1].
6.  Compute loss using Huber loss for both content and style targets [S1].
    *   Huber Loss Formula:
        $$
        L = \begin{cases} \frac{1}{2} rel^2, & \text{for } rel \le \delta \\ \delta(rel - \frac{1}{2}\delta), & \text{otherwise} \end{cases}
        $$
        where $rel = |C_{predict} - C_{target}|$, $C_{predict}$ is the rendered color, $C_{target}$ is the target color, and $\delta$ is a hyperparameter [S1].
7.  Optimize the Content Position Encoder based on the content target loss.
8.  Optimize the Style Position Encoder based on the style target loss.
9.  Optimize the Shared MLP_RGB based on the combined content and style target losses [S1].

**Inference (AdaIN Stylization):**
1.  Calculate content position encoder features ($f_c(x, y, z)$) using voxel grid positions as input [S1].
2.  Calculate style position encoder features ($f_s(x, y, z)$) using the same voxel grid positions [S1].
3.  Compute the mean ($F\mu_c, F\mu_s$) and standard deviation ($F\sigma_c, F\sigma_s$) of the content and style voxel grid features respectively [S1].
4.  Apply AdaIN transformation to the content features using the style statistics as reference:
    $$
    AdaIN(f(x, y, z)) = F\sigma_s \left( \frac{f(x, y, z) - F\mu_c}{F\sigma_c} \right) + F\mu_s
    $$
    where $f(x, y, z)$ is the content voxel grid feature, $F\mu_c, F\sigma_c$ are the mean and std of content voxel grid features, and $F\mu_s, F\sigma_s$ are the mean and std of style voxel grid features [S1].
5.  Use the adjusted (stylized) features for color prediction via the shared MLP_RGB.
6.  Use the unadjusted (content) features for density prediction via the content MLP_Density to preserve geometry consistency [S1].
7.  (Optional) Control the degree of stylization using a style weight $\alpha \in [0, 1]$:
    $$
    AdaIN_c(f(x, y, z)) = (1-\alpha)f(x, y, z) + \alpha AdaIN(f(x, y, z))
    $$
    where $\alpha=0$ is no style and $\alpha=1$ is full style [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def huber_loss(predicted_color, target_color, delta=0.01):
    """
    Computes the Huber loss between predicted and target colors.
    """
    rel = np.abs(predicted_color - target_color)
    loss = np.where(rel <= delta, 0.5 * rel**2, delta * (rel - 0.5 * delta))
    return np.mean(loss)

def adain_transform(content_features, style_mean, style_std, content_mean, content_std):
    """
    Applies Adaptive Instance Normalization to content features using style statistics.
    """
    normalized = (content_features - content_mean) / content_std
    stylized = style_std * normalized + style_mean
    return stylized

def train_dual_branch_nerf(content_images, style_images_or_image, num_steps, learning_rate):
    """
    Trains the Dual-Branch Hash Encoder NeRF.
    """
    # Initialize encoders and MLPs
    content_hash_encoder = HashEncoder()
    style_hash_encoder = HashEncoder()
    shared_dir_encoder = SphericalEncoder()
    shared_mlp_rgb = MLP_RGB()
    content_mlp_density = MLP_Density()
    
    # If style is a single image, convert to 3D colored voxels
    if is_single_image(style_images_or_image):
        style_voxels = create_3d_colored_voxels(style_images_or_image, center=True)
    else:
        style_voxels = style_images_or_image # Treat as 3D scene image set

    for step in range(num_steps):
        # Sample rays for content and style
        content_rays = sample_rays(content_images)
        style_rays = sample_rays(style_voxels)

        # Encode content
        content_pos_feat = content_hash_encoder(content_rays.positions)
        content_dir_feat = shared_dir_encoder(content_rays.directions)
        content_color, content_density = shared_mlp_rgb(concat(content_pos_feat, content_dir_feat))
        
        # Encode style
        style_pos_feat = style_hash_encoder(style_rays.positions)
        style_dir_feat = shared_dir_encoder(style_rays.directions)
        style_color, style_density = shared_mlp_rgb(concat(style_pos_feat, style_dir_feat))

        # Compute losses
        content_loss = huber_loss(content_color, content_rays.target_colors)
        style_loss = huber_loss(style_color, style_rays.target_colors)
        total_loss = content_loss + style_loss

        # Backpropagation and Optimization
        # Optimize content position encoder based on content_loss
        # Optimize style position encoder based on style_loss
        # Optimize shared MLP_RGB based on total_loss
        # Optimize content MLP_Density based on content_loss (implied by architecture)
        optimize_step([content_hash_encoder, style_hash_encoder, shared_mlp_rgb, content_mlp_density], total_loss, learning_rate)

def infer_stylized_view(content_hash_encoder, style_hash_encoder, shared_dir_encoder, shared_mlp_rgb, content_mlp_density, 
                        content_voxel_grid, style_voxel_grid, query_positions, query_directions, alpha=1.0):
    """
    Infers a stylized view using AdaIN on position encoder features.
    """
    # 1. Calculate content features
    content_pos_feat = content_hash_encoder(query_positions)
    
    # 2. Calculate style features
    style_pos_feat = style_hash_encoder(query_positions)
    
    # 3. Compute statistics from voxel grids
    # Note: In practice, these are pre-computed or computed on-the-fly from the grid features
    content_mean = np.mean(content_voxel_grid)
    content_std = np.std(content_voxel_grid)
    style_mean = np.mean(style_voxel_grid)
    style_std = np.std(style_voxel_grid)
    
    # 4. Apply AdaIN
    stylized_pos_feat = adain_transform(content_pos_feat, style_mean, style_std, content_mean, content_std)
    
    # 5. Content-style trade-off
    blended_pos_feat = (1 - alpha) * content_pos_feat + alpha * stylized_pos_feat
    
    # 6. Encode direction
    dir_feat = shared_dir_encoder(query_directions)
    
    # 7. Predict Color (using blended/stylized features)
    color = shared_mlp_rgb(concat(blended_pos_feat, dir_feat))
    
    # 8. Predict Density (using unadjusted content features for geometry consistency)
    density = content_mlp_density(content_pos_feat)
    
    return color, density
```

## Operating characteristics
*   **Time Complexity:** Training is extremely fast, completing in less than 10 minutes on modern GPU hardware (e.g., Nvidia 3080) [S1].
*   **Memory Complexity:** Increases storage compared to a single scene NeRF due to the two position encoder branches, but the hash encoder architecture keeps it efficient [S1].
*   **Strengths:** Fast training; preserves geometry consistency better than independent networks; handles both single image and 3D scene image set style targets; no retraining required for inference [S1].
*   **Limitations:** Does not handle stylization on illumination and reflection because AdaIN is only executed on position encoder features, leaving direction encoder features unchanged [S1]. Details of results vary with random encoding parameters, requiring fixed MLP weights for stable results across different scene pairs [S1].
*   **When to use:** When fast stylization of 3D scenes is required, especially when the style target is a 3D scene image set rather than just a single image.
*   **When not to use:** When stylization of lighting/reflection effects is critical, or when consistent random initialization is required without fixing MLP weights.

## Background
Originated in the paper "Instant Photorealistic Neural Radiance Fields Stylization" [S1]. It builds on Instant-NGP [Parent: Instant Neural Graphics Primitives] and NeRF [Parent: Neural Radiance Fields]. It extends previous NeRF stylization methods (like SNeRF [S1]) by leveraging the hash encoder for speed and extending style targets to 3D scene image sets.

## Sources
[S1] Instant Photorealistic Neural Radiance Fields Stylization, https://doi.org/10.1109/icassp48485.2024.10446082, fetched 2026-10-26
