# Mixture-of-Experts Implicit Neural Representation

Also called: MoE-INR

## Overview
Mixture-of-Experts Implicit Neural Representation (MoE-INR) is an architecture for time-varying volumetric data compression that automatically subdivides spatiotemporal fields and dynamically assigns input coordinates to specialized expert networks [S1]. It solves the limitations of conventional INRs (which struggle with complex features using a single network) and grid-based INRs (which suffer from boundary discontinuities due to manual decomposition) by using a shared encoder to maintain cross-boundary continuity while allowing experts to specialize in distinct subdomains [S1]. Unlike siblings like [MIMO-NeRF-self-R2](11400-mimo-nerf-self-r2.md) which may use multiple networks for different aspects of rendering, MoE-INR specifically targets the compression and representation of scientific volumetric data through a unified framework that can emulate conventional, grid-based, or ensemble INRs by adjusting the number of experts and pre-training schemes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 4D coordinate $\mathbf{x} = (x, y, z, t)$ representing a point in the spatiotemporal volume.
*   **Output:** A scalar value $v$ (e.g., voxel intensity) at that coordinate.

**Data Structures and Layout:**
The architecture consists of three components:
1.  **Policy Network:** A sequence of fully connected (FC) layers with $\sin(\cdot)$ activation, ending with a final FC layer with $\text{softmax}$ activation. It outputs a probability vector $\mathbf{p} \in \mathbb{R}^k$.
2.  **Shared Encoder:** Consists of positional encoding (learnable Fourier transformation), FC layers with $\sin(\cdot)$ activation, and a residual block with a bottleneck. It maps the input coordinate to a high-dimensional compact representation $\mathbf{h}$.
3.  **Expert Decoders:** $k$ parallel networks. Each consists of FC layers with $\sin(\cdot)$ activation and a final FC layer with no activation. Each decoder outputs a scalar prediction.

**Parameters:**
*   $M$: Number of initial neurons specified by the user.
*   $k$: Number of expert decoders. Optimal value is 7 for tested datasets [S1].

**Total Parameter Count:**
The total number of parameters is calculated as [S1]:
$$
\begin{aligned}
\text{Params} = & \underbrace{4M + M + M^2 + M + 9M \times k + k}_{\text{Policy Network}} \\
& + \underbrace{4M + M + 8M^2 + 4M + 32M^2 + 8M + 36M^2 + 12M}_{\text{Shared Encoder}} \\
& + \underbrace{k \times (8M + 1)}_{\text{Expert Decoders}}
\end{aligned}
$$
The "Larger Encoder" allocation strategy is recommended, where the majority of parameters are assigned to the shared encoder (e.g., 96.90% on the Vortex dataset) because the encoder assists in policy assignment and provides the representation for decoding [S1].

**Algorithm Steps:**

1.  **Pre-training (Optional but Recommended):**
    *   Optimize only the Policy Network and Shared Encoder.
    *   **Supervised Pre-training:** Generate ground truth assignments $y_c$ for coordinates using one of four methods:
        *   *Random Partition:* Random expert assignment.
        *   *Spatial Partition:* $k$ non-overlapping spatial blocks.
        *   *Temporal Partition:* $k$ non-overlapping time intervals.
        *   *Voxel Clustering:* $k$-means clustering on voxels (Best performing) [S1].
    *   **Loss:** Cross-Entropy Loss:
        $$
        \mathcal{L}_{pre} = - \frac{1}{k} \sum_{i=1}^{k} \log \left( \frac{e^{p_i}}{\sum_{j=1}^{k} e^{p_j}} \right) \quad \text{where } p_i \text{ corresponds to } y_c
        $$
        *Note: The source formula is written as $- \frac{1}{k} \frac{e^{p_i}}{\text{sum}(e^p)}$ but context implies standard cross-entropy with log. The text explicitly states "cross-entropy loss" and provides the vector $p$.*
    *   **Unsupervised Pre-training (Load Balancing):**
        $$
        \mathcal{L}_{bal} = \frac{1}{N^2} \sum_{j=1}^{k} c_j s_j
        $$
        where $c_j$ is the number of coordinates assigned to expert $j$, $s_j$ is the sum of assignment probabilities to expert $j$ for all coordinates, and $N$ is the total number of coordinates. *Note: Random partition and load balancing pre-training schemes were found to perform worse than no pre-training [S1].*

2.  **Training (Joint Optimization):**
    *   Optimize Policy Network, Shared Encoder, and all Expert Decoders jointly.
    *   **Forward Pass:**
        1.  Input $\mathbf{x}$ passes through the Shared Encoder to produce representation $\mathbf{h}$.
        2.  Input $\mathbf{x}$ passes through the Policy Network to produce probabilities $\mathbf{p}$.
        3.  Concatenate $\mathbf{p}$ and $\mathbf{h}$ (or feed $\mathbf{h}$ to experts as per description "expert decoders take the high-dimensional representation... as input"). The text states: "The concatenated representation... will be fed into the last fully-connected layer [of the policy network? No, the policy network outputs probabilities]. The decoder corresponding to the highest probability is then selected."
        4.  Select the expert $j^*$ with the highest probability $p_{j^*} = \max_j p_j$.
        5.  Pass $\mathbf{h}$ to the selected expert decoder $j^*$ to get prediction $v_{j^*}$.
    *   **Loss:** Mean Squared Error weighted by policy probabilities:
        $$
        \mathcal{L}_{train} = \sum_{j=1}^{k} p_j ||v_j - v||^2
        $$
        where $v_j$ is the predicted value from expert $j$, and $v$ is the ground truth voxel value. *Note: Although only one expert is "activated" for inference, the training loss sums over all experts weighted by their selection probability, allowing gradient flow to all experts [S1].*

3.  **Inference:**
    *   For a given coordinate, compute $\mathbf{p}$ and $\mathbf{h}$.
    *   Select the expert $j^*$ with $\max_j p_j$.
    *   Output $v_{j^*}$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

class MoE_INR:
    def __init__(self, M, k):
        """
        M: Number of initial neurons.
        k: Number of experts.
        """
        self.k = k
        self.M = M
        # Initialize weights for Policy Network, Shared Encoder, and k Expert Decoders
        # Details of weight initialization are not specified in sources [G: Use standard Xavier/He init]
        self.policy_net = PolicyNetwork(M)
        self.shared_encoder = SharedEncoder(M)
        self.experts = [ExpertDecoder(M) for _ in range(k)]

    def forward(self, coords):
        """
        coords: Tensor of shape (N, 4) containing (x, y, z, t)
        Returns: Predicted scalar values
        """
        # 1. Shared Encoder
        # Positional Encoding (Learnable Fourier Transformation)
        h_enc = self.shared_encoder.encode(coords)
        
        # 2. Policy Network
        # Outputs probabilities p of shape (N, k)
        p = self.policy_net(coords, h_enc)
        
        # 3. Select Expert
        # Get indices of max probability for each coordinate
        expert_indices = np.argmax(p, axis=1)
        
        # 4. Decode
        # Gather the selected expert for each coordinate
        # Note: In practice, this might be batched or handled via masking
        predictions = np.zeros(coords.shape[0])
        for i in range(coords.shape[0]):
            expert_idx = expert_indices[i]
            predictions[i] = self.experts[expert_idx].forward(h_enc[i])
            
        return predictions

    def compute_loss(self, coords, ground_truth):
        """
        coords: Tensor of shape (N, 4)
        ground_truth: Tensor of shape (N,)
        Returns: Loss scalar
        """
        # 1. Shared Encoder
        h_enc = self.shared_encoder.encode(coords)
        
        # 2. Policy Network
        p = self.policy_net(coords, h_enc) # Shape (N, k)
        
        # 3. All Experts Forward Pass
        # Compute predictions for ALL experts to allow gradient flow
        all_predictions = np.zeros((coords.shape[0], self.k))
        for j in range(self.k):
            all_predictions[:, j] = self.experts[j].forward(h_enc)
            
        # 4. Loss Calculation
        # MSE weighted by probabilities
        # diff shape (N, k)
        diff = all_predictions - ground_truth[:, np.newaxis]
        mse = np.square(diff)
        
        # Weighted sum
        # p shape (N, k)
        weighted_mse = p * mse
        loss = np.mean(np.sum(weighted_mse, axis=1))
        
        return loss

class PolicyNetwork:
    def __init__(self, M):
        # FC layers with sin activation, final FC with softmax
        pass
    def __call__(self, coords, h_enc):
        # Process coords and h_enc
        # Return softmax probabilities
        pass

class SharedEncoder:
    def __init__(self, M):
        # Positional encoding, FC layers with sin, residual block
        pass
    def encode(self, coords):
        # Return high-dimensional representation h
        pass

class ExpertDecoder:
    def __init__(self, M):
        # FC layers with sin, final FC no activation
        pass
    def forward(self, h):
        # Return scalar prediction
        pass
```

**Edge Cases and Pitfalls:**
*   **Too Many Experts:** Using more than 7 experts (in tested scenarios) degrades performance because the policy network struggles to correctly classify and assign coordinates, leading to misclassifications [S1].
*   **Pre-training Schemes:** Random partition and load balancing pre-training schemes fail to produce meaningful assignments and result in inferior performance compared to no pre-training. Voxel clustering is the recommended supervised pre-training method [S1].
*   **Boundary Continuity:** Unlike grid-based INRs, MoE-INR uses a shared encoder to preserve boundary continuity, avoiding block artifacts [S1].

## Operating characteristics
*   **Time Complexity:** Visualization (inference) cost is lower than conventional INRs but higher than grid-based INRs. Conventional INRs rely entirely on matrix multiplications. Grid-based INRs use efficient feature interpolation. MoE-INR involves a policy network forward pass and one expert decoder forward pass per coordinate [S1].
*   **Memory Complexity:** Memory usage scales with the number of experts $k$. For the Ionization dataset (CR 10,078), memory usage increased from 8,177 MB (2 experts) to 21,745 MB (9 experts) [S1].
*   **Performance:** On the Ionization (PD) dataset (CR 10,078), 7 experts achieved the best PSNR (58.76 dB) and LPIPS (0.024) [S1]. On the Vortex dataset (CR 5,065), the "Larger Encoder" allocation achieved the best PSNR/LPIPS, with 96.90% of parameters in the encoder [S1].
*   **Strengths:** Automates irregular subdivisions, maintains cross-boundary continuity via shared encoder, accommodates conventional, grid-based, and ensemble INR types [S1].
*   **Limitations:** Performance degrades if $k$ is too high (>7 in tests). Pre-training is sensitive to the scheme used [S1].

## Background
The mixture-of-experts (MoE) paradigm was originally proposed by Nowlan and Hinton [26], applying supervision to a system of separate networks where each handles a subset of training samples and specializes in a different part of the data space [S1]. MoE-INR was developed to address the limitations of existing INRs for scientific data compression, specifically the difficulty of preserving complex features in conventional INRs and the boundary discontinuity in grid-based INRs [S1]. It is used for time-varying volumetric data compression and representation in scientific datasets including argon bubble, combustion, ionization, Tangaroa, and vortex [S1].

## Sources
[S1] MoE-INR: Implicit Neural Representation with Mixture-of-Experts for ..., https://academicweb.nd.edu/~cwang11/papers/vis25-moeinr.pdf, fetched 2026-10-29
