# NeRF-VPT

Also called: NeRF with View Prompt Tuning, NeRF-VPT with Cascading Learning, Neural Radiance Fields via View Prompt Tuning

## Overview
NeRF-VPT is a cascading view prompt tuning paradigm for novel view synthesis that progressively enhances image quality by using RGB information from preceding rendering iterations as visual prompts for subsequent stages [S1]. It solves the problem of limited scene understanding in sparse-view or standard NeRF settings by iteratively refining the network's output, leveraging the inherent representation of the NeRF family as prior knowledge [S1]. Unlike methods that rely on external networks or specific data preprocessing, NeRF-VPT uses only the network's internal output as prior knowledge, making it a plug-and-play module that can be integrated into existing NeRF variants like Mip-NeRF and TensoRF [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Training images (for supervision), ray origins $\mathbf{o}$ and directions $\mathbf{d}$, and the rendered RGB image from the previous stage $i-1$ (for $i > 0$).
*   **Outputs:** Predicted volume density $\sigma$ and view-dependent color $\mathbf{c}$ for each sampling point along the ray.

**Data Structures:**
The framework consists of $N+1$ stages, indexed from Stage 0 to Stage $N$. Each stage $i$ employs a distinct mapping function $F_i$ (typically an MLP).
*   **Stage 0:** A standard NeRF network.
*   **Stage $i$ ($i > 0$):** A new network instance. Its input layer accepts the positional encoding $\text{PE}(\mathbf{x})$, the directional encoding $\text{DE}(\mathbf{d})$, and the concatenated RGB prompt $\mathbf{c}(r)_{i-1}$.

**Numbered Steps:**
1.  **Initialize Stage 0:** Train a standard NeRF network $F_0$ using position $\mathbf{x}$ and direction $\mathbf{d}$ inputs without any prior knowledge. This stage learns the basic 3D space structure and generates the initial prior RGB values [S1].
2.  **Iterate for Stages $i = 1$ to $N$:**
    a.  **Ray Sampling:** For each pixel in the training images, sample points $\mathbf{x}_j$ along the ray $\mathbf{r} = (\mathbf{o}, \mathbf{d})$ between near $t_n$ and far $t_f$ boundaries [S1].
    b.  **Encoding:** Compute positional encoding $\text{PE}(\mathbf{x}_j)$ and directional encoding $\text{DE}(\mathbf{d})$ for each sample point [S1].
    c.  **Retrieve Prompt:** Obtain the rendered RGB image $\mathbf{c}(r)_{i-1}$ from the previous stage $i-1$. This image serves as the visual prompt [S1].
    d.  **Concatenate Prompt:** Concatenate the RGB prompt $\mathbf{c}(r)_{i-1}$ to the directional encoding $\text{DE}(\mathbf{d})$. The source notes that color is high-frequency data and encoding is viewed as signal decomposition, justifying this placement [S1].
    e.  **Network Forward Pass:** Input $\text{PE}(\mathbf{x}_j)$ and the combined vector $(\text{DE}(\mathbf{d}), \mathbf{c}(r)_{i-1})$ into the current stage's network $F_i$ to predict density $\sigma_i$ and color $\mathbf{c}_i$ [S1].
    f.  **Volume Rendering:** Render the image using the volumetric rendering equation [S1].
    g.  **Loss Calculation:** Compute the loss against ground truth images [S1].
    h.  **Optimization:** Update the weights of network $F_i$ [S1].
3.  **Termination:** Training concludes when the maximum number of stages (typically 5 or 6) is reached [S1].

**Formulas:**
The volumetric rendering equation for color $\hat{c}(\mathbf{r})$ is [S1]:
$$ \hat{c}(\mathbf{r}) = \sum_{j=1}^{M} T_j (1 - \exp(-\sigma_j \delta_j)) \mathbf{c}_j $$
where $T_j = \exp(-\sum_{k=1}^{j-1} \sigma_k \delta_j)$ handles occlusion, $\delta_j$ is the distance between samples, and $\sigma_j, \mathbf{c}_j$ are the density and color predicted by the current stage.

The mapping function for iteration $i$ is defined as [S1]:
$$ (\mathbf{c}_i, \sigma_i) = F_i(\text{PE}(\mathbf{x}), (\text{DE}(\mathbf{d}), \mathbf{c}(\mathbf{r})_{i-1})) $$
where $F_0$ and $F_i$ denote the mapping functions in iterations, $\text{PE}$ is position encoding, $\text{DE}$ is direction encoding, and $\mathbf{c}(\mathbf{r})_{i-1}$ is the RGB prompt from the previous stage. For Stage 0:
$$ (\mathbf{c}_0, \sigma_0) = F_0(\text{PE}(\mathbf{x}), \text{DE}(\mathbf{d})) $$

The training loss is [S1]:
$$ L = \sum_{\mathbf{r} \in R} ||\hat{c}(\mathbf{r}) - c(\mathbf{r})|| $$
where $R$ represents all image pixels in the training data, and $c(\mathbf{r})$ is the ground truth color.

**Parameters:**
*   **Number of Stages:** 5 to 6 stages on a standard baseline achieve maximum performance [S1].
*   **Prompt Integration Point:** The RGB prompt is concatenated after the direction encoding [S1].
*   **Prior Images (Alternative Approach):** The authors explored using 20 uniformly sampled training images to extract features via Swin-Transformer and fuse them via soft-view pooling, but found this yielded limited improvement compared to the cascading output method [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class NeRFVPT:
    def __init__(self, num_stages=6, near=2.0, far=6.0):
        self.num_stages = num_stages
        self.near = near
        self.far = far
        # Initialize Stage 0 as a standard NeRF
        self.stages = [StandardNeRF()]
        # Initialize subsequent stages
        for _ in range(1, num_stages):
            self.stages.append(StageNeRF())

    def render_stage(self, stage_idx, rays_o, rays_d, gt_images):
        """
        Render images for a specific stage.
        rays_o: (B, H, W, 3) ray origins
        rays_d: (B, H, W, 3) ray directions
        gt_images: (B, H, W, 3) ground truth images
        """
        # Sample points along rays
        t_vals = torch.linspace(self.near, self.far, steps=128)
        pts = rays_o[..., None, :] + rays_d[..., None, :] * t_vals[..., None]
        
        # Encode positions and directions
        # PE and DE are positional and directional encoding functions
        pts_enc = positional_encoding(pts)
        dirs_enc = directional_encoding(rays_d)
        
        # Get previous stage's rendered image as prompt
        if stage_idx == 0:
            prompt = None
        else:
            # Retrieve rendered image from previous stage
            # In a full implementation, this would be the output of the previous stage's render
            prompt = self.previous_rendered_images[stage_idx - 1]
            # Broadcast prompt to match ray batch if necessary
            prompt = prompt.unsqueeze(1).expand(-1, pts.shape[1], -1, -1)
            
        # Concatenate prompt to direction encoding
        if prompt is not None:
            # Concatenate along the feature dimension
            # Assuming dirs_enc has shape (B, H, W, D_dir) and prompt has (B, H, W, 3)
            combined_dirs = torch.cat([dirs_enc, prompt], dim=-1)
        else:
            combined_dirs = dirs_enc
            
        # Forward pass through the stage's MLP
        # pts_enc: (B, H, W, N_pts, D_pos)
        # combined_dirs: (B, H, W, N_pts, D_dir + 3) or (B, H, W, N_pts, D_dir)
        sigma, color = self.stages[stage_idx](pts_enc, combined_dirs)
        
        # Volume Rendering
        # Calculate deltas
        deltas = t_vals[1:] - t_vals[:-1]
        deltas = torch.cat([deltas, torch.zeros_like(deltas[..., :1])], dim=-1)
        
        # Calculate transmittance
        alpha = 1.0 - torch.exp(-torch.relu(sigma) * deltas)
        transmittance = torch.cumprod(torch.cat([torch.ones_like(alpha[..., :1]), 1.0 - alpha], dim=-1), dim=-1)[..., :-1]
        
        # Render color
        weights = alpha * transmittance
        rendered_color = torch.sum(weights[..., None] * color, dim=-2)
        
        # Compute Loss
        loss = torch.mean((rendered_color - gt_images) ** 2)
        
        # Store rendered image for next stage prompt
        self.previous_rendered_images[stage_idx] = rendered_color.detach()
        
        return loss, rendered_color

    def train(self, training_data):
        """
        training_data: list of (rays_o, rays_d, gt_images) batches
        """
        # Stage 0: Standard NeRF training
        for batch in training_data:
            rays_o, rays_d, gt_images = batch
            loss, _ = self.render_stage(0, rays_o, rays_d, gt_images)
            # Backpropagate and update Stage 0 weights
            self.stages[0].optimizer.step()
            
        # Cascading Stages 1 to N
        for stage_idx in range(1, self.num_stages):
            # Load weights from previous stage to facilitate convergence [S1]
            # Note: The source mentions loading weights is an option, 
            # but structurally each stage is a new network. 
            # Implementation detail: Copy weights from stage_idx-1 to stage_idx if desired.
            
            for batch in training_data:
                rays_o, rays_d, gt_images = batch
                loss, _ = self.render_stage(stage_idx, rays_o, rays_d, gt_images)
                # Backpropagate and update Stage stage_idx weights
                self.stages[stage_idx].optimizer.step()
```

**Edge Cases and Pitfalls:**
*   **Ground Truth Unavailability:** Directly providing ground truth images as priors fails during testing for unknown viewpoints because ground truth is unavailable [S1]. NeRF-VPT solves this by using rendered images from a pre-trained model as the prompt, which are available for all viewpoints [S1].
*   **Feature Fusion Limitations:** Extracting features from multiple images via Swin-Transformer or CNN and fusing them into an MLP showed limited improvement because it is challenging to encapsulate multi-view information within a single feature [S1].
*   **Training Time Scaling:** Training time per iteration decreases significantly as the cascade progresses because increasingly accurate visual prompts reduce the learning difficulty for the network [S1].

## Operating characteristics
*   **Time Complexity:** Training time increases with the number of stages, but the time per stage decreases as the prompts become more accurate [S1].
*   **Memory Complexity:** Parameter overhead is negligible, adding only 0.0008 M additional parameters compared to baseline NeRF [S1].
*   **Measured Performance:**
    *   On the Realistic Synthetic 360 Dataset, NeRF-VPT (Stage 6) achieves PSNR 34.95, SSIM 0.977, LPIPS 0.042, outperforming NeRF (31.01, 0.947, 0.081) and Mip-NeRF (33.09, 0.961, 0.043) [S1].
    *   On the Real Forward-Facing Dataset, NeRF-VPT (Stage 6) achieves PSNR 28.95, SSIM 0.872, LPIPS 0.158, outperforming NeRF (26.50, 0.811, 0.250) and Mip-NeRF (27.13, 0.839, 0.164) [S1].
    *   On the Replica Dataset, training with 45 sparse views using NeRF-VPT achieves comparable image quality to training standard NeRF with 180 views [S1].
    *   Rendering speed on the Replica dataset is 0.108 FPS, compared to 0.103 FPS for NeRF [S1].
*   **Strengths:** Plug-and-play integration with other NeRF variants (Mip-NeRF, TensoRF); significant improvement in sparse-view synthesis; maintains depth prediction quality (MSE distance 0.02, identical to baseline NeRF) [S1].
*   **Limitations:** Requires multiple training stages, increasing total training time compared to a single-stage NeRF; depth estimation does not improve significantly over baseline [S1].

## Background
NeRF-VPT was introduced in 2024 in the paper "NeRF-VPT: Learning Novel View Representations with Neural Radiance Fields via View Prompt Tuning" [S1]. It builds on the standard NeRF architecture [S1] and addresses limitations in prior methods that relied on external networks or specific data preprocessing for priors [S1]. The method draws inspiration from cascade structures used in other computer vision applications such as object detection, super-resolution, and pose estimation [S1].

## Sources
[S1] NeRF-VPT: Learning Novel View Representations with Neural Radiance Fields via View Prompt Tuning, https://doi.org/10.1609/aaai.v38i2.27877, fetched 2026-10-27
