# NeuSample

Also called: Neural Sample Field

## Overview
NeuSample is a method for efficient novel view synthesis that replaces the hierarchical coarse-to-fine sampling procedure of NeRF with a single-shot inference pipeline [S1, unofficial]. It introduces a lightweight "neural sample field" that maps a ray directly into a distribution of sample points, eliminating the computational overhead of the coarse pass while maintaining or improving rendering quality [S1, unofficial]. Unlike NeRF, which requires two-stage inference to determine sample locations based on opacity weights, NeuSample learns the optimal sampling distribution end-to-end, reducing inference time by approximately 25% on standard benchmarks [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A ray $r$ defined by its origin $o = (x_o, y_o, z_o)$ and normalized direction $d = (\theta, \phi)$.
*   **Output:** A set of $N$ 3D coordinates $\{x_1, ..., x_N\}$ along the ray, which are fed into the radiance field to produce density $\sigma$ and color $c$. The final output is the accumulated pixel color $C(r)$.

**Data Structures**
*   **Neural Sample Field ($S_\Theta$):** A Multi-Layer Perceptron (MLP) with fully connected layers.
    *   **Input Dimension:** 6 (3 for origin, 3 for direction) before position encoding.
    *   **Position Encoding:** 10-frequency sine/cosine encoding applied to both origin and direction, expanding the input dimension to $6 \times (2 \times 10 + 1) = 126$ [S1, unofficial].
    *   **Hidden Layers:** 8 fully connected layers with ReLU activations.
    *   **Hidden Dimension:** 256 features per layer [S1, unofficial].
    *   **Skip Connection:** The original encoded input is concatenated with the output of the 4th hidden layer [S1, unofficial].
    *   **Output Layer:** A final fully connected layer maps the 256-dimensional features to an $N$-dimensional vector, followed by a Sigmoid activation [S1, unofficial].
*   **Radiance Field ($R_\Theta$):** The standard NeRF MLP architecture [S1, unofficial].

**Numbered Steps**
1.  **Ray Construction:** For a given pixel, compute the camera pose and transform it into a ray origin $o$ and direction $d$.
2.  **Position Encoding:** Apply 10-frequency positional encoding to $o$ and $d$. Concatenate the encoded origin and direction vectors.
3.  **Sample Field Inference:** Pass the concatenated encoded input through the Neural Sample Field $S_\Theta$.
    *   The network processes the input through 8 FC layers with ReLU.
    *   A skip connection concatenates the initial input with the output of the 4th layer.
    *   The result passes through a final FC layer to produce an $N$-dimensional vector.
    *   Apply Sigmoid activation to obtain relative sample positions $\hat{t}_1, ..., \hat{t}_N \in [0, 1]$.
4.  **Coordinate Transformation:** Map the relative positions $\hat{t}_i$ to absolute distances $t_i$ along the ray segment $[t_{near}, t_{far}]$:
    $$ t_i = (1 - \hat{t}_i)t_{near} + \hat{t}_i t_{far} $$
    Compute the 3D sample coordinates:
    $$ x_i = o + t_i d $$
5.  **Radiance Field Evaluation:** Feed the coordinates $x_i$ and direction $d$ into the radiance field $R_\Theta$ to obtain density $\sigma_i$ and color $c_i$ for each sample point.
6.  **Volume Rendering:** Accumulate the final pixel color $C(r)$ using standard volume rendering integration:
    $$ C(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) c_i $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance, and $\delta_i$ is the distance between adjacent samples ($\delta_i = t_{i+1} - t_i$) [S1, unofficial].

**Formulas**
*   **Sample Field Output:**
    $$ \hat{t}_1, ..., \hat{t}_N = S_\Theta(x_o, y_o, z_o, \theta, \phi) $$
    where $\hat{t}_i \in [0, 1]$ represents the relative position [S1, unofficial].
*   **Absolute Distance:**
    $$ t_i = (1 - \hat{t}_i)t_{near} + \hat{t}_i t_{far} $$
*   **Sample Coordinate:**
    $$ x_i = o + t_i d $$
*   **Rendering Loss:**
    $$ L = \| \hat{C}(r) - C(r) \|_2^2 $$
    where $\hat{C}(r)$ is the rendered color and $C(r)$ is the ground truth [S1, unofficial].

**Parameters**
*   **$N$ (Number of Samples):** The total number of samples output by the neural sample field. In the source experiments, $N$ is set to 192 [S1, unofficial]. This value replaces the sum of coarse ($N_c$) and fine ($N_f$) samples used in hierarchical NeRF.
*   **Position Encoding Frequency:** 10 frequencies [S1, unofficial].
*   **Hidden Dimension:** 256 [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Precision:** The relative positions $\hat{t}_i$ are bounded by the Sigmoid function to $[0, 1]$. If $t_{near}$ and $t_{far}$ are poorly chosen, samples may fall outside the region of interest, leading to wasted computation or artifacts.
*   **Ordering:** The source does not explicitly enforce sorting of $\hat{t}_i$. However, volume rendering assumes ordered samples along the ray. In implementation, the output $\hat{t}_i$ should be sorted in ascending order before computing $t_i$ and $\delta_i$ to ensure correct transmittance calculation [G].
*   **Degenerate Input:** If the ray direction $d$ is zero or undefined, the coordinate calculation fails. Standard ray construction from camera poses should prevent this.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import math

def positional_encoding(x, num_frequencies=10):
    """
    Applies 10-frequency positional encoding to input x.
    x shape: (..., 3) for origin or direction.
    Returns shape: (..., 3 * (2 * num_frequencies + 1))
    """
    # Standard NeRF positional encoding
    # x is scaled by powers of 2 up to 2^(num_frequencies-1)
    # Then sin and cos are applied
    # Concatenate original x with sin/cos components
    freq_bands = 2.0 ** torch.linspace(0, num_frequencies - 1, num_frequencies).to(x.device)
    x = x.unsqueeze(-1) * freq_bands
    x = torch.cat([x.sin(), x.cos()], dim=-1)
    x = torch.cat([x, x], dim=-1) # sin, cos, sin, cos...
    # Actually, standard PE is sin(2^k x), cos(2^k x)
    # Let's implement strictly:
    # x shape (..., 3) -> (..., 3, num_frequencies)
    # freq_bands shape (num_frequencies,)
    # x_pe = torch.sin(torch.cat([x[..., None] * freq_bands, x[..., None] * freq_bands + math.pi/2], dim=-1))
    # Simplified standard implementation:
    x = x.unsqueeze(-1) * freq_bands
    x = torch.cat([x.sin(), x.cos()], dim=-1)
    # x is now (..., 3, 2*num_frequencies)
    # Flatten last two dims
    x = x.reshape(*x.shape[:-2], -1)
    # Concatenate original x
    x = torch.cat([x, x], dim=-1) # Wait, standard is just sin/cos of scaled x?
    # NeRF PE: sin(2^k x), cos(2^k x) for k=0..L-1
    # Result dim: 3 * 2 * L
    # Let's stick to the description: "10-frequency position encoding"
    # Usually implies the standard NeRF PE.
    return x

class NeuralSampleField(nn.Module):
    def __init__(self, N=192):
        super().__init__()
        self.N = N
        # Input dim after PE: 6 * (2*10 + 1) = 126? 
        # NeRF PE usually maps 3D to 6*L. So 6D -> 12*L = 120.
        # Let's assume standard NeRF PE: 3 coords * 2 freqs * 10 = 60 per vector.
        # Total input = 120.
        self.input_dim = 120 
        
        # 8 FC layers with ReLU
        self.layers = nn.ModuleList([
            nn.Linear(self.input_dim, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU(),
            nn.Linear(256, 256),
            nn.ReLU()
        ])
        
        # Skip connection from 4th layer (index 3 in layers list, which is output of 4th FC)
        # The description says: "concatenate the original input with the output of the 4th layer"
        # So we need to store the 4th layer output.
        
        # Final FC layer mapping 256 + 120 = 376 to N
        self.output_layer = nn.Linear(256 + self.input_dim, N)
        self.sigmoid = nn.Sigmoid()

    def forward(self, origin, direction):
        # origin: (..., 3), direction: (..., 3)
        # Apply PE
        pe_origin = positional_encoding(origin)
        pe_direction = positional_encoding(direction)
        x = torch.cat([pe_origin, pe_direction], dim=-1)
        
        # Pass through layers
        h = x
        for i, layer in enumerate(self.layers):
            h = layer(h)
            if i == 3: # After 4th layer (indices 0,1,2,3)
                skip_connection = h
        
        # Concatenate original input with 4th layer output
        h = torch.cat([x, skip_connection], dim=-1)
        
        # Final layer
        out = self.output_layer(h)
        return self.sigmoid(out)

def render_neusample(ray_origins, ray_directions, t_near, t_far, sample_field, radiance_field, N=192):
    """
    ray_origins: (B, 3)
    ray_directions: (B, 3)
    t_near: scalar
    t_far: scalar
    sample_field: NeuralSampleField instance
    radiance_field: RadianceField instance (standard NeRF MLP)
    """
    # 1. Get relative positions from sample field
    # sample_field expects encoded inputs, handles PE internally or externally?
    # Based on pseudocode above, sample_field handles PE.
    relative_positions = sample_field(ray_origins, ray_directions) # (B, N)
    
    # 2. Map to absolute distances
    # t_i = (1 - t_hat_i) * t_near + t_hat_i * t_far
    t_abs = (1 - relative_positions) * t_near + relative_positions * t_far # (B, N)
    
    # 3. Sort samples to ensure correct ordering for volume rendering
    t_abs, _ = torch.sort(t_abs, dim=-1)
    
    # 4. Compute 3D coordinates
    # x_i = o + t_i * d
    # t_abs is (B, N), ray_directions is (B, 3)
    # Expand directions to (B, 1, 3)
    samples = ray_origins.unsqueeze(1) + t_abs.unsqueeze(2) * ray_directions.unsqueeze(1) # (B, N, 3)
    
    # 5. Evaluate Radiance Field
    # Standard NeRF forward pass
    # radiance_field takes (x, d)
    # samples: (B, N, 3), ray_directions: (B, 3) -> expand to (B, N, 3)
    d_expanded = ray_directions.unsqueeze(1).expand_as(samples)
    sigma, color = radiance_field(samples, d_expanded)
    
    # 6. Volume Rendering
    # Compute delta
    # delta_i = t_{i+1} - t_i
    # Add a small epsilon to avoid division by zero or issues at boundaries
    # Last delta is t_far - t_N
    deltas = t_abs[:, 1:] - t_abs[:, :-1] # (B, N-1)
    # Append last delta
    last_delta = t_far - t_abs[:, -1]
    deltas = torch.cat([deltas, last_delta.unsqueeze(1)], dim=1) # (B, N)
    
    # Transmittance
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # alpha_i = 1 - exp(-sigma_i * delta_i)
    alpha = 1.0 - torch.exp(-sigma * deltas) # (B, N)
    
    # Cumulative product of transmittance
    # T_1 = 1
    # T_i = T_{i-1} * (1 - alpha_{i-1})
    # Using cumulative sum of log(1-alpha)
    log_alpha = torch.log(torch.clamp(alpha, min=1e-10))
    log_T = torch.cat([torch.zeros_like(log_alpha[:, :1]), torch.cumsum(log_alpha[:, :-1], dim=1)], dim=1)
    T = torch.exp(log_T) # (B, N)
    
    # Weighted sum
    weights = T * alpha # (B, N)
    color_out = torch.sum(weights.unsqueeze(-1) * color, dim=1) # (B, 3)
    
    return color_out
```

## Operating characteristics
*   **Time Complexity:** The method performs a single forward pass through the sample field and a single forward pass through the radiance field for $N$ samples. This is significantly faster than NeRF's hierarchical approach, which requires two passes (coarse and fine) with potentially more total samples.
*   **Memory Complexity:** Memory usage is dominated by the storage of the sample field and radiance field parameters. The sample field is lightweight (8 FC layers, 256 hidden units), adding minimal overhead compared to the radiance field.
*   **Measured Performance:** On Realistic Synthetic 360° and Real ForwardFacing benchmarks, NeuSample saves approximately 25% of inference time compared to standard NeRF while achieving comparable or better PSNR (e.g., +0.59 PSNR on Ficus, +0.63 on Lego) [S1, unofficial].
*   **Strengths:** Eliminates the need for a separate coarse network and hierarchical sampling logic. Simplifies the inference pipeline to a single shot.
*   **Limitations:** The sample field must be trained end-to-end with the radiance field. It may struggle with scenes having highly complex or unknown geometry distributions if not properly regularized or initialized (e.g., using depth boost for real-world scenes) [S1, unofficial].
*   **When to use:** When inference speed is critical and the scene geometry is relatively well-behaved or can be learned by the sample field.
*   **When not to use:** If the scene has extreme depth variations or complex occlusions that the sample field fails to capture without significant training data or regularization.

## Background
NeuSample was introduced in the paper "NeuSample: Neural Sample Field for Efficient View Synthesis" [S1, unofficial]. It addresses the computational bottleneck of NeRF's hierarchical sampling, which requires two-stage inference. The authors propose replacing the coarse stage with a neural sample field that directly predicts sample locations. The method was evaluated on standard synthetic benchmarks (Realistic Synthetic 360°, Real ForwardFacing) and demonstrated improved efficiency and quality. The work also explores a "sample field extraction" method to further compress the model and reduce inference time by up to 75% with competitive quality [S1, unofficial].

## Sources
[S1] NeuSample: Neural Sample Field for Efficient View Synthesis, https://doi.org/10.48550/arxiv.2111.15552, fetched 2026-10-27 (unofficial)
