# NeRF-ID

Also called: NeRF in detail

## Overview
NeRF-ID (NeRF in detail) is a differentiable ray sampling method that replaces the heuristic coarse-to-fine sampling of vanilla NeRF with a learnable "proposer" module [S1, unofficial]. It solves the problem of suboptimal sample placement by enabling end-to-end training of the coarse network, proposer, and fine network jointly, allowing the system to learn where to concentrate samples for maximum rendering quality [S1, unofficial]. Unlike sibling methods that rely on handcrafted probability density functions derived from coarse densities, NeRF-ID uses a neural network to directly predict fine sample locations from coarse features [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by near and far planes. The coarse network outputs $N_c$ feature vectors and their corresponding normalized positions $t \in [0, 1]$ for equally spaced samples.
*   **Output:** A set of $N_f$ new normalized positions $t \in [0, 1]$ for the fine network, and the final rendered ray color $\hat{\mathbf{C}}$.

**Data Structures:**
*   **Coarse Features:** A tensor of shape $(N_c, D)$, where $D$ is the feature dimension (last activations before the density projection).
*   **Coarse Positions:** A tensor of shape $(N_c, 1)$, values normalized to $[0, 1]$.
*   **Proposer Output:** A tensor of shape $(N_f, 1)$, values normalized to $[0, 1]$.

**Rendering Equation:**
The final color is computed using the standard volume rendering equation [S1, unofficial]:
$$
\hat{\mathbf{C}} = \sum_{i=1}^N w_i \mathbf{c}_i
$$
where $N = N_c + N_f$, $\mathbf{c}_i$ is the color, and $w_i$ is the weight:
$$
w_i = T_i \left( 1 - \exp(-\sigma_i(t_{i+1} - t_i)) \right)
$$
$$
T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j(t_{i+1} - t_i)\right)
$$
Here, $\sigma_i$ is the density, $t_i$ is the location along the ray, and $T_i$ is the transmittance [S1, unofficial].

**Proposer Architectures:**
The proposer takes the coarse features and positions as input and outputs $N_f$ positions. Three architectures are described [S1, unofficial]:
1.  **Transformer:** A transformer encoder processes the input features. A transformer decoder produces the proposals via cross-attention with $N_f$ learned queries.
2.  **Pool:** Each point feature is summed with the positional encoding of its location. This sum is passed through a fully connected (FC) layer followed by ReLU. The processed features are averaged to form a single ray representation vector, which is passed through an FC layer with output dimension $N_f$.
3.  **MLPMix:** A single MLPMix block is applied to the stacked point features (sorted by position), followed by average pooling to get the ray representation, then decoded via an FC [S1, unofficial].

**Training Strategy:**
Training from scratch is unstable due to weak supervision (only final color loss). A two-stage strategy is used [S1, unofficial]:
1.  **Pre-training:** Train the coarse and fine networks to mimic vanilla NeRF (heuristic sampling).
2.  **Fine-tuning:** Jointly train the coarse network, proposer, and fine network end-to-end [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class NeRFID:
    def __init__(self, coarse_net, fine_net, proposer, N_c=64, N_f=48):
        """
        Initialize NeRF-ID components.
        :param coarse_net: Network returning (density, color, features) for coarse samples.
        :param fine_net: Network returning (density, color) for fine samples.
        :param proposer: Neural network taking coarse features/positions, outputting N_f positions.
        :param N_c: Number of coarse samples.
        :param N_f: Number of fine samples.
        """
        self.coarse_net = coarse_net
        self.fine_net = fine_net
        self.proposer = proposer
        self.N_c = N_c
        self.N_f = N_f

    def sample_ray(self, ray_origin, ray_direction, near, far):
        """
        Perform NeRF-ID sampling and rendering for a single ray.
        """
        # 1. Coarse Sampling
        # Generate N_c equally spaced samples in [0, 1]
        t_coarse = torch.linspace(0, 1, self.N_c, device=ray_origin.device)
        # Map to world coordinates: p = origin + t * direction * (far - near) + near * direction
        # Note: Standard NeRF parameterization often uses t in [0,1] mapped to [near, far]
        # Assuming t is normalized such that t=0 is near, t=1 is far.
        # p = ray_origin + (near + t * (far - near)) * ray_direction
        t_world_coarse = near + t_coarse * (far - near)
        points_coarse = ray_origin.unsqueeze(0) + t_world_coarse.unsqueeze(1).unsqueeze(0) * ray_direction.unsqueeze(0)
        
        # Query coarse network
        # coarse_net returns sigma, color, features
        sigma_c, color_c, features_c = self.coarse_net(points_coarse)
        
        # 2. Proposer: Generate Fine Samples
        # Input to proposer: features_c (N_c, D) and t_coarse (N_c, 1) normalized [0,1]
        # Proposer outputs N_f positions in [0,1]
        t_fine_norm = self.proposer(features_c, t_coarse) # Shape: (N_f, 1)
        
        # Map fine positions to world coordinates
        t_world_fine = near + t_fine_norm.squeeze(1) * (far - near)
        points_fine = ray_origin.unsqueeze(0) + t_world_fine.unsqueeze(1) * ray_direction.unsqueeze(0)
        
        # Query fine network
        sigma_f, color_f = self.fine_net(points_fine)
        
        # 3. Combine Samples
        # Union of coarse and fine samples
        t_all = torch.cat([t_coarse, t_fine_norm.squeeze(1)], dim=0)
        sigma_all = torch.cat([sigma_c.squeeze(1), sigma_f.squeeze(1)], dim=0)
        color_all = torch.cat([color_c.squeeze(1), color_f.squeeze(1)], dim=0)
        
        # Sort samples by depth to ensure correct transmittance calculation
        indices = torch.argsort(t_all)
        t_all = t_all[indices]
        sigma_all = sigma_all[indices]
        color_all = color_all[indices]
        
        # 4. Volume Rendering
        # Compute weights
        # dt = t[i+1] - t[i]
        dt = torch.diff(t_all, dim=0)
        # Append 0 for the last interval or handle boundary carefully
        # Standard NeRF uses t[i+1] - t[i] for the interval containing sample i
        # If t_all has N points, there are N-1 intervals. 
        # The formula w_i = T_i * (1 - exp(-sigma_i * dt_i))
        # T_i = exp(-sum_{j=1}^{i-1} sigma_j * dt_j)
        
        # Calculate alpha for each interval
        # sigma is density. dt is interval length.
        # alpha_i = 1 - exp(-sigma_i * dt_i)
        alpha = 1.0 - torch.exp(-sigma_all * dt)
        
        # Calculate transmittance T_i
        # T_1 = 1
        # T_i = T_{i-1} * (1 - alpha_{i-1})
        # We need T for each sample i. 
        # T_i = exp(-sum_{j=1}^{i-1} sigma_j * dt_j)
        # Let's compute cumulative sum of sigma * dt
        sigma_dt = sigma_all * dt
        # Cumulative sum excluding current element
        # T_i = exp(-cumsum(sigma_dt)[i-1]) if i>1 else 1
        cumsum_sigma_dt = torch.cumsum(sigma_dt, dim=0)
        # Shift for T_i: T_i corresponds to sum before i
        # T = [1, exp(-s1), exp(-(s1+s2)), ...]
        T = torch.cat([torch.tensor([1.0], device=t_all.device), torch.exp(-cumsum_sigma_dt[:-1])], dim=0)
        
        # Weights
        w = T * alpha
        
        # Rendered color
        C_hat = torch.sum(w * color_all, dim=0)
        
        return C_hat

    def train_step(self, ray_origins, ray_directions, gt_colors):
        """
        Single training step.
        """
        # Forward pass
        preds = []
        for i in range(ray_origins.shape[0]):
            # Assuming batch processing or loop over rays
            # For simplicity, assuming single ray or vectorized implementation
            pass
        
        # Vectorized implementation for batch of rays
        # ray_origins: (B, 3), ray_directions: (B, 3), gt_colors: (B, 3)
        # This requires expanding dimensions for the network calls
        
        # 1. Coarse
        t_coarse = torch.linspace(0, 1, self.N_c, device=ray_origins.device)
        t_world_coarse = ray_origins[..., None, 0] + t_coarse[None, None, :] * (ray_directions[..., None, 0] * (1.0)) # Simplified mapping, assuming normalized direction and near=0, far=1 for t
        # Correct mapping: p = o + (near + t*(far-near)) * d
        # Let's assume near=0, far=1 for t in [0,1] for simplicity of notation, 
        # but physically t_world = o + (near + t*(far-near)) * d
        # We need near/far per ray. Assuming constant near/far for simplicity or passed in.
        # Let's assume near=0, far=1 for the t parameterization in the proposer.
        # World points:
        t_world_coarse = ray_origins + (t_coarse.unsqueeze(0).unsqueeze(-1) * ray_directions) # Assuming near=0, far=1
        
        sigma_c, color_c, features_c = self.coarse_net(t_world_coarse) # (B, N_c, 3), (B, N_c, 3), (B, N_c, D)
        
        # 2. Proposer
        # t_coarse is (N_c,), needs to be broadcast to (B, N_c, 1)
        t_coarse_expanded = t_coarse.unsqueeze(0).unsqueeze(-1).expand_as(features_c[..., :1])
        t_fine_norm = self.proposer(features_c, t_coarse_expanded) # (B, N_f, 1)
        
        # 3. Fine
        t_world_fine = ray_origins + (t_fine_norm.squeeze(-1).unsqueeze(1) * ray_directions) # (B, N_f, 3)
        sigma_f, color_f = self.fine_net(t_world_fine) # (B, N_f, 3), (B, N_f, 3)
        
        # 4. Combine and Render
        t_all = torch.cat([t_coarse, t_fine_norm.squeeze(-1)], dim=-1) # (B, N_c + N_f)
        sigma_all = torch.cat([sigma_c, sigma_f], dim=-2) # (B, N_c + N_f, 3)
        color_all = torch.cat([color_c, color_f], dim=-2) # (B, N_c + N_f, 3)
        
        # Sort
        t_all_sorted, indices = torch.sort(t_all, dim=-1)
        sigma_all = torch.gather(sigma_all, -2, indices.unsqueeze(-1).expand_as(sigma_all))
        color_all = torch.gather(color_all, -2, indices.unsqueeze(-1).expand_as(color_all))
        
        dt = torch.diff(t_all_sorted, dim=-1) # (B, N-1)
        # Handle last sample? Usually N samples define N-1 intervals.
        # The formula uses t_{i+1} - t_i.
        # We have N points. We compute weights for N-1 intervals? 
        # Or we assume the last point is far plane?
        # Standard NeRF: N samples, N-1 intervals.
        # Let's stick to the formula: sum_{i=1}^N w_i c_i.
        # If we have N points, we have N-1 intervals.
        # The last sample doesn't have a forward interval in the sum if we strictly follow indices 1..N-1.
        # However, often the last sample is at the far plane, so dt=0 or it's ignored.
        # Let's assume we render using the N-1 intervals defined by the sorted points.
        
        alpha = 1.0 - torch.exp(-sigma_all[..., :-1, :] * dt.unsqueeze(-1)) # (B, N-1, 3)
        
        # Transmittance
        sigma_dt = sigma_all[..., :-1, :] * dt.unsqueeze(-1)
        cumsum = torch.cumsum(sigma_dt, dim=-2)
        T = torch.cat([torch.ones_like(sigma_dt[..., :1, :]), torch.exp(-cumsum[..., :-1, :])], dim=-2) # (B, N-1, 3)
        
        w = T * alpha # (B, N-1, 3)
        
        # Colors for the intervals
        color_intervals = color_all[..., :-1, :] # (B, N-1, 3)
        
        C_hat = torch.sum(w * color_intervals, dim=-2) # (B, 3)
        
        # Loss
        loss = F.mse_loss(C_hat, gt_colors)
        return loss
```

## Operating characteristics
*   **Time Complexity:** The proposer adds negligible computational overhead compared to the coarse and fine networks, as it is designed to be small [S1, unofficial]. The total cost is dominated by the two network queries per ray.
*   **Memory Complexity:** Requires storing features from the coarse network to pass to the proposer, increasing memory usage slightly compared to vanilla NeRF which only stores densities [S1, unofficial].
*   **Performance:** Achieves superior view synthesis quality on synthetic Blender benchmarks and on par or better performance on real LLFF-NeRF scenes compared to vanilla NeRF and other state-of-the-art methods [S1, unofficial].
*   **Strengths:** End-to-end differentiability allows the system to learn optimal sampling strategies. The predicted sample importance can be used to filter out less promising samples, achieving up to 25% computation savings without significant quality loss [S1, unofficial].
*   **Limitations:** Training from scratch is unstable; requires a two-stage training strategy (pre-training to mimic vanilla NeRF, then fine-tuning) [S1, unofficial].

## Background
NeRF-ID was introduced in the paper "NeRF in detail: Learning to sample for view synthesis" [S1, unofficial]. It addresses the limitation of the original NeRF's coarse-to-fine approach, which relied on a heuristic, non-trainable procedure to define a piece-wise constant PDF from coarse densities [S1, unofficial]. The method was developed to improve view synthesis quality by making the sampling process learnable and differentiable.

## Sources
[S1] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
