# DONeRF

Also called: Depth Oracle Neural Radiance Fields, DONeRF Training Configuration, DONeRF training setup, Depth Oracle Network, DoNeRF

## Overview
DONeRF is a training configuration for compact neural radiance fields that replaces the standard dense raymarching of NeRF with a two-network pipeline: a depth oracle network and a shading network [S2]. It solves the prohibitive computational cost of NeRF by using the oracle to predict sample locations, allowing the shading network to render high-quality images with only 2–4 samples per ray instead of 256 [S2]. It is distinguished from its siblings by its non-end-to-end, two-stage training procedure and its reliance on a classification-based depth oracle rather than direct depth regression or coarse-to-fine cascades [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** RGBD images (color and depth) sampled from a "view cell" [S2]. A view cell is a bounding box with a primary orientation and maximum viewing angle, capturing all view rays originating within the box and staying within rotation limits [S2].
*   **Outputs:** RGBA color for the rendered image [S2].

**Data Structures:**
*   **Depth Oracle Network:** An MLP with 8 hidden layers and 256 hidden units [S2].
*   **Shading Network:** An MLP with 8 hidden layers and 256 hidden units [S2].
*   **View Cell Parameters:** Defined by $K$ (kernel size), $Z$ (z-range), and $I$ (input positions). Typical configuration: $K=5, Z=5, I=128$ [S2].
*   **Sample Count:** $N$ varies (e.g., DONeRF-2, DONeRF-4) [S2].

**Method Steps:**
The pipeline consists of five stages [S2]:

1.  **Ray Unification:** Ray descriptions are unified within the view cell to remove input ambiguity [S2].
2.  **Non-linear Depth Space:** Depth is considered in a non-linear space to handle large, open scenes [S2].
3.  **Oracle Classification:** The depth oracle network estimates the importance of samples. It learns a classification task using filtered, discretized target depth values rather than directly estimating continuous depth [S2].
4.  **Space Warping:** Sample positions predicted by the oracle are warped towards the view cell. This warping directs high-frequency predictions of the shading network to the foreground, better capturing different frequencies in the foreground and background [S2].
5.  **Shading Accumulation:** The shading network places a small number of samples ($N$) guided by the oracle and performs NeRF-like raymarching accumulation to deliver the final RGBA output [S2].

**Training Procedure:**
The architecture is not end-to-end trainable. Training occurs in two distinct phases [S2]:

1.  **Oracle Training:** Train the depth oracle network for 300,000 iterations. The loss is Binary Cross-Entropy (BCE) on the depth classification task [S2].
2.  **Shading Training:** Train the shading network for 300,000 iterations using the sample locations predicted by the frozen oracle. The loss is a weighted sum of MSE and an opacity penalty [S2].

**Formulas:**
The opacity loss ($\text{loss}_O$) prevents dark pixel artifacts caused by low sample counts where rays fail to accumulate to opacity 1 for opaque surfaces [S2]:

$$
\text{loss}_O = \begin{cases} 
0, & \text{if } \sum_{i=1}^X \delta_i \geq 1 \\
\left( \left( \sum_{i=1}^X \delta_i \right) - 1 \right)^2, & \text{otherwise}
\end{cases}
$$

Where $X$ is the number of samples per ray and $\delta_i$ is the opacity contribution of the $i$-th sample [S2].

The total loss for shading network training is:

$$
\text{loss} = \alpha \cdot \text{loss}_{\text{MSE}} + \beta \cdot \text{loss}_O
$$

Where $\text{loss}_{\text{MSE}}$ is the mean squared error between rendered and ground truth RGB, $\alpha = 1.0$, and $\beta = 10.0$ [S2].

**Parameters:**
*   **Optimizer:** Adam [S2].
*   **Learning Rate:** 0.0005 [S2].
*   **Batch Size:** 4096 samples per iteration [S2].
*   **Alpha ($\alpha$):** 1.0 [S2].
*   **Beta ($\beta$):** 10.0 [S2].
*   **Samples per Pixel:** 4 (standard DONeRF), though variants use 2, 8, or 16 [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_donerrf(oracle_net, shading_net, optimizer_oracle, optimizer_shading, 
                  dataloader, num_iterations_oracle=300000, num_iterations_shading=300000):
    """
    Two-stage training for DONeRF.
    oracle_net: Depth Oracle Network (8 layers, 256 units)
    shading_net: Shading Network (8 layers, 256 units)
    """
    
    # Stage 1: Train Depth Oracle
    for iteration in range(num_iterations_oracle):
        batch = next(dataloader)
        # Input: Ray descriptions unified within view cell
        # Target: Filtered, discretized depth values (classification)
        ray_inputs = batch.ray_inputs 
        depth_targets = batch.depth_targets
        
        # Forward pass through Oracle
        oracle_outputs = oracle_net(ray_inputs)
        
        # Loss: Binary Cross-Entropy for classification
        loss_oracle = bce_loss(oracle_outputs, depth_targets)
        
        # Backpropagate and update Oracle
        optimizer_oracle.zero_grad()
        loss_oracle.backward()
        optimizer_oracle.step()

    # Stage 2: Train Shading Network (Oracle is frozen)
    oracle_net.eval()
    shading_net.train()
    
    for iteration in range(num_iterations_shading):
        batch = next(dataloader)
        ray_inputs = batch.ray_inputs
        rgb_targets = batch.rgb_targets
        depth_targets = batch.depth_targets # Used to guide oracle, not direct shading loss
        
        # 1. Get sample locations from Oracle
        oracle_sample_locs = oracle_net(ray_inputs)
        
        # 2. Warp sample locations towards view cell (non-linear transformation)
        warped_locs = warp_samples(oracle_sample_locs, ray_inputs)
        
        # 3. Evaluate Shading Network at warped locations
        # shading_net takes position and view direction
        shading_outputs = shading_net(warped_locs, ray_inputs.view_directions)
        
        # 4. Accumulate RGBA (NeRF-like volume rendering)
        # alpha_i = 1 - exp(-sigma_i * delta_i)
        # color_i = alpha_i * color_i
        # C = sum(color_i * product(1 - alpha_j) for j < i)
        rendered_rgb, accumulated_opacity = volume_render(shading_outputs, ray_inputs.delta)
        
        # 5. Calculate Losses
        loss_mse = mse_loss(rendered_rgb, rgb_targets)
        
        # Opacity Loss: Force accumulated opacity >= 1 for opaque surfaces
        sum_opacity = accumulated_opacity.sum(dim=1) # Sum over samples per ray
        opacity_penalty = torch.where(
            sum_opacity >= 1.0,
            torch.tensor(0.0),
            (sum_opacity - 1.0) ** 2
        )
        loss_opacity = opacity_penalty.mean()
        
        # Total Loss
        alpha = 1.0
        beta = 10.0
        loss = alpha * loss_mse + beta * loss_opacity
        
        # Backpropagate and update Shading Network
        optimizer_shading.zero_grad()
        loss.backward()
        optimizer_shading.step()

def train_donerrf_no_gt(oracle_net, shading_net, dense_nerf, ...):
    """
    Training without ground truth depth.
    """
    # 1. Train a dense NeRF (12 layers, 512 units, 128 samples/ray)
    dense_nerf = train_dense_nerf(dense_nerf, dataloader)
    
    # 2. Extract depth maps for all poses from the trained dense NeRF
    depth_maps = extract_depth_from_nerf(dense_nerf, dataloader)
    
    # 3. Use extracted depth maps as ground truth for DONeRF training
    # Proceed with standard DONeRF training using these synthetic depth maps
    train_donerrf(oracle_net, shading_net, ..., depth_targets=depth_maps)
```

## Subtopics

### [DONeRF Oracle Input Construction](4737-donerf-oracle-input-construction.md)
Defines the input vector for the depth oracle network, using a classification head over discretized depth bins and injecting 3D spatial coordinates. Choose this when you need to provide geometric context to a lightweight MLP without relying solely on positional encoding.

## Operating characteristics
*   **Time Complexity:** DONeRF requires approximately 4.35 MFLOP per pixel [S2].
*   **Memory Complexity:** Compact; requires only two MLPs (approx. 4 MB) without additional data structures or acceleration structures like sparse voxel octrees [S2].
*   **Performance:** Renders at 22 ms per frame on a single GPU at 400x400 pixels (approx. 20 fps) [S2]. On NVIDIA RTX 3090 at 800x800 with 2 samples/ray, total time is 51.3 ms [S2].
*   **Scaling:** Shading network cost scales nearly linearly with samples: 34.9 ms (2 samples), 65.4 ms (4 samples), 136.4 ms (8 samples), 270.4 ms (16 samples) [S2].
*   **Quality:** Achieves an average PSNR of 31.14 dB compared to NeRF's 30.52 dB [S2]. At 16 samples, DONeRF can achieve up to 7 dB higher PSNR than NeRF [S2].
*   **Strengths:** Real-time rendering capability; significantly faster inference (up to 48x speedup with GT depth) than NeRF; compact memory footprint; better quality on large-scale scenes than NeRF [S2].
*   **Limitations:** Not end-to-end trainable; struggles without high-quality depth supervision; reflective/transparent surfaces are difficult to replicate with low sample counts and single depth estimates; background noise in no-GT training can cause artifacts [S1, S2].

## Background
DONeRF was introduced by [S2] (Graz University of Technology and Facebook Reality Labs) to address the prohibitive computational cost of NeRF in real-time applications like games and VR. It builds on the concept of using depth information to guide sampling, inspired by earlier works like MESK22 and ENeRF [S1]. The method was designed to render large-scale synthetic scenes in real-time while maintaining the compactness of neural representations [S2].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
