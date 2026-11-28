# NeRF-VPT

Also called: NeRF with View Prompt Tuning, NeRF-VPT with Cascading Learning, Neural Radiance Fields via View Prompt Tuning

## Overview
NeRF-VPT is a cascading view prompt tuning paradigm for novel view synthesis that iteratively incorporates visual prompts—specifically, the RGB images rendered in preceding training stages—into subsequent network iterations to progressively enhance image quality [S1]. It solves the problem of limited scene understanding in sparse-view or standard NeRF settings by leveraging prior knowledge embedded in the network's own outputs, effectively guiding the optimization direction without requiring external priors or ground-truth images at test time [S1]. Unlike methods that rely on external networks or specific data preprocessing, NeRF-VPT is a plug-and-play modification that re-uses the internal representation of the NeRF family, making it transferable to variants like Mip-NeRF and TensoRF [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training images $\{I_k\}$, camera poses, and ray origins/directions. The method takes the output of the previous stage (or Stage 0 for the first iteration) as a visual prompt.
*   **Outputs:** Refined volume density $\sigma$ and view-dependent color $\mathbf{c}$ for each sampled point along the ray, resulting in a higher-fidelity rendered image $\hat{c}(r)$ [S1].

**Data Structures and Layout:**
The framework consists of $N+1$ stages, indexed $0$ to $N$ (typically $N=6$) [S1].
*   **Stage 0:** A standard NeRF network $F_0$. It takes only position $\mathbf{x}$ and direction $\mathbf{d}$ as input. It does not use any prior [S1].
*   **Stage $i$ ($i > 0$):** A new network $F_i$. It takes position $\mathbf{x}$, direction $\mathbf{d}$, and the rendered RGB image from Stage $i-1$, denoted $c(r)_{i-1}$, as inputs [S1].
*   **Feature Integration:** The RGB prompt $c(r)_{i-1}$ is concatenated after the direction encoding $DE(\mathbf{d})$ [S1]. This placement is chosen because color is high-frequency data, and encoding is viewed as signal decomposition; concatenating after direction encoding allows the network to modulate color prediction based on the prompt [S1].
*   **Parameter Overhead:** The method adds negligible parameters, specifically $0.0008$ M additional parameters compared to the baseline NeRF [S1].

**Numbered Steps:**
1.  **Initialization (Stage 0):** Train a standard NeRF network $F_0$ using position encoding $PE(\mathbf{x})$ and direction encoding $DE(\mathbf{d})$ to gain initial knowledge of the 3D space. This stage generates the first set of rendered RGB images, which serve as the initial visual prompt [S1].
    $$ (\mathbf{c}_0, \sigma_0) = F_0(PE(\mathbf{x}), DE(\mathbf{d})) $$
2.  **Cascading Iteration (Stage $i$, for $i = 1$ to $N$):**
    a.  **Ray Sampling:** Sample points $\mathbf{x}$ and directions $\mathbf{d}$ along rays from training viewpoints [S1].
    b.  **Encoding:** Encode the viewing direction $\mathbf{d}$ using $DE(\mathbf{d})$ [S1].
    c.  **Prompt Retrieval:** Retrieve the RGB image $\mathbf{c}(r)_{i-1}$ rendered by the network in the previous stage $i-1$ [S1].
    d.  **Input Combination:** Concatenate the encoded direction $DE(\mathbf{d})$ and the previous stage's RGB output $\mathbf{c}(r)_{i-1}$ [S1].
    e.  **Network Forward Pass:** Pass the position encoding $PE(\mathbf{x})$ and the combined direction-prompt vector into the current stage's network $F_i$ [S1].
    f.  **Output:** The network outputs the refined density $\sigma_i$ and color $\mathbf{c}_i$ [S1].
    $$ (\mathbf{c}_i, \sigma_i) = F_i(PE(\mathbf{x}), (DE(\mathbf{d}), \mathbf{c}(r)_{i-1})) $$
3.  **Rendering:** Compute the final pixel color $\hat{c}(r)$ using the volumetric rendering equation [S1]:
    $$ \hat{c}(r) = \sum_{i=n}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ handles occlusion, $t_n, t_f$ are near/far boundaries, and $\delta_i$ is the distance between samples [S1].
4.  **Loss Calculation:** Minimize the mean squared error between the rendered color $\hat{c}(r)$ and the ground-truth color $c(r)$ [S1]:
    $$ L = \sum_{r \in R} ||\hat{c}(r) - c(r)|| $$
    where $R$ represents all image pixels in the training data [S1].
5.  **Termination:** Training concludes once the maximum number of stages (typically 5 or 6) is reached [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nerf_vpt(training_images, poses, num_stages=6):
    """
    Trains a NeRF-VPT model using cascading view prompt tuning.
    
    Args:
        training_images: List of ground truth RGB images.
        poses: Camera poses corresponding to training images.
        num_stages: Number of cascading stages (0 to N).
        
    Returns:
        final_network: The network from the final stage.
    """
    # Stage 0: Standard NeRF initialization
    # F_0 takes only position and direction
    network_prev = initialize_standard_nerf() 
    
    for stage in range(1, num_stages + 1):
        # Initialize new network for current stage
        # F_stage takes position, direction, and previous RGB prompt
        network_curr = initialize_nerf_stage_with_prompt()
        
        # Training loop for the current stage
        for epoch in range(max_epochs):
            for ray in sample_rays(training_images, poses):
                # 1. Ray Sampling
                points_x, directions_d = ray.sample_points()
                
                # 2. Encoding
                pos_enc = positional_encoding(points_x)
                dir_enc = directional_encoding(directions_d)
                
                # 3. Retrieve Visual Prompt
                # Render image using the previous stage's network
                # This provides the 'prior knowledge' for the current stage
                prompt_rgb = render_image(network_prev, ray)
                
                # 4. Combine Inputs
                # Concatenate direction encoding and prompt RGB
                # Prompt is concatenated after direction encoding
                combined_input = concatenate(dir_enc, prompt_rgb)
                
                # 5. Forward Pass
                # Input: Position Encoding, Combined Direction-Prompt
                sigma, color = network_curr(pos_enc, combined_input)
                
                # 6. Volumetric Rendering
                # Calculate occlusion weights
                T = compute_transmittance(sigma)
                # Render final color
                rendered_color = volumetric_render(T, sigma, color)
                
                # 7. Loss Calculation
                gt_color = ray.get_ground_truth_color()
                loss = mean_squared_error(rendered_color, gt_color)
                
                # 8. Backpropagation
                network_curr.backward(loss)
                network_curr.step()
        
        # Update previous network for the next stage
        # Weights can be loaded from the previous round to facilitate convergence [S1]
        network_prev = network_curr
        
    return network_prev
```

**Parameters:**
*   **Number of Stages:** 5 to 6 stages are sufficient to achieve maximum performance [S1].
*   **Prior Images for Feature Extraction (Alternative/Exploratory):** 20 uniformly sampled training images were used in ablation studies to extract features via Swin-Transformer, but this was found to be less effective than the cascading RGB prompt [S1].
*   **Resolution:** Training resolution varies by dataset (e.g., $400 \times 400$ for Realistic Synthetic 360, $504 \times 376$ for Real Forward-Facing) [S1].

**Edge Cases and Pitfalls:**
*   **Ground Truth Unavailability:** Directly providing ground truth images as priors fails during testing for unknown viewpoints because ground truth is unavailable [S1]. NeRF-VPT solves this by using rendered images from the pre-trained model as the prompt, which are available for all viewpoints [S1].
*   **Feature Fusion Limitations:** Extracting features from multiple images via Swin-Transformer or CNN and fusing them into an MLP showed limited improvement because it is challenging to encapsulate multi-view information within a single feature [S1].
*   **Placement of Prompt:** Incorporating priors into direction encoding yields higher PSNR than position encoding because direction information induces diverse color variations for the same position after volume density is predicted [S1].

## Subtopics

### Sparse-View NeRF-VPT
A configuration where NeRF-VPT is used to achieve high-quality rendering with significantly fewer input views (e.g., 45 views achieving quality comparable to 180 views in standard NeRF) [S1].

### NeRF-VPT with Mip-NeRF/TensoRF
The plug-and-play integration of the view prompt tuning mechanism into other NeRF variants like Mip-NeRF and TensoRF to elevate their performance [S1].

## Operating characteristics
*   **Time Complexity:** Training time per iteration decreases significantly as the cascade progresses (from Stage 1 to 6). This is because increasingly accurate visual prompts reduce the learning difficulty for the network [S1]. For example, on the Replica dataset, training time drops from 8.78 hours in Stage 1 to 0.61 hours in Stage 6 [S1].
*   **Memory Complexity:** The parameter overhead is negligible, adding only $0.0008$ M parameters compared to the baseline NeRF [S1].
*   **Measured Performance:**
    *   **Realistic Synthetic 360 Dataset:** NeRF-VPT (Stage 6) achieves PSNR 34.95, SSIM 0.977, LPIPS 0.042, outperforming NeRF (31.01, 0.947, 0.081) and Mip-NeRF (33.09, 0.961, 0.043) [S1].
    *   **Real Forward-Facing Dataset:** NeRF-VPT (Stage 6) achieves PSNR 28.95, SSIM 0.872, LPIPS 0.158, outperforming NeRF (26.50, 0.811, 0.250) and Mip-NeRF (27.13, 0.839, 0.164) [S1].
    *   **Replica Dataset:** Rendering speed is 0.108 FPS, compared to 0.103 FPS for NeRF [S1].
*   **Depth Quality:** Depth prediction quality is maintained with a Mean-Squared Error distance of 0.02, identical to baseline NeRF, despite significant image quality improvements [S1].
*   **Strengths:** Plug-and-play compatibility, significant improvement in novel view synthesis quality, effective handling of sparse views, and reduced training time in later stages.
*   **Limitations:** Requires multiple training stages (increasing total training time compared to a single-stage NeRF), and depth estimation does not improve over the baseline [S1].

## Background
NeRF-VPT was introduced in 2024 in the paper "NeRF-VPT: Learning Novel View Representations with Neural Radiance Fields via View Prompt Tuning" presented at the AAAI Conference on Artificial Intelligence [S1]. The method builds on the success of NeRF and addresses limitations in prior knowledge incorporation methods that relied on external networks or ground truth images. It draws inspiration from cascading learning strategies used in other computer vision applications like object detection and super-resolution [S1].

## Sources
[S1] NeRF-VPT: Learning Novel View Representations with Neural Radiance Fields via View Prompt Tuning, https://doi.org/10.1609/aaai.v38i2.27877, fetched 2026-10-27
