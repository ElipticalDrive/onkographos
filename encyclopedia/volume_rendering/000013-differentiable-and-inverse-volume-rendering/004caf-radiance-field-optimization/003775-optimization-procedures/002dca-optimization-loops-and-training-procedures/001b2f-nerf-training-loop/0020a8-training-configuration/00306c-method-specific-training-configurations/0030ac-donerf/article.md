# DONeRF

Also called: Depth Oracle Neural Radiance Fields, DONeRF Training Configuration, DONeRF training setup, Depth Oracle Network, DoNeRF

## Overview
DONeRF is a training configuration and architectural variant of NeRF that replaces the standard uniform raymarching with a two-network pipeline: a **depth oracle network** that predicts sample locations and a **shading network** that renders the final color using only those few samples [S3]. It solves the prohibitive computational cost of standard NeRF (which requires hundreds of samples per ray) by reducing the sample count to as few as 2–4 per ray, enabling real-time rendering on consumer GPUs [S3]. It sets itself apart from standard NeRF and other acceleration methods (like NSVF or NeX) by relying entirely on compact MLPs without explicit caching structures or acceleration trees, achieving higher quality and speed than NeRF at the same memory footprint [S3].

## How it works
DONeRF operates in a two-stage pipeline. First, a depth oracle network predicts where samples should be placed along the ray. Second, a shading network evaluates the radiance at these specific locations. The method is not end-to-end trainable; the oracle is trained first, and its predictions are used as fixed inputs to train the shading network [S3].

### Inputs and Outputs
*   **Inputs:** RGBD images (color and depth) sampled from a "view cell" [S3]. A view cell is a bounding box with a primary orientation and maximum viewing angle, capturing rays that originate within the box and stay within rotation limits (e.g., 30° pitch, 20° yaw) [S3].
*   **Outputs:** Rendered RGB color $\mathbf{c}$ and volume density $\sigma$ for the selected samples [S1].

### Data Structures
*   **View Cell:** Defines the spatial and angular bounds for training data. Rays are unified within this cell to reduce input ambiguity [S3].
*   **Depth Oracle Network:** An 8-layer MLP with 256 hidden units per layer, using ReLU activations [S1].
    *   **Input:** Concatenated 3D position $\mathbf{x} \in \mathbb{R}^3$ and view direction $\mathbf{d} \in S^2$. **No positional encoding** is applied [S1].
    *   **Output:** A classification vector representing discretized depth bins [S3].
*   **Shading Network:** An 8-layer MLP with 256 hidden units per layer, using ReLU activations [S1].
    *   **Input:** 3D position $\mathbf{x}$ and view direction $\mathbf{d}$.
    *   **Encoding:** Positional Encoding (PE) is applied to $\mathbf{x}$ and $\mathbf{d}$ to produce $\gamma(\mathbf{x})$ (63-dim) and $\gamma(\mathbf{d})$ (27-dim) [S1].
    *   **Architecture:** $\gamma(\mathbf{x})$ is fed into the first layer. $\gamma(\mathbf{d})$ is concatenated to the feature vector before the last layer [S1].
    *   **Output:** RGB color $\mathbf{c} \in \mathbb{R}^3$ and density $\sigma \in \mathbb{R}^+$ [S1].

### Numbered Steps

1.  **Ray Unification:** Transform ray origins and directions into the local coordinate system of the view cell [S3].
2.  **Depth Oracle Inference:**
    *   Concatenate the unified position $\mathbf{x}$ and direction $\mathbf{d}$.
    *   Pass through the Depth Oracle MLP.
    *   The output is a classification over discretized depth bins. These bins define the locations for the shading network samples [S3].
3.  **Sample Selection and Warping:**
    *   Select $N_s$ samples based on the oracle's classification (e.g., $N_s=4$ for DONeRF-4) [S3].
    *   Apply a **logarithmic transformation** to the sample depths to handle large open scenes [S3].
    *   **Warp** the sample positions towards the view cell center to better capture high-frequency details in the foreground [S3].
4.  **Shading Network Evaluation:**
    *   For each of the $N_s$ warped samples, compute $\gamma(\mathbf{x})$ and $\gamma(\mathbf{d})$ [S1].
    *   Query the Shading Network to get $\sigma_i$ and $\mathbf{c}_i$ for each sample $i$ [S1].
5.  **Volume Rendering:**
    *   Compute the accumulated color $\hat{C}$ using the standard differentiable volume rendering integral [S1]:
        $$ \hat{C} = \sum_{i=1}^{N_s} T_i \alpha_i \mathbf{c}_i $$
    *   Where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$, $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$, and $\delta_i$ is the distance between adjacent samples [S1].
6.  **Loss Computation:**
    *   Calculate Mean Squared Error (MSE) between $\hat{C}$ and ground truth color $C$ [S3].
    *   Calculate **Opacity Loss** ($\text{loss}_O$) to prevent dark pixel artifacts caused by low sample counts failing to accumulate to full opacity [S3]:
        $$ \text{loss}_O = \begin{cases} 0, & \text{if } \sum_{i=1}^{N_s} \delta_i \geq 1 \\ \left( \left( \sum_{i=1}^{N_s} \delta_i \right) - 1 \right)^2, & \text{otherwise} \end{cases} $$
        *Note: The source formula uses $O$ and $X$ interchangeably for sample count; here $N_s$ is used for consistency with the shading network input count [S3].*
    *   Total Loss:
        $$ \text{loss} = \alpha \cdot \text{loss}_{\text{MSE}} + \beta \cdot \text{loss}_O $$
        With $\alpha = 1.0$ and $\beta = 10.0$ [S3].
7.  **Optimization:**
    *   Update Depth Oracle weights using Binary Cross-Entropy (BCE) loss on the depth classification targets [S3].
    *   Update Shading Network weights using the Total Loss defined above [S3].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class DepthOracle(nn.Module):
    """
    8-layer MLP, 256 hidden units, ReLU.
    Input: x (3), d (3). No PE.
    Output: Classification logits for depth bins.
    """
    def __init__(self):
        super().__init__()
        self.layers = nn.ModuleList([
            nn.Linear(6, 256), # 3 for x, 3 for d
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256)
        ])
        self.output_layer = nn.Linear(256, num_depth_bins) # num_depth_bins depends on discretization

    def forward(self, x, d):
        # x: [B, 3], d: [B, 3]
        input_vec = torch.cat([x, d], dim=-1)
        h = input_vec
        for layer in self.layers:
            h = F.relu(layer(h))
        return self.output_layer(h)

class ShadingNetwork(nn.Module):
    """
    8-layer MLP, 256 hidden units, ReLU.
    Input: x (3), d (3). With PE.
    Output: sigma (1), color (3).
    """
    def __init__(self, num_freqs_pos=30, num_freqs_view=2):
        super().__init__()
        # Standard NeRF PE dimensions: 2*num_freqs*3 + 3
        self.dim_pos = 2 * num_freqs_pos * 3 + 3 # 63
        self.dim_view = 2 * num_freqs_view * 3 + 3 # 27
        
        self.pos_encoder = PositionalEncoding(num_freqs_pos)
        self.view_encoder = PositionalEncoding(num_freqs_view)
        
        self.layers = nn.ModuleList([
            nn.Linear(self.dim_pos, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256),
            nn.Linear(256, 256)
        ])
        self.output_layer = nn.Linear(256 + self.dim_view, 4) # sigma + rgb

    def forward(self, x, d):
        gamma_x = self.pos_encoder(x) # [B, 63]
        gamma_d = self.view_encoder(d) # [B, 27]
        
        h = F.relu(self.layers[0](gamma_x))
        for layer in self.layers[1:]:
            h = F.relu(layer(h))
        
        # Concatenate view direction encoding before last layer
        h = torch.cat([h, gamma_d], dim=-1)
        return self.output_layer(h)

def volume_render(sigma, color, deltas):
    """
    Standard differentiable volume rendering.
    sigma: [B, N]
    color: [B, N, 3]
    deltas: [B, N] distance between samples
    """
    alphas = 1.0 - torch.exp(-sigma * deltas)
    transmittance = torch.cumsum(deltas, dim=-1)
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # Using cumulative sum of sigma*deltas
    sigma_deltas = sigma * deltas
    T = torch.exp(-torch.cat([torch.zeros_like(sigma_deltas[:, :1]), sigma_deltas[:, :-1].cumsum(dim=-1)], dim=-1))
    
    weights = T * alphas
    color_out = torch.sum(weights.unsqueeze(-1) * color, dim=-2)
    return color_out

def train_donorf(depth_oracle, shading_net, optimizer_oracle, optimizer_shade, batch_size, num_iters):
    """
    Two-stage training loop.
    """
    # Stage 1: Train Depth Oracle
    for i in range(300000):
        # 1. Load batch of rays (x, d) and ground truth depth bins
        x, d, gt_depth_bins = load_batch(batch_size)
        
        # 2. Oracle Forward
        logits = depth_oracle(x, d)
        
        # 3. Loss: BCE for classification
        loss_oracle = F.binary_cross_entropy_with_logits(logits, gt_depth_bins)
        
        # 4. Update
        optimizer_oracle.zero_grad()
        loss_oracle.backward()
        optimizer_oracle.step()

    # Stage 2: Train Shading Network
    for i in range(300000):
        # 1. Load batch of rays (x, d) and ground truth colors
        x, d, gt_color = load_batch(batch_size)
        
        # 2. Oracle Inference to get sample locations
        # Note: In implementation, this involves decoding the classification logits 
        # to specific 3D coordinates along the ray, then warping them.
        sample_x_warped, sample_d_warped, deltas = get_warped_samples(depth_oracle, x, d)
        
        # 3. Shading Network Forward
        sigma, color = shading_net(sample_x_warped, sample_d_warped)
        # sigma is [B, N], color is [B, N, 3]
        
        # 4. Volume Render
        rendered_color = volume_render(sigma, color, deltas)
        
        # 5. Loss: MSE + Opacity Loss
        loss_mse = F.mse_loss(rendered_color, gt_color)
        
        # Opacity Loss: force accumulated opacity to be >= 1
        # alpha = 1 - exp(-sigma * delta)
        alphas = 1.0 - torch.exp(-sigma * deltas)
        accumulated_opacity = torch.sum(alphas, dim=-1) # [B]
        
        # loss_O = (sum(delta) - 1)^2 if sum(delta) < 1 else 0
        # Note: Source formula uses sum(delta) but context implies sum of alpha or similar 
        # to check if ray is opaque. However, source explicitly writes:
        # "if sum_{i=1}^O delta_i >= 1 ... loss is 0 ... else (sum delta - 1)^2"
        # This seems to refer to the accumulated optical depth or similar metric 
        # ensuring the ray travels far enough or is opaque. 
        # Given the text "forces the accumulated ray opacity delta to be at least 1", 
        # and the formula uses delta_i, we follow the source formula literally.
        # However, standard opacity is sum(alpha). The source text says "accumulated ray opacity delta".
        # Let's stick to the source formula: sum of deltas? No, that's distance.
        # Re-reading [S3]: "forces the accumulated ray opacity delta to be at least 1".
        # Formula: sum_{i=1}^X delta_i. This is likely a typo in the source for alpha or sigma*delta.
        # But we must follow the source. If the source says delta_i, we use delta_i.
        # Actually, looking at the context "dark pixel artifacts... rays may not accumulate to opacity 1",
        # it is highly likely the source meant alpha or transmittance. 
        # However, the formula explicitly uses delta_i. 
        # We will implement the formula as written in the source [S3].
        
        sum_deltas = torch.sum(deltas, dim=-1)
        opacity_loss = torch.where(
            sum_deltas >= 1.0,
            torch.tensor(0.0, device=deltas.device),
            (sum_deltas - 1.0)**2
        )
        
        alpha_weight = 1.0
        beta_weight = 10.0
        loss = alpha_weight * loss_mse + beta_weight * opacity_loss
        
        # 6. Update
        optimizer_shade.zero_grad()
        loss.backward()
        optimizer_shade.step()

class PositionalEncoding(nn.Module):
    def __init__(self, num_freqs):
        super().__init__()
        self.num_freqs = num_freqs
        
    def forward(self, x):
        # x: [B, 3]
        # Standard NeRF PE: sin(2^i * pi * x) for i in 0..num_freqs-1
        freq_bands = 2.0 ** torch.arange(self.num_freqs, device=x.device).float()
        scaled_x = x.unsqueeze(-1) * freq_bands * torch.pi
        s = torch.sin(scaled_x)
        c = torch.cos(scaled_x)
        encoded = torch.cat([s, c], dim=-1)
        return torch.cat([x, encoded], dim=-1)
```

### Parameters
*   **Network Depth:** 8 hidden layers for both networks [S3].
*   **Hidden Units:** 256 per layer [S3].
*   **Samples per Ray ($N_s$):** 2, 4, 8, or 16 (e.g., DONeRF-4 uses 4 samples) [S3].
*   **Optimizer:** Adam [S3].
*   **Learning Rate:** 0.0005 [S3].
*   **Batch Size:** 4096 samples per iteration [S3].
*   **Training Iterations:** 300,000 for Oracle, then 300,000 for Shading [S3].
*   **Loss Weights:** $\alpha = 1.0$ (MSE), $\beta = 10.0$ (Opacity) [S3].
*   **View Cell Rotation Limits:** 30° pitch, 20° yaw [S3].

### Edge Cases and Pitfalls
*   **Dark Pixel Artifacts:** Low sample counts can cause rays to fail to accumulate to full opacity, resulting in dark backgrounds. This is mitigated by the opacity loss term [S3].
*   **Cheating:** Shading networks may "cheat" by using opacity to mix background color. The opacity loss prevents this [S3].
*   **No Ground Truth Depth:** If GT depth is unavailable, a dense NeRF (12 layers, 512 units, 128 samples) is trained first to extract depth maps, which are then used to train the DONeRF oracle [S3]. This results in ~1 dB PSNR drop [S3].
*   **Reflective/Transparent Surfaces:** Single depth estimates are insufficient for mirrors or refractive water. DONeRF struggles with these view-dependent phenomena [S3].
*   **Background Noise:** In no-GT training, NeRF may output high-frequency depth in the background, leading to zero-contribution samples. Background detection during oracle training is suggested as a fix [S3].

## Subtopics

### [DONeRF Oracle Input Construction](4737-donerf-oracle-input-construction.md)
Defines the input vector construction for the depth oracle network, using discretized 3D positions and logarithmic transformations to align with the shading network's sampling strategy, rather than relying on positional encoding or single-depth predictions.

## Operating characteristics
*   **Time Complexity:** DONeRF requires approximately 4.35 MFLOP per pixel [S3].
*   **Memory Complexity:** Compact; only two MLPs (approx. 4 MB total), no explicit caching or acceleration structures [S3].
*   **Performance:** Renders at 22 ms per frame on a single GPU at 400x400 pixels (approx. 45 fps) [S3]. On NVIDIA RTX 3090 at 800x800 with 2 samples/ray, it takes 51.3 ms (~20 fps) [S3].
*   **Speedup:** Up to 48x faster than NeRF when conditioning on GT depth [S3].
*   **Quality:** Average PSNR of 31.14 dB compared to NeRF's 30.52 dB [S3]. At 16 samples, up to 7 dB higher PSNR than NeRF [S3].
*   **Strengths:** Real-time performance, high quality, compact memory footprint, no need for explicit acceleration structures [S3].
*   **Limitations:** Not end-to-end trainable, requires high-quality depth supervision (or a pre-trained NeRF), struggles with reflective/transparent surfaces, and view-dependent phenomena [S3].

## Background
DONeRF was introduced by [S3] (2021) to address the computational cost of NeRF, aiming for real-time rendering of compact neural representations. It was developed by researchers from Graz University of Technology and Facebook Reality Labs [S3]. It builds on the NeRF framework [S1] but diverges by using a depth oracle to guide sampling, reducing the number of samples per ray by up to 128x [S2].

## Sources
[S1] MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields, https://arxiv.org/abs/2310.01821v1, fetched 2026-10-26
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S3] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
