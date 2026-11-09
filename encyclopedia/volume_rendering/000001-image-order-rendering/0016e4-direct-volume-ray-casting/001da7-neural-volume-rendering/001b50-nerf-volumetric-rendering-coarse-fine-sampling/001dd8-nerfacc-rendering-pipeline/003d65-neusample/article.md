# NeuSample

Also called: Neural Sample Field, Sample Field Extraction

## Overview
NeuSample is a neural volume rendering method that replaces the coarse-to-fine hierarchical sampling stage of Neural Radiance Fields (NeRF) with a lightweight "neural sample field" [S1, unofficial]. It maps a camera ray directly into a distribution of sample points, enabling single-shot inference without the intermediate coarse pass [S1, unofficial]. This approach reduces inference time by approximately 25% compared to standard NeRF while maintaining or improving rendering quality [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Camera ray defined by origin $o = (x_o, y_o, z_o)$ and normalized direction $d = (\theta, \phi)$ (or unit vector), near bound $t_n$, far bound $t_f$, and number of samples $N$.
*   **Output:** $N$ 3D sample coordinates $\{x_1, ..., x_N\}$ along the ray, which are fed into a radiance field $R_\Theta$ to produce density $\sigma$ and color $c$, resulting in a final pixel color via volume rendering.

**Data Structures:**
*   **Neural Sample Field ($S_\Theta$):** A Multi-Layer Perceptron (MLP).
    *   Input dimension: 6 (3 for origin, 3 for direction) before encoding.
    *   Hidden layers: 8 Fully Connected (FC) layers with ReLU activation.
    *   Hidden dimension: 256.
    *   Skip connection: Input concatenated with output of the 4th layer.
    *   Output layer: 1 FC layer mapping 256 dimensions to $N$ dimensions, followed by Sigmoid activation.
*   **Radiance Field ($R_\Theta$):** Standard NeRF MLP (not detailed in NeuSample source, assumed standard NeRF architecture [S1, unofficial]).

**Parameters:**
*   $N$: Number of samples. The source uses $N=192$ for the main experiments [S1, unofficial]. This replaces the sum of coarse ($N_c$) and fine ($N_f$) samples in NeRF.
*   Position Encoding: 10-frequency positional encoding applied to both origin and direction [S1, unofficial].

**Algorithm Steps:**

1.  **Ray Generation:** Compute ray origin $o$ and direction $d$ for the target pixel from the camera pose.
2.  **Position Encoding:** Apply 10-frequency positional encoding $\tilde{\tau}$ to $o$ and $d$. Concatenate the encoded origin and direction to form the input vector $v$.
3.  **Sample Field Inference:** Pass $v$ through $S_\Theta$:
    *   Pass through 8 FC layers with ReLU.
    *   Concatenate original input $v$ with the output of the 4th layer (skip connection).
    *   Pass through final FC layer to get $N$-dimensional vector.
    *   Apply Sigmoid activation to obtain relative positions $\hat{t}_1, ..., \hat{t}_N \in [0, 1]$.
4.  **Coordinate Transformation:** Map relative positions to absolute distances $t_i$ using near/far bounds:
    $$ t_i = (1 - \hat{t}_i)t_n + \hat{t}_i t_f $$
    Compute 3D sample coordinates:
    $$ x_i = o + t_i d $$
5.  **Radiance Field Query:** For each $x_i$, query $R_\Theta(x_i, d)$ to obtain density $\sigma_i$ and color $c_i$.
6.  **Volume Rendering:** Accumulate color using standard volume rendering equation:
    $$ C(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) c_i $$
    where $\delta_i = t_{i+1} - t_i$ (with $t_{N+1} = t_f$), and $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

def positional_encoding(x, num_frequencies=10):
    """
    Standard NeRF positional encoding.
    x: tensor of shape (..., 3)
    Returns: tensor of shape (..., 3 + 3*2*num_frequencies)
    """
    # x is assumed to be in radians or normalized coordinates depending on context.
    # NeRF uses sin/cos of frequencies 2^i * pi * x.
    # Here we implement the standard encoding used in NeRF [S1, unofficial].
    scales = torch.tensor([2**i for i in range(num_frequencies)], dtype=x.dtype, device=x.device)
    scaled_x = torch.flatten(x[..., None, :] * scales[None, None, :, None], -2)
    encoded = torch.cat([x, torch.sin(scaled_x), torch.cos(scaled_x)], dim=-1)
    return encoded

class NeuralSampleField(nn.Module):
    def __init__(self, N=192):
        super().__init__()
        self.N = N
        # Input dim: 6 (3 origin + 3 direction)
        # After PE: 6 + 6*2*10 = 126
        self.input_dim = 6 + 6 * 2 * 10
        self.hidden_dim = 256
        
        # 8 FC layers with ReLU
        self.fc1 = nn.Linear(self.input_dim, self.hidden_dim)
        self.fc2 = nn.Linear(self.hidden_dim, self.hidden_dim)
        self.fc3 = nn.Linear(self.hidden_dim, self.hidden_dim)
        self.fc4 = nn.Linear(self.hidden_dim, self.hidden_dim)
        self.fc5 = nn.Linear(self.hidden_dim, self.hidden_dim)
        self.fc6 = nn.Linear(self.hidden_dim, self.hidden_dim)
        self.fc7 = nn.Linear(self.hidden_dim, self.hidden_dim)
        self.fc8 = nn.Linear(self.hidden_dim, self.hidden_dim)
        
        # Skip connection from input to after 4th layer
        # Concatenation increases dim from 256 to 256 + input_dim
        self.fc9 = nn.Linear(self.hidden_dim + self.input_dim, self.hidden_dim)
        
        # Output layer: 256 -> N
        self.fc_out = nn.Linear(self.hidden_dim, self.N)
        
    def forward(self, origin, direction):
        # origin: (B, 3), direction: (B, 3)
        # Concatenate origin and direction
        x = torch.cat([origin, direction], dim=-1)
        
        # Apply positional encoding
        x_enc = positional_encoding(x, num_frequencies=10)
        
        # FC layers 1-4
        h1 = torch.relu(self.fc1(x_enc))
        h2 = torch.relu(self.fc2(h1))
        h3 = torch.relu(self.fc3(h2))
        h4 = torch.relu(self.fc4(h3))
        
        # Skip connection: concat input with h4
        h_skip = torch.cat([x_enc, h4], dim=-1)
        
        # FC layers 5-8 (applied to skip connection)
        h5 = torch.relu(self.fc5(h_skip))
        h6 = torch.relu(self.fc6(h5))
        h7 = torch.relu(self.fc7(h6))
        h8 = torch.relu(self.fc8(h7))
        
        # Final output layer
        logits = self.fc_out(h8)
        
        # Sigmoid to get relative positions in [0, 1]
        relative_t = torch.sigmoid(logits)
        
        return relative_t

def render_neusample(ray_origins, ray_directions, near_bound, far_bound, N=192, radiance_field=None):
    """
    ray_origins: (B, 3)
    ray_directions: (B, 3)
    near_bound: float
    far_bound: float
    radiance_field: callable R_theta(x, d) -> (sigma, color)
    """
    sample_field = NeuralSampleField(N=N)
    
    # 1. Get relative positions
    relative_t = sample_field(ray_origins, ray_directions) # (B, N)
    
    # 2. Map to absolute distances
    # t_i = (1 - t_hat_i) * tn + t_hat_i * tf
    absolute_t = (1 - relative_t) * near_bound + relative_t * far_bound # (B, N)
    
    # 3. Compute 3D coordinates
    # x_i = o + t_i * d
    samples = ray_origins.unsqueeze(1) + absolute_t.unsqueeze(2) * ray_directions.unsqueeze(1) # (B, N, 3)
    
    # 4. Query Radiance Field
    # Reshape for batched query if necessary, assuming radiance_field handles (B*N, 3) and (B*N, 3)
    B, N, _ = samples.shape
    samples_flat = samples.view(-1, 3)
    dirs_flat = ray_directions.unsqueeze(1).expand(-1, N, -1).view(-1, 3)
    
    sigma, color = radiance_field(samples_flat, dirs_flat)
    sigma = sigma.view(B, N, 1)
    color = color.view(B, N, 3)
    
    # 5. Volume Rendering
    # Calculate delta
    # delta_i = t_{i+1} - t_i
    # Pad absolute_t with far_bound at the end
    t_padded = torch.cat([absolute_t, torch.full((B, 1), far_bound, device=absolute_t.device)], dim=1)
    delta = t_padded[:, 1:] - t_padded[:, :-1] # (B, N)
    
    # Alpha
    alpha = 1 - torch.exp(-sigma * delta) # (B, N, 1)
    
    # Transmittance
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # Cumulative sum of sigma * delta
    sigma_delta = sigma * delta # (B, N, 1)
    cum_sigma_delta = torch.cumsum(sigma_delta, dim=1) # (B, N, 1)
    # Shift for T_i: T_1 = 1, T_2 = exp(-sigma_1 delta_1), etc.
    # We need T_i for the i-th term.
    # T_i = exp(-sum_{j=0}^{i-1} sigma_j delta_j)
    # Using cumsum, sum_{j=0}^{i-1} is cumsum[i-1] if i>0, else 0.
    # Let's compute T directly.
    T = torch.exp(-torch.cat([torch.zeros((B, 1, 1), device=sigma_delta.device), cum_sigma_delta[:, :-1]], dim=1)) # (B, N, 1)
    
    # Color accumulation
    # C = sum T_i * (1 - exp(-sigma_i delta_i)) * c_i
    weights = T * alpha # (B, N, 1)
    color_acc = torch.sum(weights * color, dim=1) # (B, 3)
    
    return color_acc
```

**Edge Cases and Pitfalls:**
*   **Precision:** The sigmoid activation ensures outputs are in $[0, 1]$, preventing out-of-bounds sampling, but may compress gradients if the network outputs values near 0 or 1 [S1, unofficial].
*   **Boundary Conditions:** The near and far bounds must be correctly set for the scene; incorrect bounds lead to missing geometry or sampling empty space [S1, unofficial].
*   **Training:** The sample field and radiance field are trained end-to-end by minimizing the MSE between rendered and ground truth colors [S1, unofficial].

**Subtopics:**
*   **Sample Field Extraction:** A compression technique that reduces the number of samples (e.g., to 64) while maintaining competitive quality, achieving 25% of the inference time of the original NeuSample [S1, unofficial]. It uses depth boost initialization to fit the mean value of the extracted field to depth predicted by a learned regular field [S1, unofficial].
*   **Comparison with NeRF:** NeuSample discards the costly coarse stage of NeRF's hierarchical sampling, using single-shot inference instead [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Single-shot inference. Saves approximately 25% of inference time compared to NeRF [S1, unofficial]. Sample Field Extraction reduces this further to 25% of the original NeuSample's time [S1, unofficial].
*   **Memory Complexity:** Requires storing the parameters for the Neural Sample Field (MLP with ~8 layers, hidden dim 256) in addition to the Radiance Field. The source does not specify exact memory footprint in MB/GB.
*   **Performance:** On Realistic Synthetic 360° and Real ForwardFacing benchmarks, NeuSample achieves better or similar PSNR to NeRF while being faster [S1, unofficial].
*   **Strengths:** Faster inference than NeRF; simpler pipeline (no hierarchical sampling logic); end-to-end trainable.
*   **Limitations:** Requires training a separate network for sampling; quality depends on the learned sampling distribution; may struggle with complex geometry if not properly initialized (addressed by Sample Field Extraction) [S1, unofficial].
*   **When to use:** When inference speed is critical and the scene geometry can be effectively modeled by a learned sampling distribution.
*   **When not to use:** When hierarchical sampling is required for high-frequency details that a single MLP cannot capture efficiently, or when memory is extremely constrained (due to additional network parameters).

## Background
NeuSample was proposed in the paper "NeuSample: Neural Sample Field for Efficient View Synthesis" [S1, unofficial]. It addresses the computational overhead of NeRF's inference stage, specifically the coarse-to-fine hierarchical sampling procedure [S1, unofficial]. The method was developed to alleviate the burden of the coarse stage by replacing it with a lightweight neural sample field [S1, unofficial].

## Sources
[S1] NeuSample: Neural Sample Field for Efficient View Synthesis, https://doi.org/10.48550/arxiv.2111.15552, fetched 2026-10-27 (unofficial)
