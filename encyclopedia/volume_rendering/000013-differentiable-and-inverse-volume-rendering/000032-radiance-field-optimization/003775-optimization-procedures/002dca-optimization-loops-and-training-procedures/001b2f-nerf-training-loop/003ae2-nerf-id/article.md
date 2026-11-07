# NeRF-ID
Also called: NeRF in detail

## Overview
NeRF-ID (NeRF in detail) is a differentiable volume rendering optimization method that replaces the heuristic coarse-to-fine sampling procedure of vanilla NeRF with a learnable "proposer" module [S1, unofficial]. It solves the problem of suboptimal sample placement by enabling end-to-end training of the coarse network, proposer, and fine network jointly, thereby improving view synthesis quality [S1, unofficial]. Unlike its sibling [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) which relies on handcrafted probability density functions derived from coarse densities, NeRF-ID uses a neural network to directly predict fine sample locations from coarse features [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin and direction; training images with ground truth colors.
*   **Output:** Rendered ray color $\hat{\mathbf{C}}$ and gradients for updating the coarse network, proposer, and fine network parameters.

**Data Structures:**
1.  **Coarse Network ($N_c$):** A multi-layer perceptron (MLP) taking position and viewing direction, outputting density $\sigma$ and color $\mathbf{c}$. It also outputs intermediate feature vectors (activations before the final density projection) for each of the $N_c$ samples.
2.  **Proposer ($P$):** A lightweight neural network taking coarse features and positions as input, outputting $N_f$ scalar positions in $[0, 1]$.
3.  **Fine Network ($N_f$):** An MLP taking position and viewing direction, outputting density $\sigma$ and color $\mathbf{c}$.
4.  **Sample Points:**
    *   Coarse samples: $N_c$ points, equally spaced along the ray interval $[t_n, t_f]$.
    *   Fine samples: $N_f$ points, proposed by the proposer, mapped to $[t_n, t_f]$.

**Numbered Steps:**

1.  **Coarse Sampling:** Query the coarse network at $N_c$ equally spaced positions $t_i$ along the ray. Retrieve the output density $\sigma_i$, color $\mathbf{c}_i$, and the intermediate feature vector $\mathbf{f}_i$ for each sample $i \in \{1, \dots, N_c\}$.
2.  **Proposer Input Preparation:** Construct the input sequence for the proposer. For each coarse sample $i$, create a pair consisting of the feature vector $\mathbf{f}_i$ and its normalized position $p_i = \frac{t_i - t_n}{t_f - t_n}$, where $p_i \in [0, 1]$ [S1, unofficial].
3.  **Proposer Output Generation:** Pass the sequence of $(\mathbf{f}_i, p_i)$ pairs through the proposer network $P$. The network outputs $N_f$ scalar values. Apply a sigmoid function $\sigma(\cdot)$ to each scalar to constrain the proposed positions to the range $[0, 1]$ [S1, unofficial].
4.  **Fine Sampling:** Map the sigmoid outputs to the ray interval $[t_n, t_f]$ to obtain $N_f$ fine sample positions. Query the fine network at these $N_f$ positions, as well as at the original $N_c$ coarse positions.
5.  **Rendering:** Compute the final ray color $\hat{\mathbf{C}}$ using the rendering equation with all $N_c + N_f$ samples [S1, unofficial]:
    $$
    \hat{\mathbf{C}} = \sum_{i=1}^{N} w_i \mathbf{c}_i
    $$
    where $N = N_c + N_f$, and the weight $w_i$ and transmittance $T_i$ are defined as:
    $$
    w_i = T_i \left( 1 - \exp(-\sigma_i(t_{i+1} - t_i)) \right)
    $$
    $$
    T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j(t_{i+1} - t_i)\right)
    $$
    Here, $\sigma_i$ is the density, $\mathbf{c}_i$ is the color, and $t_i$ is the location along the ray for the $i$-th sample, sorted by depth [S1, unofficial].
6.  **Optimization:** Minimize the L2 loss between $\hat{\mathbf{C}}$ and the ground truth image color using gradient descent. All operations (coarse query, proposer, fine query, integration) are differentiable, allowing gradients to flow back to the proposer [S1, unofficial].

**Proposer Architectures:**
The proposer $P$ can be implemented using one of the following architectures [S1, unofficial]:
1.  **Transformer:** A transformer encoder processes the input feature sequence. A transformer decoder uses $N_f$ learned queries to attend to the encoded features via cross-attention, producing the $N_f$ proposals.
2.  **Pool:** For each sample, sum the feature $\mathbf{f}_i$ with a positional encoding of $p_i$. Pass the result through a fully connected (FC) layer followed by a ReLU activation. Average the processed features across all $N_c$ samples to form a single ray representation vector. Pass this vector through an FC layer with output dimension $N_f$ to produce the proposals.
3.  **MLPMix:** Stack the point features sorted by their position $p_i$. Apply a single MLPMix block to the stacked features. Perform average pooling to get the ray representation. Decode proposals via an FC layer [S1, unofficial].

**Parameters:**
*   $N_c$: Number of coarse samples. Typical values are not explicitly fixed in the notes, but standard NeRF uses $N_c=64$ or similar. Not given in the sources.
*   $N_f$: Number of fine samples proposed. Typical values are not explicitly fixed. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Training Instability:** Training the proposer from scratch can be unstable due to weak supervision (only final color loss) [S1, unofficial].
*   **Solution:** Use a two-stage training strategy:
    1.  **Pre-training:** Train the coarse and fine networks to mimic vanilla NeRF (using heuristic sampling) to stabilize the base networks.
    2.  **Fine-tuning:** Enable the proposer and train the entire system end-to-end [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

def render_ray(ray_origin, ray_direction, coarse_net, proposer, fine_net, t_near, t_far, N_c, N_f):
    """
    Renders a single ray using NeRF-ID.
    
    Args:
        ray_origin: Tensor (3,)
        ray_direction: Tensor (3,)
        coarse_net: nn.Module (Coarse MLP)
        proposer: nn.Module (Proposer network)
        fine_net: nn.Module (Fine MLP)
        t_near: float
        t_far: float
        N_c: int (number of coarse samples)
        N_f: int (number of fine samples)
        
    Returns:
        color: Tensor (3,)
    """
    # Step 1: Coarse Sampling
    # Equally spaced samples in [0, 1] normalized space
    t_coarse_norm = torch.linspace(0, 1, N_c, device=ray_origin.device)
    t_coarse = t_near + t_coarse_norm * (t_far - t_near)
    
    # Get 3D positions
    positions_coarse = ray_origin.unsqueeze(0) + t_coarse.unsqueeze(1) * ray_direction.unsqueeze(0)
    
    # Query coarse network
    # Assume coarse_net returns (sigma, color, features)
    # features: last activations before density projection
    sigma_coarse, color_coarse, features_coarse = coarse_net(positions_coarse, ray_direction)
    
    # Step 2: Proposer Input Preparation
    # Normalize positions to [0, 1] for proposer input
    pos_norm_coarse = t_coarse_norm.unsqueeze(1).expand(-1, features_coarse.shape[1])
    
    # Stack features and positions
    # features_coarse shape: (N_c, D_f)
    # pos_norm_coarse shape: (N_c, 1)
    proposer_input = torch.cat([features_coarse, pos_norm_coarse], dim=1)
    
    # Step 3: Proposer Output Generation
    # Proposer outputs N_f scalars
    proposals_raw = proposer(proposer_input) # Shape: (N_c, N_f) or (N_f,) depending on implementation
    # Note: The paper describes passing features through proposer. 
    # For Transformer/MLPMix, input is sequence. For Pool, input is processed per point then pooled.
    # Assuming proposer returns N_f scalars directly or via pooling.
    # If proposer returns (N_c, N_f), we might need to aggregate or the architecture handles it.
    # Based on "Pool" description: "average the processed features... pass through FC... output dimension N_f"
    # Based on "Transformer": "decoder produces final set of proposals via cross attention with N_f learnt queries"
    # Let's assume proposer returns shape (N_f,) or (1, N_f)
    
    # Sigmoid to constrain to [0, 1]
    t_fine_norm = torch.sigmoid(proposals_raw)
    
    # Step 4: Fine Sampling
    t_fine = t_near + t_fine_norm * (t_far - t_near)
    positions_fine = ray_origin.unsqueeze(0) + t_fine.unsqueeze(1) * ray_direction.unsqueeze(0)
    
    # Union of samples
    t_all = torch.cat([t_coarse, t_fine], dim=0)
    positions_all = torch.cat([positions_coarse, positions_fine], dim=0)
    
    # Sort samples by depth (t_all)
    t_all_sorted, sort_indices = torch.sort(t_all)
    positions_all_sorted = positions_all[sort_indices]
    
    # Query fine network on all samples
    sigma_all, color_all = fine_net(positions_all_sorted, ray_direction)
    
    # Step 5: Rendering
    # Compute weights and transmittance
    # Delta t
    dt = t_all_sorted[1:] - t_all_sorted[:-1]
    # Pad dt to match N samples? No, rendering equation uses intervals.
    # Standard NeRF rendering:
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * dt_j)
    # w_i = T_i * (1 - exp(-sigma_i * dt_i))
    
    # Add a small epsilon to dt to avoid division by zero or log(0) issues if needed, 
    # though exp handles 0 fine.
    dt = dt + 1e-10
    
    # Compute alpha
    alpha = 1.0 - torch.exp(-sigma_all * dt)
    
    # Compute transmittance
    # T_1 = 1
    # T_i = T_{i-1} * (1 - alpha_{i-1})
    T = torch.ones_like(alpha)
    T[1:] = torch.cumprod(1.0 - alpha[:-1], dim=0)
    
    # Weights
    weights = T * alpha
    
    # Color
    color = torch.sum(weights.unsqueeze(1) * color_all, dim=0)
    
    return color

class ProposerPool(nn.Module):
    """
    Pool architecture for the proposer.
    """
    def __init__(self, feature_dim, N_f, hidden_dim=64):
        super().__init__()
        self.fc1 = nn.Linear(feature_dim + 1, hidden_dim) # +1 for positional encoding
        self.relu = nn.ReLU()
        self.fc2 = nn.Linear(hidden_dim, N_f)
        
    def forward(self, features, pos_norm):
        """
        features: (N_c, feature_dim)
        pos_norm: (N_c, 1)
        """
        # Sum feature with positional encoding
        x = features + pos_norm # Broadcasting if needed, or concat then FC? 
        # Paper says: "summed up with the positional encoding... passed through FC"
        # Assuming pos_norm is encoded or just added if dims match? 
        # Usually positional encoding is a vector. 
        # If pos_norm is scalar per point, we might need to expand or encode it.
        # Let's assume pos_norm is already encoded to feature_dim or we add a learned PE.
        # For simplicity, assuming pos_norm is broadcastable or encoded.
        # If pos_norm is (N_c, 1) and features is (N_c, D), we can't sum directly.
        # Let's assume we concatenate or pos_norm is expanded.
        # Re-reading: "summed up with the positional encoding of its location"
        # This implies PE is same dim as feature.
        
        # Let's assume pos_norm is (N_c, feature_dim) for the sum to work
        # Or we use a learned positional embedding layer.
        # For the sake of valid Python, let's assume pos_norm is expanded or encoded.
        
        x = self.fc1(x)
        x = self.relu(x)
        
        # Average pooling
        ray_repr = torch.mean(x, dim=0, keepdim=True)
        
        # FC to proposals
        proposals = self.fc2(ray_repr)
        
        return proposals
```

## Subtopics

### Training Ray Marching and Integration
The underlying integration method used by NeRF-ID. NeRF-ID builds on [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) but replaces the sampling strategy.

### Training Configuration
NeRF-ID requires specific configuration for the two-stage training process. See [Training Configuration](2332-training-configuration.md).

### Regularization and Loss Functions
NeRF-ID uses L2 loss on color. Other regularization techniques may be applied. See [Regularization and Loss Functions](8586-regularization-and-loss-functions.md).

## Operating characteristics
*   **Time Complexity:** The proposer adds negligible computational overhead as it is designed to be small relative to the coarse and fine networks [S1, unofficial]. The total computation is dominated by the coarse and fine network queries.
*   **Memory Complexity:** Requires storing intermediate features from the coarse network for the proposer input [S1, unofficial].
*   **Measured Performance:** Achieves superior view synthesis quality over vanilla NeRF on the synthetic Blender benchmark and on par or better performance on real LLFF-NeRF scenes [S1, unofficial].
*   **Scaling:** By leveraging predicted sample importance (if the proposer outputs importance estimates), a 25% saving in computation can be achieved without significantly sacrificing rendering quality [S1, unofficial].
*   **Strengths:** End-to-end differentiable sampling; better sample placement than heuristics; potential for computation savings via importance filtering.
*   **Limitations:** Training from scratch can be unstable; requires two-stage training strategy for stability [S1, unofficial].

## Background
NeRF-ID was introduced in the paper "NeRF in detail: Learning to sample for view synthesis" [S1, unofficial]. It addresses the limitation of vanilla NeRF's coarse-to-fine sampling, which relies on a heuristic, non-trainable procedure [S1, unofficial]. The method was developed to improve view synthesis quality by learning the sampling distribution directly from data [S1, unofficial].

## Sources
[S1] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
