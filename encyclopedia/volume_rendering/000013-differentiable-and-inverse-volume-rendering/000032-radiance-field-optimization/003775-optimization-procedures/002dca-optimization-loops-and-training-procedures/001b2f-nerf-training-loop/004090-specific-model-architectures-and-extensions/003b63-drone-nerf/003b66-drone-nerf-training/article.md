# Drone-NeRF Training

Also called: NeRF Training in Drone-View Block

## Overview
Drone-NeRF Training is a specialized optimization procedure for reconstructing large-scale 3D scenes from aerial imagery, addressing the challenges of unbounded space representation and varying camera exposure in drone-captured data. It distinguishes itself from standard NeRF training by integrating view frustum sampling for anti-aliasing, a pose optimizer for joint pose refinement, and appearance embeddings to handle exposure discrepancies, while employing a spatial contraction function to map unbounded scenes into a bounded domain for efficient rendering.

## How it works

### Inputs and Outputs
**Inputs:**
*   A set of drone-view images $I_i$ with associated camera poses $P_i$ (extrinsics) and intrinsics $K$.
*   A predefined number of samples $N$ for the initial stage.
*   A spatial distortion correction scope parameter (implicit in the contraction function).

**Outputs:**
*   Optimized neural network weights (MLP parameters for density $\sigma$ and color $c$).
*   Optimized camera poses $P_i$.
*   Appearance embeddings for each image.

### Data Structures
*   **Image Pyramid:** A list of downsampled versions of the input image $I_i$, typically 3-4 levels deep, used to guide sampling density.
*   **View Frustum:** Defined by the camera intrinsics $K$ and extrinsics $P_i$. For a pixel $(u, v)$, the frustum is approximated by the volume between the near and far planes intersected by the pixel's angular extent.
*   **MLP:** A Multi-Layer Perceptron taking encoded position, direction, and appearance embedding as input, outputting volume density $\sigma$ and RGB color $c$.
*   **Appearance Embedding:** A learnable vector $e_i$ associated with each image $i$, concatenated with position/direction encodings.

### Steps

1.  **Image Pyramid Construction:**
    Generate a pyramid of downsampled images from the input $I_i$. This allows the training process to focus on details at various spatial scales, enhancing adaptability to diverse scene structures [S1].

2.  **View Frustum Sampling:**
    For each pixel $(u, v)$ in the target image, sample points within a viewing frustum rather than a single ray. This approximates the pixel area to achieve anti-aliasing and better represent intricate details [S1]. The pixel area is an approximation [S1].

3.  **Spatial Contraction (Unbounded to Bounded):**
    To handle unbounded scenes, apply a spatial distortion correction function $f(x)$ to warp scene coordinates. This ensures scene shrinkage warps unbounded samples into bounded space [S1]. The function is defined as:
    $$
    f(x) = \begin{cases} 
    x & \sqrt{x^2} \leq 1 \\ 
    \left(2 - \frac{1}{\sqrt{x^2}}\right) \cdot \frac{x}{\sqrt{x^2}} & \sqrt{x^2} > 1 
    \end{cases}
    $$
    where $x$ is the position vector. The $L_\infty$ contraction is preferred for cube-space scenes to reduce edge distortion compared to spherical ($L_2$) contraction [S1].

4.  **Multi-Stage Stratified Sampling:**
    *   **Stage 1 (Pre-sampling):** Perform pre-sampling based on the targeted number of samples $N$ using a predefined sampler (e.g., uniform or stratified). Transform the sampled points into Euclidean space. Identify samples that contribute significantly to the final color rendering [S1].
    *   **Stage 2 (PDF Sampling):** Use the weights from Stage 1 to construct a Probability Distribution Function (PDF). Perform stratified sampling according to this PDF, intensifying sampling density in regions with larger weights (high density or high variance) [S1].

5.  **Neural Network Forward Pass:**
    *   Encode the sampled positions and view directions using positional encoding.
    *   Concatenate the appearance embedding $e_i$ (specific to image $i$) with the encoded position and direction.
    *   Pass the concatenated vector through the MLP to predict volume density $\sigma$ and RGB color $c$ [S1].
    *   The appearance embedding resolves discrepancies arising from variations in camera exposure, balancing brightness differences across images [S1].

6.  **Volume Rendering:**
    Compute the rendered color $C$ for each pixel using standard volume rendering integration over the sampled points along the frustum rays:
    $$
    C = \sum_{i=1}^{N} T_i \alpha_i c_i
    $$
    where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$, $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$, and $\delta_i$ is the distance between samples.

7.  **Pose Optimization:**
    Direct the ray bundle (formed by the viewing frustum) into a Pose Optimizer. Back-propagate the loss gradients to the input pose calculation $P_i$ to optimize the pose parameters. This iterative refinement enhances rendering precision [S1].

8.  **Loss Computation and Update:**
    Compute the loss between the rendered image and the ground truth image $I_i$. Update the MLP weights, appearance embeddings, and camera poses using gradient descent.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def drone_nerf_training(images, poses, intrinsics, num_stages=2, num_samples_stage1=64, num_samples_stage2=128):
    """
    Main training loop for Drone-NeRF.
    
    Args:
        images: List of input drone-view images.
        poses: Initial camera poses (extrinsics).
        intrinsics: Camera intrinsics matrix K.
        num_stages: Number of sampling stages (default 2).
        num_samples_stage1: Number of pre-samples for Stage 1.
        num_samples_stage2: Number of PDF samples for Stage 2.
        
    Returns:
        model: Trained MLP model.
        poses: Optimized camera poses.
        embeddings: Learned appearance embeddings.
    """
    
    # Initialize model components
    model = MLP()  # Contains density and color prediction layers
    embeddings = {i: torch.zeros(16) for i in range(len(images))}  # Appearance embeddings
    optimizer_pose = torch.optim.Adam([p for p in poses], lr=1e-4)
    optimizer_model = torch.optim.Adam(list(model.parameters()) + list(embeddings.values()), lr=1e-3)
    
    # Preprocessing: Image Pyramids
    image_pyramids = [build_image_pyramid(img) for img in images]
    
    for epoch in range(num_epochs):
        for i, img in enumerate(images):
            # 1. View Frustum Sampling
            # Sample points within frustum for each pixel to achieve anti-aliasing
            frustum_samples = sample_frustum(img.shape, poses[i], intrinsics, num_samples_stage1)
            
            # 2. Spatial Contraction
            # Warp unbounded samples to bounded space using L_infinity contraction
            contracted_samples = spatial_contraction(frustum_samples)
            
            # 3. Stage 1: Pre-sampling and Euclidean Transformation
            # Transform to Euclidean space and evaluate initial contribution
            # (Assuming model is initialized or from previous epoch)
            pos_enc, dir_enc = encode_positions_and_directions(contracted_samples, poses[i])
            app_emb = embeddings[i]
            inputs = concat(pos_enc, dir_enc, app_emb)
            
            sigma_stage1, color_stage1 = model(inputs)
            
            # Identify significant samples (e.g., based on density or gradient)
            weights_stage1 = compute_sample_weights(sigma_stage1, contracted_samples)
            
            # 4. Stage 2: PDF Sampling
            # Intensify sampling in high-weight regions
            pdf = normalize(weights_stage1)
            final_samples = stratified_pdf_sampling(contracted_samples, pdf, num_samples_stage2)
            
            # Re-encode final samples
            pos_enc_final, dir_enc_final = encode_positions_and_directions(final_samples, poses[i])
            inputs_final = concat(pos_enc_final, dir_enc_final, app_emb)
            
            # 5. Neural Network Forward Pass
            sigma_final, color_final = model(inputs_final)
            
            # 6. Volume Rendering
            rendered_color = volume_rendering(final_samples, sigma_final, color_final)
            
            # 7. Loss Computation
            loss = compute_loss(rendered_color, img)
            
            # 8. Back-propagation
            # Back-propagate to optimize pose and model parameters
            optimizer_pose.zero_grad()
            optimizer_model.zero_grad()
            
            # Compute gradients for pose optimization
            # Note: In practice, this requires differentiable rendering through the pose
            loss.backward(retain_graph=True)
            
            # Update poses
            optimizer_pose.step()
            
            # Update model and embeddings
            optimizer_model.step()
            
    return model, poses, embeddings

def spatial_contraction(x):
    """
    Applies spatial distortion correction function f(x) to warp unbounded space.
    Uses L_infinity contraction logic as described in [S1].
    """
    norm_x = torch.norm(x, dim=-1, keepdim=True)
    # f(x) = x if ||x|| <= 1 else (2 - 1/||x||) * (x / ||x||)
    # Note: The source formula uses sqrt(x^2) which is ||x||.
    # The source specifies L_infinity is preferred for cube space, but the formula 
    # provided is a general radial contraction. We implement the formula as given.
    
    mask = norm_x <= 1.0
    result = torch.where(
        mask,
        x,
        (2.0 - 1.0 / norm_x) * (x / norm_x)
    )
    return result
```

### Parameters and Typical Values
*   **Number of Samples (Stage 1):** Predefined, e.g., 64.
*   **Number of Samples (Stage 2):** Determined by PDF, e.g., 128.
*   **Contraction Type:** $L_\infty$ is preferred for cube-space scenes to reduce edge distortion [S1].
*   **Appearance Embedding Dimension:** Not given in the sources.

### Edge Cases and Pitfalls
*   **Spatial Distortion Scope:** If the space warp scope is not suitable for the scene, the scene within the original reasonable field of view will be distorted and deformed [S1].
*   **Frustum Approximation:** The pixel area utilized in frustum sampling is an approximation [S1].
*   **Exposure Discrepancies:** Without appearance embeddings, variations in camera exposure will cause rendering artifacts [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time:** Higher than standard NeRF due to multi-stage sampling and pose optimization. The frustum sampling adds overhead per pixel compared to single-ray sampling.
*   **Memory:** Increased memory usage due to storing appearance embeddings for each image and maintaining image pyramids.

### Measured Performance
*   Not given in the sources.

### Scaling
*   The method is designed for large-scale drone surveys by dividing the scene into sub-scenes (blocks) [S1].

### Strengths and Limitations
*   **Strengths:** Handles unbounded scenes effectively via spatial contraction; improves rendering quality for intricate details via frustum sampling and anti-aliasing; robust to exposure variations via appearance embeddings; improves pose accuracy via joint optimization [S1].
*   **Limitations:** Sensitive to the choice of spatial warp scope; computational cost of multi-stage sampling and pose optimization is higher than standard NeRF [S1].

## Background

### Origin
Drone-NeRF was introduced in the paper "Drone-NeRF: Efficient NeRF based 3D scene reconstruction for large-scale drone survey" [S1].

### History and Development
The method builds upon NeRF and addresses specific challenges in aerial photogrammetry, such as unbounded scene representation and exposure variations. It integrates techniques from differentiable rendering and pose optimization [S1].

### Usage
Used for 3D scene reconstruction from drone-captured images, particularly in large-scale surveying applications [S1].

## Sources

[S1] Drone-NeRF: Efficient NeRF based 3D scene reconstruction for large-scale drone survey, https://doi.org/10.1016/j.imavis.2024.104920, fetched 2026-10-30
