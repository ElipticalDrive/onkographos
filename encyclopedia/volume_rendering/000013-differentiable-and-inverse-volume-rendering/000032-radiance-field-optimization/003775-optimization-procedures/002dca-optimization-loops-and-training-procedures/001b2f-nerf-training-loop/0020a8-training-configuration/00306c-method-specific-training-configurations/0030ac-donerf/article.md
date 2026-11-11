# DONeRF

Also called: Depth Oracle Neural Radiance Fields, DONeRF Training Configuration, DONeRF training setup, Depth Oracle Network, DoNeRF

## Overview
DONeRF is a training configuration for neural radiance fields that replaces standard dense raymarching with a two-network architecture: a **depth oracle network** and a **shading network** [S3]. It solves the prohibitive computational cost of NeRF by using the oracle to predict a small set of high-importance sample locations, allowing the shading network to render high-quality images with only 2–16 samples per ray [S3]. Unlike standard NeRF configurations that rely on hierarchical sampling or positional encoding for all inputs, DONeRF explicitly feeds discretized 3D positions to the oracle and uses a classification-based depth prediction strategy to guide local sampling [S1, S3].

## How it works
**Inputs and Outputs:**
*   **Input:** RGBD images (color and ground truth depth) sampled from a **view cell**. A view cell is a bounding box with a primary orientation and maximum viewing angle, capturing rays within specific rotation limits [S3].
*   **Output:** Rendered RGB images and accumulated opacity.

**Data Structures:**
1.  **Depth Oracle Network:** An 8-layer MLP with 256 hidden units per layer.
    *   **Input:** Concatenated position $x \in \mathbb{R}^3$ and view direction $d \in S^2$.
    *   **Encoding:** No positional encoding is applied to inputs [S1].
    *   **Activation:** ReLU for hidden layers [S1].
    *   **Output:** Depth value (or classification logits for discretized depth bins) [S1, S3].
2.  **Shading Network:** An 8-layer MLP with 256 hidden units per layer.
    *   **Input:** Position $x$ and view direction $d$ for samples selected by the oracle.
    *   **Encoding:** Positional encoding $\gamma$ is applied. $\gamma(x)$ is 63-dimensional, $\gamma(d)$ is 27-dimensional [S1].
    *   **Architecture:** $\gamma(x)$ is used at the first layer. $\gamma(d)$ is concatenated to the feature vector before the last layer [S1].
    *   **Output:** RGB color $c \in \mathbb{R}^3$ and volume density $\sigma \in \mathbb{R}^+$ [S1].
3.  **Dense NeRF (for No-GT initialization):** 12-layer MLP, 512 hidden units per layer [S3].

**Numbered Steps:**
1.  **Ray Unification:** Transform view rays into a unified space relative to the view cell's primary orientation [S3].
2.  **Non-linear Depth Space:** Transform depth values into a non-linear space to handle large open scenes and focus on close regions [S3].
3.  **Oracle Prediction:** The depth oracle network predicts sample locations. It learns a classification task using filtered, discretized target depth values rather than direct depth regression [S3].
4.  **Warping:** Sample positions predicted by the oracle are warped towards the view cell to direct high-frequency predictions of the shading network to the foreground [S3].
5.  **Shading and Accumulation:** The shading network evaluates color and density at the warped sample locations. The final color is accumulated using standard volume rendering integration [S3].

**Formulas:**
*   **Number of Runs (# Run):** In DONeRF, the effective computational cost relative to a baseline is calculated as:
    $$ \# \text{Run} = 1 + \frac{N_s}{N_p} $$
    where $N_s$ is the number of samples selected by the depth oracle and used in the shading network, and $N_p$ is the number of grouped samples. In standard DONeRF, $N_p = 1$ [S1].
*   **Opacity Loss ($\text{loss}_O$):** A penalty term to prevent dark pixel artifacts caused by low sample counts failing to accumulate to full opacity:
    $$ \text{loss}_O = \begin{cases} 0, & \text{if } \sum_{i=1}^X \delta_i \geq 1 \\ \left( \left( \sum_{i=1}^X \delta_i \right) - 1 \right)^2, & \text{otherwise} \end{cases} $$
    where $X$ is the number of samples per ray and $\delta_i$ is the opacity contribution of sample $i$ [S3].
*   **Total Loss:**
    $$ \text{loss} = \alpha \cdot \text{loss}_{\text{MSE}} + \beta \cdot \text{loss}_O $$
    where $\text{loss}_{\text{MSE}}$ is the Mean Squared Error between rendered and ground truth RGB, $\alpha$ is the MSE weight, and $\beta$ is the opacity loss weight [S3].

**Parameters:**
*   **$N_s$ (Samples per ray):** Typically 2, 4, 8, or 16. Named DONeRF-2, DONeRF-4, etc. [S3].
*   **$\alpha$ (MSE Weight):** 1.0 [S3].
*   **$\beta$ (Opacity Loss Weight):** 10.0 [S3].
*   **Oracle Configuration:** K-5, Z-5, I-128 (parameters related to the discretization and input dimensions of the oracle) [S3].
*   **Optimizer:** Adam [S3].
*   **Learning Rate:** 0.0005 [S3].
*   **Batch Size:** 4096 samples per iteration [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# DONeRF Training Loop
# Assumes access to view cell parameters, ground truth depth maps, and image data

def train_donerf():
    # 1. Initialize Networks
    oracle_net = MLP(layers=8, hidden=256, activation='ReLU', no_pos_enc=True)
    shading_net = MLP(layers=8, hidden=256, activation='ReLU', pos_enc=True)
    
    # 2. Training Schedule
    # Phase 1: Train Oracle
    optimizer_oracle = Adam(oracle_net.parameters(), lr=0.0005)
    for iteration in range(300,000):
        batch_rays = sample_rays_from_view_cell(batch_size=4096)
        
        # Get ground truth depth for these rays
        gt_depths = get_ground_truth_depth(batch_rays)
        
        # Transform depth to non-linear space and discretize for classification
        transformed_depths = transform_depth_nonlinear(gt_depths)
        discretized_targets = discretize(transformed_depths)
        
        # Predict depth classification
        pred_logits = oracle_net(batch_rays.position, batch_rays.direction)
        
        # Compute BCE loss for classification
        loss_oracle = BCE_loss(pred_logits, discretized_targets)
        
        optimizer_oracle.zero_grad()
        loss_oracle.backward()
        optimizer_oracle.step()
        
    # Phase 2: Train Shading Network
    optimizer_shading = Adam(shading_net.parameters(), lr=0.0005)
    for iteration in range(300,000):
        batch_rays = sample_rays_from_view_cell(batch_size=4096)
        
        # Get ground truth depth
        gt_depths = get_ground_truth_depth(batch_rays)
        
        # Use Oracle to select samples
        # Oracle predicts importance/locations based on unified ray description
        selected_samples = oracle_select_samples(batch_rays, oracle_net, Ns=4)
        
        # Warp samples towards view cell
        warped_samples = warp_samples_towards_view_cell(selected_samples, batch_rays)
        
        # Query Shading Network for color and density
        colors, densities = shading_net(warped_samples.position, warped_samples.direction)
        
        # Volume Rendering Accumulation
        rendered_colors, accumulated_opacities = volume_render(warped_samples, colors, densities)
        
        # Compute Losses
        loss_mse = MSE_loss(rendered_colors, batch_rays.gt_color)
        
        # Opacity Loss: Force accumulated opacity >= 1 for opaque surfaces
        opacity_sum = accumulated_opacities
        opacity_loss = 0
        for i in range(len(opacity_sum)):
            if opacity_sum[i] < 1.0:
                opacity_loss += (opacity_sum[i] - 1.0)**2
        opacity_loss = opacity_loss / len(opacity_sum)
        
        total_loss = 1.0 * loss_mse + 10.0 * opacity_loss
        
        optimizer_shading.zero_grad()
        total_loss.backward()
        optimizer_shading.step()

def train_donerf_no_gt():
    """
    Alternative initialization for scenes without ground truth depth.
    """
    # Train a dense NeRF first
    dense_nerf = MLP(layers=12, hidden=512)
    optimizer_dense = Adam(dense_nerf.parameters(), lr=0.0005)
    
    for iteration in range(300,000): # Approximate duration
        batch_rays = sample_rays_from_view_cell(batch_size=4096)
        # Render with 128 samples per ray
        colors, densities = dense_nerf(batch_rays.position, batch_rays.direction)
        rendered_colors, _ = volume_render(batch_rays, colors, densities, num_samples=128)
        
        loss = MSE_loss(rendered_colors, batch_rays.gt_color)
        optimizer_dense.zero_grad()
        loss.backward()
        optimizer_dense.step()
        
    # Extract depth maps from the trained dense NeRF for all poses
    extracted_depth_maps = extract_depth_from_nerf(dense_nerf, all_poses)
    
    # Use extracted depth maps as ground truth for DONeRF training
    # Proceed with standard DONeRF training using extracted_depth_maps as gt_depths
    pass
```

**Edge Cases and Pitfalls:**
*   **Dark Pixel Artifacts:** At low sample counts, rays may not accumulate to opacity 1 for opaque surfaces, causing darkening. This is remedied by the opacity loss term [S3].
*   **Cheating:** Shading networks with low sample counts may "cheat" by relying on opacity outputs to mix background color into the accumulation [S3].
*   **Reflective/Transparent Surfaces:** Low sample counts and single depth estimates are insufficient for perfect replication of view-dependent phenomena like mirrors or refractive water [S3].
*   **No-GT Background Noise:** When training without ground truth depth, the initializing NeRF may output high-frequency depth values in the background, creating zero-contribution samples. Adding background detection during oracle training is suggested as a remedy [S3].
*   **Non-End-to-End:** The architecture is not end-to-end trainable and struggles without high-quality depth supervision [S2].

## Subtopics

### [DONeRF Oracle Input Construction](4737-donerf-oracle-input-construction.md)
Defines the input vector construction for the depth oracle network, solving the problem of providing sufficient spatial context for accurate depth range classification.

## Operating characteristics
*   **Time Complexity:** DONeRF requires approximately 4.35 MFLOP per pixel [S3].
*   **Memory Complexity:** Compact; requires only two MLPs (approx. 4 MB) without additional data structures or acceleration structures like sparse voxel octrees [S3].
*   **Measured Performance:**
    *   Renders at 22 ms per frame on a single GPU at 400x400 pixels resolution [S3].
    *   On NVIDIA RTX 3090 at 800x800 resolution with 2 samples/ray: Total 51.3 ms (~20 fps). Breakdown: Ray unification (0.21 ms), Oracle (12.64 ms), Warping (3.46 ms), Shading (34.9 ms), Final color (0.09 ms) [S3].
    *   Shading network cost scales nearly linearly with samples: 34.9 ms (2 samples), 65.4 ms (4 samples), 136.4 ms (8 samples), 270.4 ms (16 samples) [S3].
*   **Quality:** Average PSNR of 31.14 dB compared to NeRF's 30.52 dB [S3]. At 16 samples, up to 7 dB higher PSNR than NeRF [S3].
*   **Speedup:** Inference costs reduced by up to 48x compared to NeRF when conditioning on ground truth depth [S3].
*   **Strengths:** Real-time rendering capability, high compactness, better quality than NeRF on large scenes, no need for explicit caching structures.
*   **Limitations:** Requires ground truth depth (or a pre-trained dense NeRF initialization), struggles with reflective/transparent surfaces, not end-to-end trainable.

## Background
DONeRF was introduced by [S3] as a method to bring compact neural representations closer to practical real-time rendering applications such as games and virtual reality. It addresses the prohibitive computational cost of NeRF, which requires dozens of petaFLOPS. The method was developed to enable interactive rendering (20 fps) on a single GPU while maintaining high image quality. It builds on the concept of depth oracles but introduces specific training configurations including view cells, non-linear depth spaces, and opacity loss regularization [S3].

## Sources
[S1] MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields, https://arxiv.org/abs/2310.01821v1, fetched 2026-10-26
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S3] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
