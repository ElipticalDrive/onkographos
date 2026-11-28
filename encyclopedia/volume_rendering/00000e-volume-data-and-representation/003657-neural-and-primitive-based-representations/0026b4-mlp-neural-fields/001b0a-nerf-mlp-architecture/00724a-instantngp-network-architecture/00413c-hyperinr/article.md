# HyperINR

Also called: Hypernetwork for Implicit Neural Representations

## Overview
HyperINR is a hypernetwork architecture that directly predicts the weights of a compact Implicit Neural Representation (INR) conditioned on scene parameters, enabling interactive volumetric path tracing with inference times less than 1ms [S1, S2, unofficial]. It differs from standard INRs, which require millions of operations per forward pass and are too slow for interactive visualization, and from data interpolation methods, which operate on data values rather than network weights and are less accurate for unseen parameters [S1, unofficial]. By using a shared synthesis function and interpolating encoding weights via an ensemble of multi-resolution hash encoders organized in a KD-tree, HyperINR achieves up to 100× faster inference than teacher models like CoordNet while maintaining high reconstruction quality [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Scene parameters $\theta$ (e.g., viewpoint, time, lighting direction) belonging to a parameter space $\Theta$.
*   **Output:** A set of weights for a compact INR $\Phi(\tilde{x} | \theta)$, which maps 2D or 3D coordinates $\tilde{x}$ to field values $\tilde{v}$.

**Data Structures:**
1.  **Encoder Positions ($\mathcal{E}$):** A set of $N$ scene parameters $\mathcal{E} = \{\theta_j | j = a, b, c, ...\}$ serving as anchor points in the parameter space.
    *   **Generation:** Generated using Bridson’s fast Poisson Disk sampling combined with Gaussian kernel sampling to ensure even distribution and integration of application-specific knowledge [S1, unofficial].
    *   **Counts:** 177 positions for Novel View Synthesis (NVS) in $[0,1]^2$; 24 evenly distributed for Temporal Super-Resolution (TSR); 206 for Dynamic Global Shadows (DGS) using Poisson disk + Gaussian distribution [S1, unofficial].
2.  **Multiresolution Hash Encoders ($\mathcal{E}_j$):** For each $\theta_j \in \mathcal{E}$, a multiresolution hash encoder is constructed.
    *   **Configuration:** $L=8$ encoding levels, $T=2^{15}$ feature vectors per level, feature size $F=4$ [S1, unofficial].
    *   **Initialization:** Hash table entries initialized using uniform distribution $U(-10^{-4}, 10^{-4})$ [S1, unofficial].
3.  **Spatial Data Structure:**
    *   **High Dimensions:** A KD-tree organizes the encoder positions $\mathcal{E}$ [S1, S2, unofficial].
    *   **1D Dimension:** A linear array is used instead of a KD-tree [S1, unofficial].
4.  **Shared Synthesis MLP ($S$):** A Multi-Layer Perceptron shared across all encoders.
    *   **Configuration:** Four hidden layers, width of 64 neurons [S1, unofficial].
    *   **Initialization:** Not explicitly detailed for HyperINR's shared MLP, but CoordNet (teacher) uses SIREN initialization $U(-\sqrt{6/n}, \sqrt{6/n})$ [S1, unofficial].

**Numbered Steps:**

1.  **Pretrain Teacher Model:** Train a large conditional INR teacher model (CoordNet) on the training set $\mathcal{C}$.
    *   For NVS: 300 epochs.
    *   For TSR and DGS: 30,000 epochs [S1, unofficial].
    *   Teacher Optimizer: Adam ($\beta_1=0.9, \beta_2=0.999, \epsilon=10^{-8}$), weight decay $10^{-6}$, learning rate $10^{-5}$ [S1, unofficial].
2.  **Construct Distillation Set:** Sample parameters $\{\theta_k\}$ and compute data $\{\Omega(\theta_k)\}$ using the trained teacher $\Omega$ to create distillation set $\mathcal{D}$ [S1, unofficial].
    *   NVS: Poisson disk + Gaussian sampling.
    *   TSR: Uniform sampling of time axis.
    *   DGS: 400 samples [S1, unofficial].
3.  **Initialize HyperINR:** Initialize hash table entries for the base hash encoder using $U(-10^{-4}, 10^{-4})$ [S1, unofficial].
4.  **Query Encoder Positions:** Given an input parameter $\theta$, traverse the spatial data structure (KD-tree or linear array) to gather the $K$ nearest encoder positions $\theta_j$ [S1, S2, unofficial].
5.  **Interpolate Weights:** Interpolate the weights of the $K$ nearest encoders using Inverse Distance Weighting (IDW) [S1, unofficial].
    *   Weights: $w_j = d(\theta, \theta_j)^{-p}$ with $p=1$ [S1, unofficial].
    *   Formula:
        $$
        E(\theta) = \begin{cases} 
        \frac{\sum_{j=1}^{K} w_j E_j}{\sum_{j=1}^{K} w_j} & \text{if } d(\theta, \theta_j) \neq 0 \text{ for all } j \\
        E_i & \text{if } d(\theta, \theta_i) = 0
        \end{cases}
        $$
    *   For 1D parameter spaces, replace KD-tree with a linear array and perform linear interpolation [S1, unofficial].
6.  **Form INR:** Combine the interpolated encoding weights $E(\theta)$ with the weights of the shared synthesis function $S$ to form the final INR weights [S1, unofficial].
    *   Decomposition: $\Phi(\tilde{x} | \theta) = S \circ E(\theta)(\tilde{x})$ [S1, unofficial].
7.  **Evaluate:** Use a stateless INR evaluator that takes a coordinate matrix $C$ and the predicted network weight matrix $W$ as inputs to generate output data $V_{pred}$ [S1, unofficial].
8.  **Optimize (Training Phase):**
    *   Calculate loss: L2 loss for NVS; L1 loss for TSR and DGS [S1, unofficial].
    *   Backpropagate gradients from the evaluator with respect to $W$ to the hypernetwork to optimize it end-to-end [S1, unofficial].
    *   HyperINR Optimizer: Adam ($\beta_1=0.9, \beta_2=0.999, \epsilon=10^{-10}$), learning rate $10^{-3}$ [S1, unofficial].
    *   Use automatic mixed precision and gradient scaling to avoid data underflow [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np

class HyperINR:
    def __init__(self, num_encoders, encoder_positions, shared_mlp_config, 
                 hash_levels=8, hash_table_size=2**15, feature_size=4):
        """
        Initialize HyperINR.
        
        Args:
            num_encoders (int): Number of anchor encoders (N).
            encoder_positions (np.ndarray): Array of shape (N, dim_theta) containing 
                                            the scene parameters for each encoder.
            shared_mlp_config (dict): Configuration for the shared synthesis MLP.
            hash_levels (int): Number of multiresolution hash encoding levels (L).
            hash_table_size (int): Number of feature vectors per level (T).
            feature_size (int): Size of each feature vector (F).
        """
        self.num_encoders = num_encoders
        self.encoder_positions = encoder_positions
        self.dim_theta = encoder_positions.shape[1]
        
        # 1. Initialize Spatial Data Structure
        # For high dimensions, use KD-tree. For 1D, use linear array.
        if self.dim_theta == 1:
            # Sort positions for linear interpolation
            sorted_indices = np.argsort(encoder_positions.flatten())
            self.sorted_positions = encoder_positions[sorted_indices]
            self.sorted_encoder_weights = None # To be populated during training
            self.use_linear_interpolation = True
        else:
            from scipy.spatial import KDTree
            self.kd_tree = KDTree(encoder_positions)
            self.use_linear_interpolation = False
            
        # 2. Initialize Hash Encoder Weights
        # Initialize hash table entries using U(-10^-4, 10^-4)
        # Shape: (num_encoders, num_levels, hash_table_size, feature_size)
        self.encoder_weights = torch.randn(
            num_encoders, hash_levels, hash_table_size, feature_size,
            dtype=torch.float32
        ) * 2e-4 # Scale for uniform distribution
        
        # 3. Initialize Shared Synthesis MLP (S)
        # Configuration: 4 hidden layers, width 64
        layers = []
        in_dim = feature_size
        for _ in range(4):
            layers.append(nn.Linear(in_dim, 64))
            layers.append(nn.ReLU()) # Assuming ReLU, though SIREN is used in teacher
            in_dim = 64
        layers.append(nn.Linear(64, 1)) # Output dimension depends on task
        self.shared_mlp = nn.Sequential(*layers)
        
        # Initialize shared MLP weights (using standard init or SIREN if applicable)
        # Note: Source specifies SIREN init for CoordNet, not explicitly for HyperINR's shared MLP.
        # Using standard init as placeholder [G].
        for m in self.shared_mlp.modules():
            if isinstance(m, nn.Linear):
                nn.init.kaiming_normal_(m.weight, nonlinearity='relu')
                nn.init.zeros_(m.bias)

    def get_interpolated_encoder_weights(self, theta, k=8):
        """
        Interpolate encoder weights for a given parameter theta.
        
        Args:
            theta (torch.Tensor): Scene parameter vector of shape (dim_theta,).
            k (int): Number of nearest neighbors to consider.
            
        Returns:
            torch.Tensor: Interpolated encoder weights of shape (num_levels, hash_table_size, feature_size).
        """
        theta_np = theta.cpu().numpy()
        
        if self.use_linear_interpolation:
            # 1D Linear Interpolation
            # Find insertion point
            indices = np.searchsorted(self.sorted_positions.flatten(), theta_np[0])
            if indices == 0:
                idx1, idx2 = 0, 1
            elif indices == len(self.sorted_positions):
                idx1, idx2 = len(self.sorted_positions) - 2, len(self.sorted_positions) - 1
            else:
                idx1, idx2 = indices - 1, indices
                
            # Get positions and weights
            pos1 = self.sorted_positions[idx1][0]
            pos2 = self.sorted_positions[idx2][0]
            
            # Avoid division by zero
            if pos2 == pos1:
                return self.sorted_encoder_weights[idx1]
                
            # Linear interpolation factor
            t = (theta_np[0] - pos1) / (pos2 - pos1)
            
            # Interpolate weights
            w1 = self.sorted_encoder_weights[idx1]
            w2 = self.sorted_encoder_weights[idx2]
            interpolated_weights = (1 - t) * w1 + t * w2
            
        else:
            # High-Dimensional IDW Interpolation
            # Query KD-tree for k nearest neighbors
            distances, indices = self.kd_tree.query(theta_np, k=k)
            
            # Calculate weights: w_j = d(theta, theta_j)^-1
            # Handle zero distance
            if np.any(distances == 0):
                # If exact match, return that encoder's weights
                exact_idx = indices[np.argmin(distances)]
                return self.encoder_weights[exact_idx]
            
            weights = 1.0 / distances
            weights = weights / np.sum(weights) # Normalize
            
            # Gather and interpolate weights
            gathered_weights = self.encoder_weights[indices] # Shape: (k, num_levels, T, F)
            # Weighted sum
            interpolated_weights = torch.sum(
                gathered_weights * weights.unsqueeze(-1).unsqueeze(-1).unsqueeze(-1), 
                dim=0
            )
            
        return interpolated_weights

    def forward(self, coordinates, theta):
        """
        Forward pass of the HyperINR.
        
        Args:
            coordinates (torch.Tensor): Input coordinates of shape (N, 3) or (N, 2).
            theta (torch.Tensor): Scene parameters of shape (dim_theta,).
            
        Returns:
            torch.Tensor: Output field values of shape (N, 1).
        """
        # 1. Get interpolated encoder weights
        encoder_weights = self.get_interpolated_encoder_weights(theta)
        
        # 2. Encode coordinates using the interpolated hash encoder
        # Note: This requires implementing the multiresolution hash encoding logic
        # which maps coordinates to indices in the hash table and gathers features.
        # For brevity, we assume a function `hash_encode` exists that uses 
        # `encoder_weights` to produce a feature vector for each coordinate.
        features = hash_encode(coordinates, encoder_weights) # [G] Implementation detail
        
        # 3. Pass features through shared MLP
        output = self.shared_mlp(features)
        
        return output

    def train_step(self, coordinates, theta, ground_truth, optimizer):
        """
        Perform one training step.
        
        Args:
            coordinates (torch.Tensor): Input coordinates.
            theta (torch.Tensor): Scene parameters.
            ground_truth (torch.Tensor): Target values.
            optimizer (torch.optim.Optimizer): Optimizer for HyperINR.
            
        Returns:
            float: Loss value.
        """
        optimizer.zero_grad()
        
        # Forward pass
        pred = self.forward(coordinates, theta)
        
        # Calculate loss (L2 for NVS, L1 for TSR/DGS)
        # Assuming task type is determined by context or passed as argument
        loss = torch.nn.functional.mse_loss(pred, ground_truth) # Placeholder for L1/L2 selection [G]
        
        # Backward pass
        loss.backward()
        optimizer.step()
        
        return loss.item()

def hash_encode(coordinates, encoder_weights):
    """
    Placeholder for multiresolution hash encoding logic.
    In a real implementation, this would map coordinates to hash table indices
    using the multiresolution grid resolutions and gather features from encoder_weights.
    """
    # This function is not fully defined in the sources.
    # It relies on the Tiny-CUDA-NN framework implementation details [S1, unofficial].
    raise NotImplementedError("Hash encoding logic depends on Tiny-CUDA-NN implementation.")
```

**Parameters:**
*   **Inference Time:** Less than 1ms for INR generation [S1, unofficial].
*   **Speedup:** Up to 100× faster than CoordNet [S1, S2, unofficial].
*   **Performance:** 20–30 FPS with volumetric path tracing at 800 × 600 resolution [S2, unofficial].
*   **Optimizer (HyperINR):** Adam ($\beta_1 = 0.9, \beta_2 = 0.999, \epsilon = 10^{-10}$), learning rate $10^{-3}$ [S1, unofficial].
*   **Optimizer (CoordNet Teacher):** Adam ($\beta_1 = 0.9, \beta_2 = 0.999, \epsilon = 10^{-8}$), weight decay $10^{-6}$, learning rate $10^{-5}$ [S1, unofficial].
*   **Hash Encoder:** $L=8$ levels, $T=2^{15}$ vectors/level, $F=4$ feature size [S1, unofficial].
*   **Base Grid Resolution ($R_1$):** 8 for NVS; 4 for TSR and DGS [S1, unofficial].
*   **MLP Unit:** 4 hidden layers, width 64 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Use automatic mixed precision and gradient scaling to avoid data underflow during backpropagation [S1, unofficial].
*   **Boundaries:** If $d(\theta, \theta_j) = 0$ for any $j$, return the exact encoder's weights to avoid division by zero in IDW [S1, unofficial].
*   **Generalizability:** Reducing INR size and using efficient encoding compromises generalizability for unseen parameters, making it unsuitable for tasks like temporal super-resolution if not properly trained [S1, unofficial].
*   **Range Mismatch:** CoordNet expects input/output values in $[-1, 1]$, while hash-encoding-based INRs use $[0, 1]$. The implementation converts $[0, 1]$ to $[-1, 1]$ only when interacting with CoordNet [S1, unofficial].
*   **Quality Limit:** Quality is upper-bounded by the teacher model [S2, unofficial].

## Subtopics

### HyperFLINT
HyperFLINT combines a hypernetwork conditioned on simulation parameters with a flow interpolation network for temporal super-resolution of ensemble data [S2, unofficial]. By generating parameter-specific network weights, it achieves sub-second inference times and 52.70 dB PSNR on cosmological Nyx data at 5× temporal interpolation, substantially outperforming CoordNet (44.02 dB) and STSR-INR (44.21 dB) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** INR generation takes less than 1ms, enabling interactive visualization [S1, unofficial]. Inference bandwidth is 100× that of the teacher model [S2, unofficial].
*   **Memory Complexity:** Depends on the size of the hash encoders and the shared MLP. The hash encoder uses $N \times L \times T \times F$ parameters for the ensemble, plus the shared MLP parameters [S1, unofficial].
*   **Measured Performance:** Achieves 20–30 FPS with volumetric path tracing at 800 × 600 resolution [S2, unofficial].
*   **Strengths:** Interactive speed, high generalizability for unseen parameters via knowledge distillation, consistent parameter meaning across INR instances due to shared MLP [S1, unofficial].
*   **Limitations:** Quality is upper-bounded by the teacher model [S2, unofficial]. Reducing INR size can compromise generalizability [S1, unofficial].
*   **When to Use:** Interactive volumetric path tracing, real-time parameter exploration, novel view synthesis, temporal super-resolution, and dynamic global shadows [S1, unofficial].
*   **When Not to Use:** Tasks requiring higher quality than the teacher model can provide, or where the overhead of maintaining the hypernetwork and encoder ensemble is unjustified for static scenes [S1, S2, unofficial].

## Background
HyperINR was introduced by Qi Wu, David Bauer, Yuyang Chen, and Kwan-Liu Ma from the University of California, Davis. It was published on arXiv on 9 Apr 2023 (arXiv:2304.04188v1) [S1, unofficial]. The method addresses the limitation of standard INRs, which require millions of operations per forward pass, hindering interactive visualization [S1, unofficial]. It builds on [InstantNGP Network Architecture](5049-instantngp-network-architecture.md) by using multiresolution hash encoding [S1, unofficial]. It is used for Novel View Synthesis (NVS), Temporal Super-Resolution of Volume Data (TSR), and Volume Rendering with Dynamic Global Shadows (DGS) [S1, unofficial].

## Sources
[S1] HyperINR: A Fast and Predictive Hypernetwork for Implicit Neural Representations via Knowledge Distillation, https://doi.org/10.48550/arxiv.2304.04188, fetched 2026-10-26 (unofficial)
[S2] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
