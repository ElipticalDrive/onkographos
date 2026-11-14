# NeRF-VPT

Also called: NeRF with View Prompt Tuning, NeRF-VPT with Cascading Learning, Neural Radiance Fields via View Prompt Tuning

## Overview
NeRF-VPT is a cascading view prompt tuning paradigm for novel view synthesis that progressively enhances image quality by using RGB information from preceding rendering iterations as visual prompts for subsequent stages [S1]. It solves the problem of limited scene understanding in sparse-view or standard NeRF settings by iteratively refining the network's output, leveraging prior knowledge sampled from the RGB space to guide optimization [S1]. Unlike methods that rely on external networks or specific data preprocessing, NeRF-VPT uses only the network's internal output as prior knowledge, making it a plug-and-play modification that can be integrated into variants like Mip-NeRF and TensoRF [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** Training images (RGB), camera poses, and the rendered RGB image from the previous stage (for stages $i > 0$).
*   **Outputs:** Volume density $\sigma$, color $\mathbf{c}$, and the rendered image $\hat{c}(r)$ for the current stage.

**Data Structures**
The framework consists of $N+1$ stages (Stage 0 to Stage $N$). Each stage $i$ utilizes a distinct neural network mapping function $F_i$.
*   **Stage 0:** A standard NeRF network.
*   **Stage $i$ ($i > 0$):** A new network instance. The input concatenation includes positional encoding $\text{PE}(\mathbf{x})$, direction encoding $\text{DE}(\mathbf{d})$, and the RGB prompt $\mathbf{c}(r)_{i-1}$ from the previous stage.
*   **Prompt Integration:** The RGB prompt is concatenated after the direction encoding. This placement is chosen because color is high-frequency data and encoding is viewed as signal decomposition; direction information induces diverse color variations for the same position after volume density is predicted [S1].

**Numbered Steps**
1.  **Initialize Stage 0:** Train a standard NeRF network without any prior input. This stage learns the basic 3D space representation and generates initial RGB values, which serve as the view prompt for Stage 1 [S1].
2.  **Iterate Stages 1 to $N$:** For each subsequent stage $i$:
    a.  **Ray Sampling:** Sample points $\mathbf{x}$ and directions $\mathbf{d}$ along rays.
    b.  **Encoding:** Encode the viewing direction $(\theta, \phi)$ using $\text{DE}(\mathbf{d})$ and position $\mathbf{x}$ using $\text{PE}(\mathbf{x})$ [S1].
    c.  **Retrieve Prompt:** Retrieve the RGB image $\mathbf{c}(r)_{i-1}$ rendered in the previous stage $i-1$.
    d.  **Concatenate:** Combine the encoded direction $\text{DE}(\mathbf{d})$ and the previous stage's RGB output $\mathbf{c}(r)_{i-1}$ as input to the current stage's network. The input to $F_i$ is $(\text{PE}(\mathbf{x}), (\text{DE}(\mathbf{d}), \mathbf{c}(r)_{i-1}))$ [S1].
    e.  **Predict:** The network $F_i$ outputs density $\sigma_i$ and color $\mathbf{c}_i$ for the current stage [S1].
    f.  **Render:** Compute the rendered color $\hat{c}(r)$ using volumetric rendering.
    g.  **Train:** Minimize the loss against ground truth images.
3.  **Termination:** Training concludes once the maximum number of stages is reached. Five to six stages typically achieve maximum performance [S1].

**Formulas**
The volumetric rendering equation for color $\hat{c}(r)$ is:
$$
\hat{c}(r) = \sum_{i=n}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$
where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ handles occlusion, $t_n, t_f$ are near/far boundaries, and $\delta_i$ is the distance between samples [S1].

The training loss is:
$$
L = \sum_{r \in R} ||\hat{c}(r) - \mathbf{c}(r)||
$$
where $R$ represents all image pixels in the training data and $\mathbf{c}(r)$ is the ground-truth color [S1].

The mapping function for iteration $i$ is defined as:
$$
(\mathbf{c}_i, \sigma_i) = F_i(\text{PE}(\mathbf{x}), (\text{DE}(\mathbf{d}), \mathbf{c}(r)_{i-1}))
$$
where $F_0$ is the Stage 0 mapping function and $F_i$ ($i>0$) is the mapping function for stage $i$ [S1].

**Parameters**
*   **Number of stages:** 6 stages (Stage 0 to Stage 6) are used in the standard implementation [S1].
*   **Prompt Integration Point:** Concatenated after direction encoding [S1].
*   **Prior Images (for ablation/experimentation):** 20 uniformly sampled training images were used in ablation studies for feature extraction via Swin-Transformer, but the core method uses rendered outputs [S1].

**Edge Cases and Pitfalls**
*   **Ground Truth Unavailability:** Directly providing ground truth images as priors fails during testing for unknown viewpoints because ground truth is unavailable. NeRF-VPT solves this by using rendered images from the previous stage as the prompt, which are available for all viewpoints [S1].
*   **Feature Fusion Limitations:** Extracting features from multiple images via Swin-Transformer or CNN and fusing them into an MLP showed limited improvement because it is challenging to encapsulate multi-view information within a single feature [S1].
*   **Training Time Scaling:** Training time per iteration decreases significantly as the cascade progresses because increasingly accurate visual prompts reduce the learning difficulty for the network [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_vpt_train(training_data, num_stages=6):
    """
    Training loop for NeRF-VPT.
    training_data: List of (image, pose) tuples.
    num_stages: Total number of stages (0 to num_stages-1).
    """
    # Stage 0: Standard NeRF
    # Initialize standard NeRF network F_0
    network_0 = StandardNeRF()
    network_0 = train_stage(network_0, training_data, stage_index=0)
    
    # Store the rendered images from Stage 0 to use as prompts for Stage 1
    # In practice, this requires rendering the training views using the trained network
    prompts = render_training_views(network_0, training_data)
    
    networks = [network_0]
    
    # Stages 1 to N
    for i in range(1, num_stages):
        # Initialize new network for stage i
        # Note: Loading weights from previous stage is an option for faster convergence [S1]
        network_i = NewNeRFNetwork()
        
        # Train stage i using the prompt from stage i-1
        # The prompt is concatenated with direction encoding
        network_i = train_stage_with_prompt(
            network_i, 
            training_data, 
            previous_prompt=prompts, 
            stage_index=i
        )
        
        networks.append(network_i)
        
        # Update prompts for the next stage using the newly trained network
        prompts = render_training_views(network_i, training_data)
        
        # Check for convergence or max stages
        if i >= num_stages - 1:
            break
            
    return networks

def train_stage_with_prompt(network, training_data, previous_prompt, stage_index):
    """
    Trains a single stage of NeRF-VPT.
    previous_prompt: Rendered RGB images from the previous stage.
    """
    for epoch in range(max_epochs):
        for batch in training_data:
            image_gt, pose = batch
            
            # Get corresponding prompt for this pose from previous stage
            prompt_rgb = get_prompt_for_pose(previous_prompt, pose)
            
            # Sample rays
            rays = sample_rays(pose, image_gt.shape)
            
            # Forward pass
            # Inputs: Position encoding, Direction encoding, Previous RGB prompt
            # The prompt is concatenated after direction encoding
            positions = rays.positions
            directions = rays.directions
            
            pe_pos = positional_encoding(positions)
            de_dir = directional_encoding(directions)
            
            # Concatenate direction encoding with prompt
            # Input to network: (PE(x), (DE(d), prompt_rgb))
            combined_dir_input = concatenate(de_dir, prompt_rgb)
            
            sigma, color = network(pe_pos, combined_dir_input)
            
            # Render
            rendered_color = volumetric_render(rays, sigma, color)
            
            # Loss
            loss = mse_loss(rendered_color, image_gt)
            
            # Backpropagate
            network.update(loss)
            
    return network

def render_training_views(network, training_data):
    """
    Renders the training views using the current network to generate prompts.
    """
    prompts = []
    for image_gt, pose in training_data:
        rays = sample_rays(pose, image_gt.shape)
        positions = rays.positions
        directions = rays.directions
        
        pe_pos = positional_encoding(positions)
        de_dir = directional_encoding(directions)
        
        # For Stage 0, there is no prompt. For subsequent stages, 
        # this function is called after training, so the network 
        # expects the prompt input. However, to generate the prompt 
        # for the *next* stage, we use the current network's output.
        # If this is Stage 0, we pass a dummy or empty prompt.
        # If this is Stage i > 0, we would need the prompt from Stage i-1 
        # to render, but here we are generating the prompt for Stage i+1.
        # So we use the prompt from Stage i-1 (which is already in the network's 
        # training context) or simply re-render using the network's learned 
        # parameters. 
        # To be precise: The prompt for Stage i is the output of Stage i-1.
        # So to generate prompts for Stage 2, we render Stage 1's output.
        
        # For Stage 0 rendering (to get prompts for Stage 1):
        if stage_index == 0:
            # Standard NeRF forward pass
            sigma, color = network(pe_pos, de_dir)
        else:
            # This function is called after training stage i.
            # To generate the prompt for stage i+1, we need to render 
            # using the network trained at stage i.
            # The network at stage i expects a prompt from stage i-1.
            # We must re-provide that prompt to render.
            # However, the prompt for stage i+1 is simply the output of stage i.
            # So we need to know what prompt was used to train stage i.
            # Let's assume we have access to the prompt used for stage i.
            prompt_for_i = get_prompt_for_stage(i-1)
            combined_dir_input = concatenate(de_dir, prompt_for_i)
            sigma, color = network(pe_pos, combined_dir_input)
            
        rendered = volumetric_render(rays, sigma, color)
        prompts.append(rendered)
    return prompts
```

## Subtopics

### Cascading Learning
A structure where each stage refines the visual prompts and updates network weights, progressively enhancing image quality [S1].

### View Prompt Tuning
The technique of using RGB information from preceding iterations as informative visual prompts to guide the network's understanding of the scene [S1].

## Operating characteristics
*   **Time Complexity:** Training time per iteration decreases significantly as the cascade progresses because increasingly accurate visual prompts reduce the learning difficulty for the network [S1].
*   **Memory Complexity:** Parameter overhead is negligible, with only 0.0008 M additional parameters compared to baseline NeRF [S1].
*   **Measured Performance:**
    *   **Rendering Speed:** 0.108 FPS on the Replica dataset, compared to 0.103 FPS for NeRF [S1].
    *   **Realistic Synthetic 360 Dataset (Stage 6):** PSNR 34.95, SSIM 0.977, LPIPS 0.042, outperforming NeRF (31.01, 0.947, 0.081) and Mip-NeRF (33.09, 0.961, 0.043) [S1].
    *   **Real Forward-Facing Dataset (Stage 6):** PSNR 28.95, SSIM 0.872, LPIPS 0.158, outperforming NeRF (26.50, 0.811, 0.250) and Mip-NeRF (27.13, 0.839, 0.164) [S1].
    *   **Replica Dataset (Sparse Views):** Training with 45 sparse views using NeRF-VPT achieves comparable image quality to training standard NeRF with 180 views [S1].
*   **Strengths:** Plug-and-play integration with other NeRF variants (Mip-NeRF, TensoRF); effective for sparse-view synthesis; reduces learning difficulty through guided optimization [S1].
*   **Limitations:** Requires multiple training stages (increased total training time compared to single-stage NeRF); depth prediction quality remains similar to baseline (MSE distance 0.02) despite image quality improvements [S1].

## Background
NeRF-VPT was introduced in 2024 by authors publishing in the Thirty-Eighth AAAI Conference on Artificial Intelligence [S1]. It builds on the success of NeRF and aims to enhance visual quality by incorporating effective guidance through cascading view prompt tuning [S1]. The method addresses limitations of prior approaches that relied on external networks or specific data preprocessing, offering a more transferable solution [S1].

## Sources
[S1] NeRF-VPT: Learning Novel View Representations with Neural Radiance Fields via View Prompt Tuning, https://doi.org/10.1609/aaai.v38i2.27877, fetched 2026-10-27
