# Mixture-of-Experts Implicit Neural Representation

Also called: MoE-INR

## Overview
Mixture-of-Experts Implicit Neural Representation (MoE-INR) is an architecture for time-varying volumetric data compression that automatically subdivides spatiotemporal fields and dynamically assigns input coordinates to specialized expert networks [S1]. It solves the limitations of conventional INRs (which use a single network for the entire field) and grid-based INRs (which manually decompose the field, causing boundary discontinuities) by using a shared encoder to maintain cross-boundary continuity while allowing experts to specialize in distinct subdomains [S1]. Unlike siblings like [MIMO-NeRF-self-R2](11400-mimo-nerf-self-r2.md) which may use different routing or aggregation strategies, MoE-INR specifically targets scientific visualization datasets with a policy network that selects a single expert decoder per coordinate during inference, while using a weighted loss during training [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A spatiotemporal coordinate $\mathbf{x} = (x, y, z, t)$ and a set of ground truth voxel values $v$ for training.
*   **Output:** A predicted scalar voxel value $\hat{v}$ during inference.

**Data Structures and Layout:**
The architecture consists of three components:
1.  **Policy Network:** A sequence of fully connected (FC) layers with $\sin(\cdot)$ activation, followed by a final FC layer with Softmax activation. It outputs a probability vector $p \in \mathbb{R}^k$ over $k$ experts [S1].
2.  **Shared Encoder:** Takes $\mathbf{x}$, applies positional encoding (learnable Fourier transformation), passes through FC layers with $\sin(\cdot)$ activation to increase dimensionality, and refines the representation using a residual block with a bottleneck into a compact feature vector $h$ [S1].
3.  **Expert Decoders:** $k$ parallel networks. Each takes $h$ as input, processes it through FC layers with $\sin(\cdot)$ activation, and outputs a scalar via a final FC layer without activation [S1].

**Number of Parameters:**
Let $M$ be the number of initial neurons specified by users, and $k$ be the number of experts. The total parameter count is [S1]:
$$
\text{Params} = \underbrace{4M + M + M^2 + M + 9M \times k + k}_{\text{Policy Network}} + \underbrace{4M + M + 8M^2 + 4M + 32M^2 + 8M + 36M^2 + 12M}_{\text{Shared Encoder}} + \underbrace{k \times (8M + 1)}_{\text{Expert Decoders}}
$$
The "Larger Encoder" allocation strategy is recommended, where the majority of parameters are assigned to the shared encoder (e.g., 96.90% for the vortex dataset) to assist both policy assignment and feature representation [S1].

**Algorithm Steps:**

1.  **Pre-training (Initialization):**
    *   Optimize the Policy Network and Shared Encoder only.
    *   **Supervised Pre-training:** Generate ground truth expert assignments $y_c$ for each coordinate $c$ using one of four methods:
        *   *Random Partition:* Random assignment.
        *   *Spatial Partition:* $k$ non-overlapping spatial blocks.
        *   *Temporal Partition:* $k$ non-overlapping time intervals.
        *   *Voxel Clustering:* $k$-means clustering on voxels (recommended as it yields best performance) [S1].
    *   Minimize Cross-Entropy Loss:
        $$
        \mathcal{L}_{pre} = -\frac{1}{k} \sum_{i=1}^k \log \left( \frac{e^{p_i}}{\sum_{j=1}^k e^{p_j}} \right) \quad \text{for assigned expert } i \text{ (where } y_c=i \text{)}
        $$
        *Note: The source formula [S1] is written as $- \frac{1}{k} \frac{e^{p_i}}{\text{sum}(e^p)}$ inside a sum context, but standard cross-entropy for classification is $-\log(\text{softmax}(p_i))$. The text explicitly states "Cross-Entropy Loss" and defines $p_j$ as probability. The formula in the notes appears to be a typo for the log-softmax term. We implement standard cross-entropy.*
    *   **Unsupervised Pre-training (Alternative):** Use Load Balancing Loss to encourage even distribution:
        $$
        \mathcal{L}_{bal} = \frac{1}{N^2} \sum_{j=1}^k c_j s_j
        $$
        where $c_j$ is the count of coordinates assigned to expert $j$, $s_j$ is the sum of assignment probabilities to expert $j$ for all coordinates, and $N$ is total coordinates. *Note: Sources indicate this performs worse than no pre-training or supervised methods* [S1].

2.  **Joint Training:**
    *   Optimize Policy Network, Shared Encoder, and all Expert Decoders jointly.
    *   For each coordinate $\mathbf{x}$, compute probabilities $p$ via Policy Network and features $h$ via Shared Encoder.
    *   Compute predictions $\hat{v}_j$ from each Expert Decoder $j$ using $h$.
    *   Minimize Weighted MSE Loss:
        $$
        \mathcal{L}_{train} = \sum_{j=1}^k p_j \| \hat{v}_j - v \|^2
        $$
        where $v$ is the ground truth voxel value [S1].

3.  **Inference:**
    *   Input $\mathbf{x}$.
    *   Compute $p$ and $h$.
    *   Select the expert $j^*$ with the highest probability: $j^* = \arg\max_j p_j$.
    *   Output $\hat{v} = \text{ExpertDecoder}_{j^*}(h)$ [S1].
    *   *Note: While ensemble averaging is possible, the standard MoE-INR inference uses top-1 selection* [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from sklearn.cluster import KMeans

class SinActivation(nn.Module):
    def forward(self, x):
        return torch.sin(x)

class SharedEncoder(nn.Module):
    def __init__(self, input_dim=4, hidden_dim=256, output_dim=256):
        super().__init__()
        # Positional Encoding (Learnable Fourier Transformation)
        # Simplified as a linear projection for implementation clarity, 
        # as source specifies "learnable Fourier transformation" [S1]
        self.pe = nn.Linear(input_dim, hidden_dim)
        
        # FC layers with sin activation
        self.fc1 = nn.Linear(hidden_dim, hidden_dim)
        self.act1 = SinActivation()
        
        # Residual block with bottleneck
        self.bottleneck = nn.Linear(hidden_dim, hidden_dim // 2)
        self.act2 = SinActivation()
        self.fc_out = nn.Linear(hidden_dim // 2, output_dim)
        
    def forward(self, x):
        h = self.pe(x)
        h = self.act1(self.fc1(h))
        h_res = self.act2(self.bottleneck(h))
        h = self.fc_out(h_res) + h # Residual connection implied by "refine... compact"
        return h

class PolicyNetwork(nn.Module):
    def __init__(self, input_dim=4, hidden_dim=256, num_experts=7):
        super().__init__()
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.act1 = SinActivation()
        self.fc2 = nn.Linear(hidden_dim, hidden_dim)
        self.act2 = SinActivation()
        self.fc_out = nn.Linear(hidden_dim, num_experts)
        
    def forward(self, x):
        h = self.act1(self.fc1(x))
        h = self.act2(self.fc2(h))
        logits = self.fc_out(h)
        return F.softmax(logits, dim=-1)

class ExpertDecoder(nn.Module):
    def __init__(self, input_dim=256, hidden_dim=256):
        super().__init__()
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.act1 = SinActivation()
        self.fc2 = nn.Linear(hidden_dim, hidden_dim)
        self.act2 = SinActivation()
        self.fc_out = nn.Linear(hidden_dim, 1) # Scalar output
        
    def forward(self, h):
        h = self.act1(self.fc1(h))
        h = self.act2(self.fc2(h))
        return self.fc_out(h)

class MoEINR(nn.Module):
    def __init__(self, num_experts=7, M=256):
        super().__init__()
        self.policy = PolicyNetwork(input_dim=4, hidden_dim=M, num_experts=num_experts)
        self.encoder = SharedEncoder(input_dim=4, hidden_dim=M, output_dim=M)
        self.experts = nn.ModuleList([
            ExpertDecoder(input_dim=M, hidden_dim=M) for _ in range(num_experts)
        ])
        self.num_experts = num_experts

    def forward(self, x):
        """
        x: (B, 4) coordinates
        Returns: (B, 1) predicted values
        """
        # 1. Policy Network
        probs = self.policy(x) # (B, k)
        
        # 2. Shared Encoder
        h = self.encoder(x) # (B, M)
        
        # 3. Expert Processing (Inference mode: Top-1)
        # Select expert with highest probability
        expert_indices = torch.argmax(probs, dim=-1) # (B,)
        
        # Gather features for selected experts
        # We need to route h to the correct expert
        # Since experts are separate modules, we iterate or use gather
        # Efficient implementation:
        batch_size = x.size(0)
        predictions = torch.zeros(batch_size, 1, device=x.device)
        
        for j in range(self.num_experts):
            mask = (expert_indices == j)
            if mask.any():
                h_expert = h[mask]
                pred = self.experts[j](h_expert)
                predictions[mask] = pred
                
        return predictions

    def training_step(self, x, v_gt):
        """
        x: (B, 4)
        v_gt: (B, 1)
        Returns: Loss
        """
        # 1. Policy Network
        probs = self.policy(x) # (B, k)
        
        # 2. Shared Encoder
        h = self.encoder(x) # (B, M)
        
        # 3. Expert Processing (Training mode: Weighted Sum)
        # Compute predictions from ALL experts
        all_preds = []
        for j in range(self.num_experts):
            pred = self.experts[j](h) # (B, 1)
            all_preds.append(pred)
        all_preds = torch.stack(all_preds, dim=1) # (B, k, 1)
        
        # 4. Loss Calculation
        # L = sum_j p_j * ||v_j - v||^2
        # probs: (B, k), all_preds: (B, k, 1), v_gt: (B, 1)
        # Expand v_gt to (B, k, 1)
        v_gt_expanded = v_gt.unsqueeze(1).expand_as(all_preds)
        
        mse_per_expert = (all_preds - v_gt_expanded) ** 2 # (B, k, 1)
        weighted_mse = probs.unsqueeze(-1) * mse_per_expert # (B, k, 1)
        loss = weighted_mse.sum(dim=1).mean() # Mean over batch
        
        return loss

    def pretrain_supervised(self, x, v_gt, expert_assignments_gt):
        """
        expert_assignments_gt: (B,) integer indices of the correct expert for each coord
        """
        probs = self.policy(x)
        # Cross Entropy Loss
        # probs: (B, k), expert_assignments_gt: (B,)
        loss = F.cross_entropy(probs, expert_assignments_gt)
        
        # Also train encoder to produce useful features? 
        # Source says "optimize policy network and shared encoder only"
        # But loss is defined on assignment. 
        # To train encoder, we might need a reconstruction loss or just rely on 
        # the joint training later. 
        # However, source says "Pre-training... aims to offer good initialization".
        # We will return the assignment loss. 
        # Note: Source does not specify a reconstruction loss for pre-training, 
        # only the cross-entropy for assignment.
        return loss
```

**Parameters:**
*   $M$: Number of initial neurons. Determines network width.
*   $k$: Number of experts. Optimal value is 7 for many datasets (e.g., Ionization PD dataset at CR 10,078 achieved 58.76 dB PSNR with $k=7$) [S1].
*   Pre-training Scheme: Voxel Clustering (k-means) is recommended for supervised pre-training [S1].

**Edge Cases and Pitfalls:**
*   **Too many experts:** Increasing $k$ beyond the optimal point (e.g., $>7$ in tested scenarios) degrades performance because the policy network struggles to correctly classify coordinates, leading to misclassifications [S1].
*   **Bad Pre-training:** Random partition and load balancing pre-training schemes fail to produce meaningful assignments, resulting in inferior performance compared to no pre-training [S1].
*   **Boundary Continuity:** Unlike grid-based INRs, MoE-INR uses a shared encoder to preserve boundary continuity, avoiding block artifacts [S1].

## Subtopics

### Grid-based INR
Manually decomposes the field into spatial blocks with independent encoders/decoders, causing boundary discontinuity. MoE-INR automates this with a shared encoder [S1].

### Conventional INR
Uses a single network for the entire field. MoE-INR with $k=1$ reduces to this, but offers better scalability with $k>1$ [S1].

### Ensemble INR
Decodes a single coordinate using multiple experts and averages results. MoE-INR can function as an ensemble INR but typically uses top-1 selection for efficiency [S1].

## Operating characteristics
*   **Time Complexity:** Visualization cost is lower than conventional INRs (which rely entirely on matrix multiplications) but higher than grid-based INRs (which use efficient interpolation) [S1]. Compression time (CT) increases with the number of experts $k$ (e.g., 21.59 hours for $k=7$ vs 11.83 hours for $k=2$ on Ionization PD dataset) [S1].
*   **Memory Usage:** Memory usage (MU) scales with $k$ and $M$. For Ionization PD dataset, MU ranges from 8,177 MB ($k=2$) to 21,745 MB ($k=9$) [S1].
*   **Performance:** MoE-INR significantly outperforms existing non-MoE and MoE-based INRs and traditional lossy compression methods across PSNR and LPIPS metrics [S1].
*   **Strengths:** Automatic irregular subdivision, cross-boundary continuity via shared encoder, unified framework for various INR types [S1].
*   **Limitations:** Policy network misclassification with too many experts; sensitivity to pre-training scheme [S1].
*   **When to use:** Time-varying volumetric data compression in scientific visualization (e.g., argon bubble, combustion, ionization, Tangaroa, vortex datasets) [S1].

## Background
The mixture-of-experts (MoE) paradigm was originally proposed by Nowlan and Hinton [26], applying supervision to a system of separate networks where each handles a subset of training samples and specializes in a different part of the data space [S1]. MoE-INR was developed to address limitations in scientific data compression, specifically the boundary discontinuities of grid-based INRs and the computational cost of conventional INRs [S1]. It is used for time-varying volumetric data compression and representation in scientific datasets [S1].

## Sources
[S1] MoE-INR: Implicit Neural Representation with Mixture-of-Experts for ..., https://academicweb.nd.edu/~cwang11/papers/vis25-moeinr.pdf, fetched 2026-10-29
