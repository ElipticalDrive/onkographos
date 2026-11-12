# NeRF-ID
Also called: NeRF in detail

## Overview
NeRF-ID replaces the heuristic, non-differentiable coarse-to-fine sampling procedure of vanilla NeRF with a learnable, differentiable "proposer" module, enabling end-to-end training of the entire system (coarse network, proposer, and fine network) to improve view synthesis quality [S1, unofficial]. Unlike NeRF, which uses a handcrafted piece-wise constant probability density function based on coarse densities to determine fine sample locations, NeRF-ID uses a neural network to directly predict fine sample positions from coarse features, allowing the sampling strategy to be optimized for the specific task [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A ray defined by near and far planes; a coarse network $N_c$; a proposer module $P$; a fine network $N_f$.
*   **Outputs:** A rendered color $\hat{\mathbf{C}}$ for the ray.

**Data Structures:**
*   **Coarse Samples:** A set of $N_c$ positions $t_i$ equally spaced in $[t_n, t_f]$.
*   **Coarse Features:** For each coarse sample $i$, the feature vector $\mathbf{f}_i$ is the output of the coarse network's last activation layer before the projection that produces density $\sigma_i$. The size of $\mathbf{f}_i$ is determined by the coarse network's hidden dimension.
*   **Proposer Input:** A sequence of pairs $(\mathbf{f}_i, \tau_i)$ for $i=1 \dots N_c$, where $\tau_i \in [0, 1]$ is the normalized position of the $i$-th sample along the ray.
*   **Proposer Output:** A set of $N_f$ normalized positions $\tau'_j \in [0, 1]$ for $j=1 \dots N_f$.

**Numbered Steps:**
1.  **Coarse Sampling:** Query the coarse network $N_c$ on $N_c$ equally spaced samples along the ray to obtain densities $\sigma_i$ and features $\mathbf{f}_i$.
2.  **Proposal Generation:** Pass the coarse features $\mathbf{f}_i$ and their normalized positions $\tau_i$ through the proposer module $P$ to generate $N_f$ new normalized positions $\tau'_j$.
3.  **Fine Sampling:** Map the normalized positions $\tau'_j$ back to the ray interval $[t_n, t_f]$ to get absolute positions $t'_j$. Query the fine network $N_f$ on the union of the original $N_c$ samples and the proposed $N_f$ samples (total $N_c + N_f$ points).
4.  **Rendering:** Compute the final ray color using the standard volume rendering equation with the densities and colors from all $N_c + N_f$ samples.

**Formulas:**
The rendering equation for ray color $\hat{\mathbf{C}}$ is:
$$ \hat{\mathbf{C}} = \sum_{i=1}^N w_i \mathbf{c}_i $$
where $N = N_c + N_f$, $\mathbf{c}_i$ is the color at the $i$-th sample, and the weight $w_i$ and transmittance $T_i$ are:
$$ w_i = T_i \left( 1 - \exp(-\sigma_i(t_{i+1} - t_i)) \right) $$
$$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j(t_{j+1} - t_j)\right) $$
Here, $\sigma_i$ is the density, $\mathbf{c}_i$ is the color, and $t_i$ is the absolute location along the ray for the $i$-th sample [S1, unofficial].

The proposer outputs normalized positions $\tau'_j \in [0, 1]$. These are converted to absolute positions $t'_j$ via:
$$ t'_j = t_n + \tau'_j (t_f - t_n) $$

**Parameters:**
*   $N_c$: Number of equally spaced samples queried from the coarse network. Typical values are not explicitly given in the notes, but standard NeRF uses small integers (e.g., 64).
*   $N_f$: Number of additional samples proposed by the proposer for the fine network. Typical values are not explicitly given in the notes, but standard NeRF uses larger integers (e.g., 128).

**Pseudocode:**
```python
import torch
import torch.nn as nn

def render_ray(ray_origin, ray_direction, near, far, coarse_net, proposer, fine_net, N_c, N_f):
    """
    Renders a single ray using NeRF-ID.
    
    Args:
        ray_origin: (3,) Tensor, origin of the ray.
        ray_direction: (3,) Tensor, direction of the ray.
        near: float, near plane distance.
        far: float, far plane distance.
        coarse_net: nn.Module, the coarse MLP.
        proposer: nn.Module, the proposer network (Transformer, Pool, or MLPMix).
        fine_net: nn.Module, the fine MLP.
        N_c: int, number of coarse samples.
        N_f: int, number of fine samples.
        
    Returns:
        color: (3,) Tensor, rendered color.
    """
    # Step 1: Coarse Sampling
    # Generate N_c equally spaced samples in [near, far]
    t_coarse = torch.linspace(near, far, N_c)
    # Expand to (N_c, 3) for batched query
    pts_coarse = ray_origin.unsqueeze(0) + t_coarse.unsqueeze(1).unsqueeze(2) * ray_direction.unsqueeze(0).unsqueeze(1)
    
    # Query coarse network
    # Assume coarse_net returns (sigma, color, features)
    # Note: The proposer needs features before the density projection.
    # We assume coarse_net has a method or attribute to get intermediate features.
    # If not, we might need to modify the network or assume the last hidden layer.
    # For this pseudocode, we assume coarse_net returns (sigma, color, features)
    # where features are the activations before the final linear layer for sigma.
    sigma_coarse, color_coarse, features_coarse = coarse_net(pts_coarse)
    
    # Normalize positions to [0, 1] for proposer input
    tau_coarse = (t_coarse - near) / (far - near)
    
    # Step 2: Proposal Generation
    # Proposer input: features (N_c, D) and positions (N_c,)
    # Proposer output: N_f normalized positions
    tau_fine = proposer(features_coarse, tau_coarse)
    
    # Map normalized positions back to [near, far]
    t_fine = tau_fine * (far - near) + near
    
    # Step 3: Fine Sampling
    # Combine coarse and fine samples
    t_all = torch.cat([t_coarse, t_fine])
    pts_all = ray_origin.unsqueeze(0) + t_all.unsqueeze(1).unsqueeze(2) * ray_direction.unsqueeze(0).unsqueeze(1)
    
    # Query fine network
    sigma_fine, color_fine, _ = fine_net(pts_all)
    
    # Step 4: Rendering
    # Compute weights and transmittance
    # Sort samples by t_all to ensure correct transmittance calculation
    sorted_indices = torch.argsort(t_all)
    t_sorted = t_all[sorted_indices]
    sigma_sorted = sigma_fine[sorted_indices].squeeze(1)
    color_sorted = color_fine[sorted_indices].squeeze(1)
    
    # Compute intervals
    dt = t_sorted[1:] - t_sorted[:-1]
    # Pad dt to match N samples (last interval doesn't matter for weight but needed for T)
    dt = torch.cat([dt, dt[-1:]])
    
    # Compute transmittance T_i = exp(-sum_{j=1}^{i-1} sigma_j * dt_j)
    # alpha_i = 1 - exp(-sigma_i * dt_i)
    alpha = 1.0 - torch.exp(-sigma_sorted * dt)
    
    # T_1 = 1
    # T_i = T_{i-1} * (1 - alpha_{i-1})
    T = torch.ones_like(alpha)
    T[1:] = torch.cumprod(1.0 - alpha[:-1], dim=0)
    
    # Weights w_i = T_i * alpha_i
    weights = T * alpha
    
    # Rendered color
    color = torch.sum(weights.unsqueeze(1) * color_sorted, dim=0)
    
    return color

class TransformerProposer(nn.Module):
    """
    Transformer-based proposer architecture.
    """
    def __init__(self, feature_dim, N_f, hidden_dim=256, num_heads=4, num_layers=2):
        super().__init__()
        self.N_f = N_f
        # Learnable queries for the decoder
        self.queries = nn.Parameter(torch.randn(1, N_f, hidden_dim))
        
        # Encoder: processes input features
        encoder_layer = nn.TransformerEncoderLayer(d_model=hidden_dim, nhead=num_heads, batch_first=True)
        self.encoder = nn.TransformerEncoder(encoder_layer, num_layers=num_layers)
        
        # Decoder: cross-attention with queries and encoder output
        decoder_layer = nn.TransformerDecoderLayer(d_model=hidden_dim, nhead=num_heads, batch_first=True)
        self.decoder = nn.TransformerDecoder(decoder_layer, num_layers=num_layers)
        
        # Linear layer to map decoder output to normalized positions [0, 1]
        self.output_layer = nn.Linear(hidden_dim, 1)
        
        # Positional encoding for input features (optional, but mentioned in Pool/MLPMix)
        # Transformer encoder usually handles positional info internally or via input embedding
        self.input_embedding = nn.Linear(feature_dim, hidden_dim)
        
    def forward(self, features, positions):
        """
        Args:
            features: (N_c, feature_dim)
            positions: (N_c,) normalized positions in [0, 1]
        Returns:
            tau_fine: (N_f,) normalized positions in [0, 1]
        """
        # Add positional encoding to features? 
        # The paper says "A transformer encoder processes the input features".
        # It doesn't explicitly say how positional info is added to the encoder input.
        # However, the Pool architecture explicitly sums features with positional encoding.
        # For Transformer, we might add a learned positional embedding or use the positions as a separate input.
        # Given the description, we assume the encoder takes the features directly.
        # To incorporate position, we might add a positional encoding to the features.
        # Let's assume we add a sinusoidal positional encoding to the features.
        
        # Create positional encoding (N_c, hidden_dim)
        # Standard sinusoidal PE
        pe = self._get_positional_encoding(positions, hidden_dim)
        
        # Add PE to features
        x = self.input_embedding(features) + pe
        
        # Encode
        x = self.encoder(x)  # (N_c, hidden_dim)
        
        # Decode
        # Queries: (1, N_f, hidden_dim)
        # Key/Value: x (N_c, hidden_dim)
        output = self.decoder(self.queries, x)  # (1, N_f, hidden_dim)
        
        # Output layer
        tau_fine = self.output_layer(output).squeeze(0).squeeze(1)  # (N_f,)
        
        # Sigmoid to ensure [0, 1]
        tau_fine = torch.sigmoid(tau_fine)
        
        return tau_fine
    
    def _get_positional_encoding(self, positions, dim):
        # Simple sinusoidal PE
        half_dim = dim // 2
        emb = math.log(10000) / (half_dim - 1)
        emb = torch.exp(torch.arange(half_dim) * -emb)
        emb = positions.unsqueeze(1) * emb.unsqueeze(0)
        emb = torch.cat((torch.sin(emb), torch.cos(emb)), dim=1)
        return emb

class PoolProposer(nn.Module):
    """
    Pool-based proposer architecture.
    """
    def __init__(self, feature_dim, N_f, hidden_dim=256):
        super().__init__()
        self.fc1 = nn.Linear(feature_dim + 1, hidden_dim) # +1 for positional encoding
        self.relu = nn.ReLU()
        self.fc2 = nn.Linear(hidden_dim, N_f)
        
    def forward(self, features, positions):
        """
        Args:
            features: (N_c, feature_dim)
            positions: (N_c,) normalized positions in [0, 1]
        Returns:
            tau_fine: (N_f,) normalized positions in [0, 1]
        """
        # Sum features with positional encoding
        # Positional encoding: sin/cos of position? Or just the position itself?
        # The paper says "positional encoding of its location along the ray".
        # It doesn't specify the type of PE. Let's assume a simple sin/cos PE like in Transformers.
        pe = self._get_positional_encoding(positions, features.shape[1])
        
        # Concatenate features and PE
        x = torch.cat([features, pe], dim=1)
        
        # FC + ReLU
        x = self.relu(self.fc1(x))
        
        # Average pooling
        ray_repr = torch.mean(x, dim=0)
        
        # FC to produce proposals
        tau_fine = self.fc2(ray_repr)
        
        # Sigmoid
        tau_fine = torch.sigmoid(tau_fine)
        
        return tau_fine
    
    def _get_positional_encoding(self, positions, dim):
        # Sinusoidal PE
        half_dim = dim // 2
        emb = math.log(10000) / (half_dim - 1)
        emb = torch.exp(torch.arange(half_dim) * -emb)
        emb = positions.unsqueeze(1) * emb.unsqueeze(0)
        emb = torch.cat((torch.sin(emb), torch.cos(emb)), dim=1)
        return emb

class MLPMixProposer(nn.Module):
    """
    MLPMix-based proposer architecture.
    """
    def __init__(self, feature_dim, N_f, hidden_dim=256):
        super().__init__()
        # MLPMix block
        # Assuming MLPMix is a custom layer that mixes features
        # For pseudocode, we'll approximate it with a simple MLP if MLPMix is not defined
        # The paper says "Apply a single MLPMix block to the stacked point features sorted by their position"
        # Then average pooling, then FC.
        self.mix_block = MLPMixBlock(feature_dim, hidden_dim) # Placeholder for MLPMix
        self.fc = nn.Linear(hidden_dim, N_f)
        
    def forward(self, features, positions):
        """
        Args:
            features: (N_c, feature_dim)
            positions: (N_c,) normalized positions in [0, 1]
        Returns:
            tau_fine: (N_f,) normalized positions in [0, 1]
        """
        # Sort features by position
        sorted_indices = torch.argsort(positions)
        sorted_features = features[sorted_indices]
        
        # Apply MLPMix block
        ray_repr = self.mix_block(sorted_features)
        
        # Average pooling (MLPMix might output a single vector if it's a global mixer)
        # If MLPMix outputs (N_c, hidden_dim), we need to pool.
        # The paper says "followed by average pooling to get the ray representation"
        # So MLPMix likely outputs (N_c, hidden_dim)
        ray_repr = torch.mean(ray_repr, dim=0)
        
        # FC to produce proposals
        tau_fine = self.fc(ray_repr)
        
        # Sigmoid
        tau_fine = torch.sigmoid(tau_fine)
        
        return tau_fine

class MLPMixBlock(nn.Module):
    """
    Placeholder for MLPMix block.
    MLPMix is a specific architecture for mixing features.
    For the purpose of this pseudocode, we assume it's a simple MLP that processes each point.
    """
    def __init__(self, input_dim, hidden_dim):
        super().__init__()
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.fc2 = nn.Linear(hidden_dim, input_dim)
        self.relu = nn.ReLU()
        
    def forward(self, x):
        # x: (N_c, input_dim)
        x = self.relu(self.fc1(x))
        x = self.fc2(x)
        return x

**Two-Stage Training Strategy:**
1.  **Pre-training:** Train the coarse and fine networks to mimic vanilla NeRF (using heuristic sampling) to stabilize training [S1, unofficial].
2.  **Fine-tuning:** Enable the proposer module and train the entire system (coarse, proposer, fine) end-to-end to learn better sampling strategies [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Training Instability:** Training the proposer from scratch can be unstable due to weak supervision (only final color loss). The two-stage training strategy mitigates this [S1, unofficial].
*   **Precision:** Normalized positions must be carefully mapped to absolute positions to avoid precision issues, especially for rays with large near-far ranges.
*   **Degenerate Input:** If the coarse network produces very low densities everywhere, the proposer might struggle to learn meaningful proposals. The pre-training stage helps initialize the proposer with useful features.

## Subtopics

### Transformer Proposer
Uses a transformer encoder-decoder architecture with learned queries. It is suitable when the relationship between coarse features and fine sample locations is complex and non-local. [Link to Transformer Proposer](transformer-proposer.md)

### Pool Proposer
Uses a simple MLP with positional encoding and average pooling. It is computationally efficient and suitable for simpler scenes or when computational resources are limited. [Link to Pool Proposer](pool-proposer.md)

### MLPMix Proposer
Uses an MLPMix block for feature mixing. It offers a balance between the complexity of the Transformer and the simplicity of the Pool proposer. [Link to MLPMix Proposer](mlpmix-proposer.md)

## Operating characteristics
*   **Time Complexity:** The proposer adds a small computational overhead compared to vanilla NeRF, as it is designed to be small relative to the coarse and fine networks. The total number of MLP evaluations is $N_c + N_f$, same as vanilla NeRF, but the sampling is more efficient, potentially allowing for a reduction in $N_f$ for the same quality.
*   **Memory Complexity:** The proposer adds a small number of parameters. The memory usage is dominated by the coarse and fine networks.
*   **Measured Performance:** NeRF-ID achieves superior view synthesis quality over NeRF and state-of-the-art methods on synthetic Blender benchmarks and performs on par or better on real LLFF-NeRF scenes [S1, unofficial]. It can achieve a 25% saving in computation without significantly sacrificing rendering quality by filtering out least promising samples [S1, unofficial].
*   **Strengths:** End-to-end differentiability, improved quality, potential for computational savings.
*   **Limitations:** Requires two-stage training, which adds complexity to the training pipeline. The proposer architecture choice can impact performance.
*   **When to use:** When high-quality view synthesis is required and computational resources allow for the additional training complexity.
*   **When not to use:** When computational resources are extremely limited and the overhead of the proposer is prohibitive, or when the two-stage training strategy is not feasible.

## Background
NeRF-ID was introduced in the paper "NeRF in detail: Learning to sample for view synthesis" [S1, unofficial]. It addresses the limitation of the vanilla NeRF coarse-to-fine approach, which relies on a heuristic and is not trained end-to-end. The method was developed to improve view synthesis quality by learning a differentiable sampling strategy.

## Sources
[S1] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
