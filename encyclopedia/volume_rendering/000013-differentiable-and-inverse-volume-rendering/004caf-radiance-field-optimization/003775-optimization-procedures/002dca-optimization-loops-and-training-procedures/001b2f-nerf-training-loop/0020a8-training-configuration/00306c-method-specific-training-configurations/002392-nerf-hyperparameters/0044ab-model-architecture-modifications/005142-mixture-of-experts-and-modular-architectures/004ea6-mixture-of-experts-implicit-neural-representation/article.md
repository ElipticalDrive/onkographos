# Mixture-of-Experts Implicit Neural Representation

Also called: MoE-INR

## Overview
MoE-INR is an implicit neural representation architecture that uses a mixture-of-experts framework to automatically subdivide spatiotemporal fields and dynamically assign input coordinates to specialized expert networks for time-varying volumetric data compression [S1]. Unlike conventional INRs that use a single network for the entire field or grid-based INRs that manually decompose the field into spatial blocks (causing boundary discontinuity), MoE-INR automates irregular subdivisions and uses a shared encoder to maintain cross-boundary continuity [S1]. It is designed for scientific visualization of time-varying volumetric data, accommodating conventional, grid-based, and ensemble INR types [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A coordinate vector $\mathbf{c} = (x, y, z, t)$ representing a point in the spatiotemporal domain.
*   **Output:** A scalar value $v$ (e.g., voxel intensity) at the input coordinate.

**Data Structures:**
The architecture consists of three components:
1.  **Policy Network:** A Multi-Layer Perceptron (MLP) with fully connected (FC) layers using $\sin(\cdot)$ activation, ending with an FC layer using $\text{softmax}$ activation.
2.  **Shared Encoder:** An MLP with positional encoding (learnable Fourier transformation), FC layers with $\sin(\cdot)$ activation, and a residual block with a bottleneck.
3.  **Expert Decoders:** $k$ parallel MLPs, each with FC layers using $\sin(\cdot)$ activation and a final FC layer without activation.

**Parameters:**
*   $M$: Number of initial neurons specified by users.
*   $k$: Number of expert decoders. Optimal value is 7 for the ionization (PD) dataset [S1].

**Numbered Steps:**

1.  **Pre-training (Optional but Recommended):**
    *   Optimize the policy network and shared encoder only.
    *   **Supervised Pre-training:** Generate ground truth expert assignments $y_c$ for each coordinate $c$. Common methods include:
        *   *Random partition:* Random expert assignment.
        *   *Spatial partition:* $k$ non-overlapping spatial blocks.
        *   *Temporal partition:* $k$ non-overlapping time intervals.
        *   *Voxel clustering:* $k$-means clustering on voxels.
    *   **Unsupervised Pre-training:** Use load balancing to encourage even distribution of coordinates among experts.
    *   **Loss:**
        *   Supervised (Cross-Entropy):
            $$ L_{sup} = - \frac{1}{k} \sum_{i=1}^{k} \log \left( \frac{e^{p_i}}{\sum_{j=1}^{k} e^{p_j}} \right) \quad \text{for } i = y_c $$
            Where $p$ is the 1D vector of $k$ elements outputted from the policy network, and $p_j$ is the probability of assigning to the $j$-th expert.
        *   Unsupervised (Load Balancing):
            $$ L_{unsup} = \frac{1}{N^2} \sum_{j=1}^{k} c_j s_j $$
            Where $c_j$ is the number of coordinates assigned to the $j$-th expert, $s_j$ is the sum of assignment probabilities to the $j$-th expert for all coordinates, and $N$ is the total number of coordinates.

2.  **Shared Encoder Forward Pass:**
    *   Map input coordinate $\mathbf{c}$ to a high-dimensional representation using positional encoding (learnable Fourier transformation).
    *   Pass embeddings through FC layers with $\sin(\cdot)$ activation, increasing feature dimensionality.
    *   Refine the representation into a compact vector $\mathbf{h}$ using a residual block with a bottleneck.

3.  **Policy Network Forward Pass:**
    *   Process input coordinate $\mathbf{c}$ through FC layers with $\sin(\cdot)$ activation.
    *   Concatenate the policy network's output features with the shared encoder's output $\mathbf{h}$.
    *   Feed the concatenated representation into the last FC layer.
    *   Apply $\text{softmax}$ activation to convert features into probabilities $p_1, \dots, p_k$.
    *   Select the expert decoder $j^*$ with the highest probability: $j^* = \arg\max_j p_j$.

4.  **Expert Decoder Forward Pass:**
    *   Take the high-dimensional representation $\mathbf{h}$ from the shared encoder as input.
    *   Process $\mathbf{h}$ through FC layers with $\sin(\cdot)$ activation.
    *   Output the predicted scalar value $v_{j^*}$ using the final FC layer (no activation).
    *   Only the single expert decoder $j^*$ selected by the policy network is activated.

5.  **Training Stage:**
    *   Jointly optimize the policy network, shared encoder, and all expert decoders.
    *   **Loss Function:** Mean Squared Error (MSE) weighted by policy probabilities:
        $$ L_{train} = \sum_{j=1}^{k} p_j ||v_j - v||^2 $$
        Where $v_j$ is the predicted value from the $j$-th expert decoder, $v$ is the ground truth voxel value, and $p_j$ is the probability of assigning to the $j$-th expert.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SinActivation(nn.Module):
    def forward(self, x):
        return torch.sin(x)

class PositionalEncoding(nn.Module):
    """Learnable Fourier transformation for positional encoding."""
    def __init__(self, input_dim, output_dim):
        super().__init__()
        # Learnable frequency matrix
        self.freqs = nn.Parameter(torch.randn(input_dim, output_dim // 2) * 0.1)
        
    def forward(self, x):
        # x shape: (..., input_dim)
        # Output shape: (..., output_dim)
        # Standard PE: [x, sin(2^d * pi * x), cos(2^d * pi * x)]
        # Here we use a learnable linear projection of sin/cos of random frequencies
        # Simplified to linear projection for "learnable Fourier" as per source description
        # Note: Source says "learnable Fourier transformation", implemented as linear layer 
        # on sin/cos of fixed or learnable frequencies. 
        # For implementation clarity, we use a linear layer as the primary "learnable" part 
        # mapping low-dim to high-dim, potentially preceded by fixed sin/cos if strictly following NeRF PE.
        # However, source emphasizes "learnable", so we assume a linear map from a fixed PE or direct learnable map.
        # Let's assume standard NeRF-like PE with learnable scaling or just a linear layer if "learnable Fourier" implies the weights are learned.
        # Given the ambiguity, we use a linear layer as the core "learnable" component of the encoding.
        return F.linear(x, self.freqs.t())

class ResidualBlock(nn.Module):
    def __init__(self, dim):
        super().__init__()
        self.bottleneck = nn.Sequential(
            nn.Linear(dim, dim // 2),
            SinActivation(),
            nn.Linear(dim // 2, dim),
            SinActivation()
        )
        
    def forward(self, x):
        return x + self.bottleneck(x)

class PolicyNetwork(nn.Module):
    def __init__(self, input_dim, hidden_dim, num_experts):
        super().__init__()
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.fc2 = nn.Linear(hidden_dim, hidden_dim)
        self.fc3 = nn.Linear(hidden_dim, num_experts)
        
    def forward(self, x):
        x = F.sin(self.fc1(x))
        x = F.sin(self.fc2(x))
        logits = self.fc3(x)
        probs = F.softmax(logits, dim=-1)
        return probs

class SharedEncoder(nn.Module):
    def __init__(self, input_dim, hidden_dim, output_dim):
        super().__init__()
        self.pos_enc = PositionalEncoding(input_dim, hidden_dim)
        self.fc1 = nn.Linear(hidden_dim, hidden_dim)
        self.fc2 = nn.Linear(hidden_dim, hidden_dim)
        self.residual = ResidualBlock(hidden_dim)
        self.fc_out = nn.Linear(hidden_dim, output_dim)
        
    def forward(self, x):
        x = self.pos_enc(x)
        x = F.sin(self.fc1(x))
        x = F.sin(self.fc2(x))
        x = self.residual(x)
        h = self.fc_out(x)
        return h

class ExpertDecoder(nn.Module):
    def __init__(self, input_dim, hidden_dim):
        super().__init__()
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.fc2 = nn.Linear(hidden_dim, hidden_dim)
        self.fc_out = nn.Linear(hidden_dim, 1)
        
    def forward(self, h):
        x = F.sin(self.fc1(h))
        x = F.sin(self.fc2(x))
        v = self.fc_out(x)
        return v

class MoEINR(nn.Module):
    def __init__(self, input_dim=4, hidden_dim=128, num_experts=7, encoder_output_dim=256):
        super().__init__()
        self.policy = PolicyNetwork(input_dim, hidden_dim, num_experts)
        self.encoder = SharedEncoder(input_dim, hidden_dim, encoder_output_dim)
        self.experts = nn.ModuleList([
            ExpertDecoder(encoder_output_dim, hidden_dim) for _ in range(num_experts)
        ])
        self.num_experts = num_experts
        
    def forward(self, x):
        # x shape: (..., 4)
        # 1. Shared Encoder
        h = self.encoder(x)
        
        # 2. Policy Network
        # Policy takes raw coordinate x to determine expert
        probs = self.policy(x)
        
        # 3. Select Expert
        # Get index of max probability
        expert_indices = torch.argmax(probs, dim=-1)
        
        # 4. Decode
        # Gather outputs from selected experts
        # We need to handle batch dimensions. 
        # For simplicity, assume x is (N, 4)
        N = x.shape[0]
        selected_experts = [self.experts[i] for i in expert_indices]
        
        # Process each expert separately or vectorize if possible. 
        # Vectorizing across experts is complex due to different weights.
        # We iterate or use advanced indexing.
        
        # Simple loop for clarity (can be optimized)
        predictions = torch.zeros(N, 1, device=x.device)
        for i in range(N):
            expert_idx = expert_indices[i].item()
            predictions[i] = selected_experts[i](h[i])
            
        return predictions, probs

def compute_loss(predictions, ground_truth, probs):
    # L_train = sum(p_j * ||v_j - v||^2)
    # Note: In the forward pass, we only output the selected expert's prediction.
    # However, the loss formula implies using all experts' predictions weighted by probability.
    # To implement this, we need to compute outputs for all experts.
    pass 
```

*Note: The pseudocode above simplifies the loss computation. In practice, to compute $L_{train} = \sum_{j=1}^{k} p_j ||v_j - v||^2$, the forward pass must compute $v_j$ for all $k$ experts, not just the selected one. The selection step is for inference or specific routing, but training requires all outputs.*

**Corrected Forward Pass for Training:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
    def forward_train(self, x):
        # x shape: (N, 4)
        h = self.encoder(x) # (N, encoder_output_dim)
        probs = self.policy(x) # (N, k)
        
        # Compute outputs for all experts
        # expert_outputs shape: (N, k, 1)
        expert_outputs = torch.stack([expert(h) for expert in self.experts], dim=1)
        
        # Weighted sum for prediction (optional, but loss uses individual terms)
        # For loss calculation, we keep individual expert outputs
        
        return expert_outputs, probs
```

**Edge Cases and Pitfalls:**
*   **Too Many Experts:** Using more than 7 experts (in tested scenarios) can degrade performance because the policy network struggles to correctly classify and assign coordinates, leading to misclassifications [S1].
*   **Pre-training Schemes:** Random partition and load balancing pre-training schemes fail to produce meaningful assignments, resulting in inferior performance compared to no pre-training [S1]. Voxel clustering is the recommended supervised pre-training method [S1].
*   **Parameter Allocation:** Allocating too few parameters to the shared encoder degrades performance. The "Larger encoder" strategy (allocating most parameters to the encoder) is optimal because the encoder assists the policy network and provides compact representation [S1].

## Operating characteristics
*   **Time and Memory Complexity:**
    *   Memory Usage (MU) increases with the number of experts $k$. For the ionization (PD) dataset (CR 10,078), MU ranges from 8,177 MB (2 experts) to 21,745 MB (9 experts) [S1].
    *   Compression Time (CT) also increases with $k$. For the same dataset, CT ranges from 11.83 hours (2 experts) to 25.36 hours (9 experts) [S1].
*   **Measured Performance:**
    *   **Vortex Dataset (CR 5,065):** Larger encoder allocation achieved 96.90% of parameters in the encoder, 0.82% in experts, and 2.28% in the policy network. Best PSNR/LPIPS [S1].
    *   **Ionization (PD) Dataset (CR 10,078):** 7 experts yielded the best PSNR (58.76 dB) and LPIPS (0.024). PSNR decreased slightly for >7 experts [S1].
    *   **Ionization (T) Dataset (CR 5,033, 7 experts):** Voxel clustering pre-training achieved the best PSNR (56.71 dB) and LPIPS (0.047) [S1].
*   **Strengths:**
    *   Automates irregular subdivisions of spatiotemporal fields.
    *   Maintains cross-boundary continuity via a shared encoder, avoiding block artifacts common in grid-based INRs.
    *   Unifies conventional, grid-based, and ensemble INR types.
*   **Limitations:**
    *   Performance degrades if the number of experts is too high due to policy network misclassification.
    *   Requires careful selection of pre-training scheme (voxel clustering recommended).
*   **When to Use:**
    *   Time-varying volumetric data compression and representation in scientific visualization.
    *   Datasets with complex, time-varying patterns (e.g., argon bubble, combustion, ionization, Tangaroa, vortex) [S1].
*   **When Not to Use:**
    *   Static volumetric data where a single conventional INR might suffice.
    *   Scenarios where computational resources are extremely limited and $k$ must be small, potentially sacrificing accuracy.

## Background
*   **Origin:** The mixture-of-experts (MoE) paradigm was originally proposed by Nowlan and Hinton [26], applying supervision to a system of separate networks where each handles a subset of training samples and specializes in a different part of the data space [S1].
*   **Development:** MoE-INR was developed to address limitations in existing INRs for scientific data compression, such as the inability of conventional INRs to preserve complex features and the boundary discontinuity of grid-based INRs [S1].
*   **Usage:** Used for time-varying volumetric data compression and representation in scientific datasets including argon bubble, combustion, ionization, Tangaroa, and vortex [S1].

## Sources
[S1] MoE-INR: Implicit Neural Representation with Mixture-of-Experts for ..., https://academicweb.nd.edu/~cwang11/papers/vis25-moeinr.pdf, fetched 2026-10-29
