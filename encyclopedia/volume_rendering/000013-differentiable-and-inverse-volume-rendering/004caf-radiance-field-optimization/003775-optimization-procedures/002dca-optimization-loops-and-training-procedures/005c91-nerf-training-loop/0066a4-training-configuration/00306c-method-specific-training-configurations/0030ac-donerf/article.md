# DONeRF

Also called: Depth Oracle Neural Radiance Fields, DONeRF Training Configuration, DONeRF training setup, Depth Oracle Network, DoNeRF

## Overview
DONeRF is a training configuration and architectural variant of Neural Radiance Fields (NeRF) designed for real-time rendering by replacing standard dense raymarching with a two-network local sampling strategy [S3]. It solves the prohibitive computational cost of NeRF by using a **depth oracle network** to predict optimal sample locations and a **shading network** to render the image using only a few samples per ray [S2, S3]. Unlike standard NeRF which uses hundreds of samples, DONeRF achieves equal or better quality with 2–4 samples per pixel, reducing inference costs by up to 48x compared to NeRF when ground truth depth is available [S3]. It sets itself apart from sibling methods like [NeRF Hyperparameters](3065-nerf-hyperparameters.md) by explicitly decoupling geometry estimation (oracle) from appearance rendering (shading) and training them sequentially rather than end-to-end [S2, S3].

## How it works
DONeRF consists of two Multi-Layer Perceptrons (MLPs): a depth oracle network and a shading network. The pipeline operates in five stages: (1) ray unification within a view cell, (2) non-linear depth transformation, (3) oracle classification of sample importance, (4) warping of sample positions, and (5) shading network raymarching [S3].

### Inputs and Outputs
*   **Inputs:** Camera position, viewing direction, and optionally ground truth depth maps (or extracted depth maps for training without GT).
*   **Outputs:** RGB color and accumulated opacity for each pixel.

### Data Structures and Network Layouts

**1. View Cell**
A view cell is a bounding box with a primary orientation and maximum viewing angle, capturing all view rays originating in the box within rotation limits [S3]. Training is performed per view cell [S3].

**2. Depth Oracle Network**
*   **Architecture:** 8 hidden layers, 256 hidden units per layer, ReLU activation. The last layer is linear.
*   **Input:** Concatenated 3D position $x \in \mathbb{R}^3$ and view direction $d \in S^2$.
*   **Encoding:** No positional encoding is applied to inputs [S1].
*   **Output:** Depth classification logits. The network solves a classification task using filtered, discretized target depth values rather than direct depth regression [S3].
*   **Loss:** Binary Cross-Entropy (BCE) loss [S3].

**3. Shading Network**
*   **Architecture:** 8 hidden layers, 256 hidden units per layer, ReLU activation. The last layer is linear.
*   **Input:** 3D position $x$ and view direction $d$.
*   **Encoding:** Positional encoding is applied.
    *   $\gamma(x)$: 63-dimensional vector.
    *   $\gamma(d)$: 27-dimensional vector.
*   **Usage:** $\gamma(x)$ is used at the first layer. $\gamma(d)$ is concatenated to the feature vector before the last layer [S1].
*   **Output:** RGB color $c \in \mathbb{R}^3$ and volume density $\sigma \in \mathbb{R}^+$ [S1].
*   **Loss:** Mean Squared Error (MSE) loss on RGB output [S3].

### Training Procedure

The training is **not end-to-end**. It proceeds in two sequential phases [S3]:

**Phase 1: Oracle Training**
1.  Train the depth oracle network for 300,000 iterations.
2.  Use ground truth depth maps (or extracted depth maps if GT is unavailable) to create classification targets.
3.  Optimizer: Adam, learning rate 0.0005, batch size 4096 samples per iteration [S3].

**Phase 2: Shading Network Training**
1.  Freeze the depth oracle network.
2.  Use the oracle to predict sample locations for the shading network.
3.  Train the shading network for 300,000 iterations.
4.  Optimizer: Adam, learning rate 0.0005, batch size 4096 samples per iteration [S3].

### Sampling and Warping

1.  **Ray Unification:** Rays are unified within the view cell [S3].
2.  **Logarithmic Transformation:** Sample positions are transformed logarithmically to handle large depth ranges [S3].
3.  **Oracle Selection:** The oracle classifies depth ranges. The number of samples selected by the oracle is denoted as $N_s$ (e.g., DONeRF-16 uses $N_s=16$) [S1].
4.  **Warping:** Sample positions are warped towards the view cell to direct high-frequency predictions to the foreground [S3].
5.  **Shading:** The shading network evaluates the warped samples to accumulate RGBA output [S3].

### Loss Functions

The total loss for the shading network training combines MSE and an opacity penalty [S3]:

$$ \text{loss} = \alpha \cdot \text{loss}_{\text{MSE}} + \beta \cdot \text{loss}_{\text{O}} $$

Where:
*   $\alpha = 1.0$
*   $\beta = 10.0$
*   $\text{loss}_{\text{MSE}}$ is the standard MSE between rendered and ground truth RGB.
*   $\text{loss}_{\text{O}}$ is the opacity loss, defined as:

$$ \text{loss}_O = \begin{cases} 0, & \text{if } \sum_{i=1}^X \delta_i \geq 1 \\ \left( \left( \sum_{i=1}^X \delta_i \right) - 1 \right)^2, & \text{otherwise} \end{cases} $$

Here, $X$ is the number of samples per ray, and $\delta_i$ represents the opacity contribution of the $i$-th sample [S3]. This loss prevents "dark pixel" artifacts caused by low sample counts failing to accumulate to full opacity for opaque surfaces [S3].

### Training Without Ground Truth Depth (DONeRF-X-noGT)

If ground truth depth is unavailable:
1.  Train a dense NeRF with 12 layers of 512 hidden units each, using 128 samples per ray [S3].
2.  Extract depth maps for all training poses from this dense NeRF [S3].
3.  Use these extracted depth maps as the ground truth for training the DONeRF oracle and shading networks in the standard two-phase procedure [S3].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

# Standard NeRF positional encoding parameters
NUM_BANDS = 10
MIN_FREQ_LOG = 0.0
MAX_FREQ_LOG = 10.0

def positional_encoding(x, min_freq_log, max_freq_log, num_bands):
    """
    Applies positional encoding to input x.
    x shape: (..., 3) for positions, (..., 3) for directions.
    Returns encoded vector.
    """
    l = min_freq_log
    b = (max_freq_log - min_freq_log) / (num_bands - 1)
    
    scales = torch.tensor([2**i for i in range(num_bands)])
    scales = scales.to(x.device).view(1, -1, 1)
    
    xb = torch.reshape(x, (x.shape[:-1] + (-1, 4))).float() * scales
    fouriered = torch.sin(torch.cat([xb, xb + 0.5 * torch.pi], dim=-1))
    
    # Concatenate original x with fouriered components
    # Original x is 3 dims. Fouriered is 2 * 3 * num_bands = 60 dims.
    # Total 63 dims for position.
    # For direction, same logic, but typically only direction is encoded in NeRF variants.
    # DONeRF Shading Network encodes both x and d.
    return torch.cat([x, fouriered], dim=-1)

class DepthOracleNet(nn.Module):
    def __init__(self):
        super().__init__()
        # 8 hidden layers, 256 units, ReLU. Last layer linear.
        # Input: x (3) + d (3) = 6 dims. No positional encoding.
        layers = []
        layers.append(nn.Linear(6, 256))
        layers.append(nn.ReLU())
        for _ in range(6): # 6 more hidden layers
            layers.append(nn.Linear(256, 256))
            layers.append(nn.ReLU())
        layers.append(nn.Linear(256, 1)) # Output: depth classification logit
        self.net = nn.Sequential(*layers)

    def forward(self, x, d):
        # x: (B, 3), d: (B, 3)
        input_vec = torch.cat([x, d], dim=-1)
        return self.net(input_vec)

class ShadingNet(nn.Module):
    def __init__(self):
        super().__init__()
        # 8 hidden layers, 256 units, ReLU. Last layer linear.
        # Input x encoded to 63 dims. Input d encoded to 27 dims.
        # gamma(x) used at first layer. gamma(d) concatenated before last layer.
        
        # First layer: gamma(x) -> 256
        self.first_layer = nn.Linear(63, 256)
        
        # Hidden layers 2-8: 256 -> 256
        self.hidden_layers = nn.ModuleList([
            nn.Sequential(nn.Linear(256, 256), nn.ReLU()) for _ in range(6)
        ])
        
        # Before last layer: concat gamma(d) (27 dims)
        # Feature dim becomes 256 + 27 = 283
        self.pre_output_layer = nn.Linear(283, 256)
        
        # Last layer: 256 -> 4 (RGB + sigma)
        self.output_layer = nn.Linear(256, 4)

    def forward(self, x, d):
        # x: (B, 3), d: (B, 3)
        gamma_x = positional_encoding(x, MIN_FREQ_LOG, MAX_FREQ_LOG, NUM_BANDS)
        gamma_d = positional_encoding(d, MIN_FREQ_LOG, MAX_FREQ_LOG, NUM_BANDS)
        
        # First layer uses gamma(x)
        h = self.first_layer(gamma_x)
        
        # Hidden layers
        for layer in self.hidden_layers:
            h = layer(h)
            
        # Concat gamma(d) before last layer
        h = torch.cat([h, gamma_d], dim=-1)
        h = self.pre_output_layer(h)
        
        # Output
        out = self.output_layer(h)
        # out[..., :3] is RGB, out[..., 3] is sigma
        return out

def opacity_loss(deltas):
    """
    deltas: (B, X) opacity contributions of X samples per ray.
    """
    sum_opacity = torch.sum(deltas, dim=-1)
    # If sum >= 1, loss is 0. Else (sum - 1)^2.
    # Note: The formula in [S3] uses sum(delta_i) - 1. 
    # Usually delta_i in volume rendering is alpha * exp(-sum(sigma)). 
    # Here we assume deltas are the accumulated opacity contributions.
    mask = (sum_opacity < 1.0).float()
    loss = mask * (sum_opacity - 1.0)**2
    return torch.mean(loss)

def train_donerrf(oracle_net, shading_net, optimizer_o, optimizer_s, 
                  dataloader, num_iter_o, num_iter_s, alpha, beta):
    """
    Two-phase training for DONeRF.
    """
    
    # Phase 1: Train Oracle
    oracle_net.train()
    shading_net.eval() # Freeze shading net
    
    for i in range(num_iter_o):
        # Get batch: rays (x, d), ground truth depth (for classification targets), GT RGB
        rays, gt_depth, gt_rgb = next(dataloader)
        
        # Oracle predicts depth logits
        pred_depth_logits = oracle_net(rays.x, rays.d)
        
        # Loss is BCE against discretized depth targets derived from gt_depth
        # [S3] states oracle solves classification task.
        loss_o = criterion_bce(pred_depth_logits, gt_depth_targets)
        
        optimizer_o.zero_grad()
        loss_o.backward()
        optimizer_o.step()
        
    # Phase 2: Train Shading
    oracle_net.eval()
    shading_net.train()
    
    for i in range(num_iter_s):
        rays, gt_rgb = next(dataloader)
        
        # 1. Oracle predicts sample locations (indices or warped positions)
        # [S3] Oracle provides sample locations.
        sample_locs = oracle_net.predict_samples(rays.x, rays.d)
        
        # 2. Warp samples
        warped_locs = warp_samples(sample_locs, rays.view_cell)
        
        # 3. Evaluate Shading Network at warped locations
        # shading_net returns RGB and sigma
        rgb_preds, sigma_preds = shading_net(warped_locs, rays.d)
        
        # 4. Volume Rendering Accumulation
        # Standard NeRF accumulation using sigma and RGB
        # [S3] Shading network places small number of samples guided by oracle.
        # Accumulate RGBA.
        # Note: Implementation of volume rendering from sigma/RGB is standard.
        rendered_rgb, deltas = volume_render(warped_locs, sigma_preds, rgb_preds, rays.t_near, rays.t_far)
        
        # 5. Compute Loss
        loss_mse = criterion_mse(rendered_rgb, gt_rgb)
        loss_op = opacity_loss(deltas)
        loss = alpha * loss_mse + beta * loss_op
        
        optimizer_s.zero_grad()
        loss.backward()
        optimizer_s.step()

# Configuration Constants
NUM_ITERATIONS = 300000
LEARNING_RATE = 0.0005
BATCH_SIZE = 4096
ALPHA = 1.0
BETA = 10.0
NUM_SAMPLES_PER_RAY = [2, 4, 8, 16] # Varies by variant (DONeRF-2, etc.)
```

## Subtopics

### [DONeRF Oracle Input Construction](4737-donerf-oracle-input-construction.md)
Constructs the input vector for the depth oracle network, providing spatial context by supplying multiple 3D positions along the ray. It sets itself apart by explicitly feeding discretized 3D positions with a logarithmic transformation aligned to the shading network's sampling strategy, omitting positional encoding. Choose this when implementing the specific input pipeline for the oracle network.

## Operating characteristics
*   **Time Complexity:** DONeRF requires approximately 4.35 MFLOP per pixel [S3]. Inference costs are reduced by up to 48x compared to NeRF when conditioning on available ground truth depth information [S3].
*   **Memory Complexity:** DONeRF stays in the realm of compact MLPs without additional data structures or increased storage requirements, unlike baking methods which may require hundreds of MBs to GBs [S3]. It does not require additional memory for explicit caching or acceleration structures, unlike methods such as NSVF [S3].
*   **Measured Performance:** On an NVIDIA RTX 3090 at 800x800 resolution with 2 samples/ray, DONeRF renders in 51.3 ms (~20 fps) [S3]. Breakdown: Ray unification/first feature gen (0.21 ms), Oracle network (12.64 ms), Space warping/second feature gen (3.46 ms), Shading network (34.9 ms), Final color gen (0.09 ms) [S3]. The shading network cost scales nearly linearly with samples: 34.9 ms (2 samples), 65.4 ms (4 samples), 136.4 ms (8 samples), 270.4 ms (16 samples) [S3].
*   **Quality:** DONeRF achieves an average PSNR of 31.14 dB compared to NeRF's 30.52 dB [S3]. At 16 samples, DONeRF is up to 7 dB higher PSNR and 15-78x faster than NeRF [S3].
*   **Strengths:** Real-time rendering capability, compact memory footprint, high quality on large-scale scenes.
*   **Limitations:** Not end-to-end trainable [S2]. Struggles without high-quality depth supervision [S2]. Reflective/transparent surfaces are difficult to replicate with low sample counts and single depth estimates [S3]. Dark pixel artifacts can occur at low sample counts if opacity loss is not applied [S3]. Background noise in no-GT training can create zero-contribution samples [S3].

## Background
DONeRF was introduced by [S3] to bring compact neural representations closer to practical real-time rendering in applications like games and VR. It addresses the prohibitive computational cost of NeRF, which requires dozens of petaFLOPS. The method was developed to enable interactive rendering (20 fps) on a single GPU. It is used for real-time rendering of large synthetic scenes [S3].

## Sources
[S1] MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields, https://arxiv.org/abs/2310.01821v1, fetched 2026-10-26
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S3] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
