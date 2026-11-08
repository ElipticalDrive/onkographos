# Neural Implicit Network Compression

Also called: Coordinate-based MLP Compression, SIREN-based INR Compression, neurcomp

## Overview
Neural Implicit Network Compression (INR) replaces explicit voxel grids with a coordinate-based neural network that maps spatial coordinates directly to field values, serving as simultaneously compressed storage and a queryable data source [S1, unofficial]. Unlike traditional lossy compressors (e.g., SZ, ZFP) that produce bitstreams requiring full decoding before visualization, INRs allow direct evaluation at arbitrary resolutions [S1, unofficial]. The method sets itself apart by achieving high compression ratios on scalar fields (e.g., 545:1) while maintaining high fidelity, though it typically requires significant training time and lacks native interactive rendering capabilities compared to cached or decompressed alternatives [S1, unofficial].

## How it works
The core method, introduced as **neurcomp**, uses a coordinate-based SIREN (Sinusoidal Representation Network) MLP with residual connections [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A 3D scalar field volume $V$ defined on a grid (e.g., $512^3$).
*   **Output:** A compressed neural network model (weights) that approximates $V$.

**Data Structures:**
*   **Network Architecture:** A Multi-Layer Perceptron (MLP) using SIREN layers (sine activation functions) with residual connections.
*   **Weight Storage:** The network weights are quantized to 9 bits per weight using k-means clustering [S1, unofficial].

**Steps:**
1.  **Sampling:** Sample spatial coordinates $(x, y, z)$ from the volume domain.
2.  **Encoding:** Apply positional or hash encoding $\gamma(x)$ to the coordinates [S1, unofficial].
3.  **Forward Pass:** Compute the field value $v$ by passing the encoded coordinates through the SIREN MLP:
    $$ v = \text{MLP}(\gamma(x, y, z)) $$
4.  **Training:** Train the network to minimize the reconstruction error between predicted values and ground truth voxel values. An optional gradient regularization term can be added to improve isosurface fidelity, though this may slightly reduce PSNR [S1, unofficial].
5.  **Quantization:** Apply k-means clustering to the trained network weights to quantize them to 9 bits [S1, unofficial].

**Parameters:**
*   **Weight Quantization Bits:** 9 bits [S1, unofficial].
*   **Hardware:** Training performed on a V100 GPU [S1, unofficial].
*   **Training Time:** Hours per volume [S1, unofficial].

**Pitfalls:**
*   **Rendering Speed:** The method could not render interactively from the volume due to the computational cost of evaluating the network per pixel/voxel [S1, unofficial].
*   **Training Cost:** Requires significant GPU time for training [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import numpy as np
from sklearn.cluster import KMeans

class SIRENLayer(nn.Module):
    """SIREN layer with sine activation."""
    def __init__(self, in_features, out_features, w_init=30, bias_init=30):
        super().__init__()
        self.linear = nn.Linear(in_features, out_features)
        self.w_init = w_init
        self.bias_init = bias_init
        self.sine = nn.Sine()
        self._reset_parameters()

    def _reset_parameters(self):
        # SIREN initialization
        with torch.no_grad():
            self.linear.weight.uniform_(
                -1 / self.linear.in_features,
                1 / self.linear.in_features
            )
            self.linear.bias.uniform_(
                -self.bias_init / self.linear.in_features,
                self.bias_init / self.linear.in_features
            )

    def forward(self, x):
        return self.sine(self.linear(x))

class NeurCompModel(nn.Module):
    """NeurComp: SIREN MLP with residual connections."""
    def __init__(self, in_dim=3, hidden_dim=256, out_dim=1, num_layers=4):
        super().__init__()
        layers = []
        for i in range(num_layers):
            if i == 0:
                layers.append(SIRENLayer(in_dim, hidden_dim))
            else:
                layers.append(SIRENLayer(hidden_dim, hidden_dim))
        layers.append(nn.Linear(hidden_dim, out_dim))
        self.network = nn.Sequential(*layers)
        # Residual connections logic would be implemented here
        # e.g., adding input features to intermediate layers if specified

    def forward(self, x):
        return self.network(x)

def train_neurcomp(volume_data, num_epochs=1000):
    """
    Train NeurComp on a volume.
    volume_data: Tensor of shape (N, 3, 1) containing sampled coordinates and values.
    """
    model = NeurCompModel()
    optimizer = torch.optim.Adam(model.parameters(), lr=1e-4)
    
    for epoch in range(num_epochs):
        optimizer.zero_grad()
        # Forward pass
        # x_coords: (N, 3)
        # y_true: (N, 1)
        y_pred = model(x_coords)
        
        # Loss
        loss = nn.MSELoss()(y_pred, y_true)
        
        # Optional gradient regularization for isosurface fidelity
        # grad_loss = compute_gradient_loss(model, x_coords, y_true)
        # total_loss = loss + lambda_grad * grad_loss
        
        total_loss = loss
        total_loss.backward()
        optimizer.step()

def quantize_weights(model, bits=9):
    """
    Quantize model weights using k-means clustering.
    """
    # Flatten all weights
    all_weights = []
    for param in model.parameters():
        all_weights.append(param.data.view(-1))
    all_weights = torch.cat(all_weights).cpu().numpy()
    
    # Determine number of levels
    num_levels = 2 ** bits
    
    # K-means clustering
    kmeans = KMeans(n_clusters=num_levels, random_state=42)
    kmeans.fit(all_weights.reshape(-1, 1))
    
    # Replace weights with cluster centers
    # Note: In practice, this requires mapping back to the model structure
    # and updating the tensor values.
    cluster_centers = kmeans.cluster_centers_.flatten()
    
    # Placeholder for actual weight replacement logic
    # for param in model.parameters():
    #     flat = param.data.view(-1).cpu().numpy()
    #     labels = kmeans.predict(flat.reshape(-1, 1))
    #     quantized = cluster_centers[labels]
    #     param.data = torch.tensor(quantized).view(param.shape).to(param.device)
    
    return model

# Example usage
# volume = load_volume("rayleigh_taylor.dat")
# coords, values = sample_volume(volume)
# model = train_neurcomp((coords, values))
# compressed_model = quantize_weights(model, bits=9)
```

## Subtopics

### Time-varying INR compression via Laplacian pyramid
A method that decomposes time-varying volumes via a Laplacian pyramid with per-block MLPs to improve decoding speed [S1, unofficial]. It achieves decoding speeds 13–30× faster than neurcomp on datasets up to $1,728^3$ [S1, unofficial].

### MVNet
A method for compressing multivariate data by formulating it as multi-task learning, using a Residual SIREN with a v-dimensional output layer (where v is the number of physical variables) that shares all intermediate parameters across variables [S1, unofficial]. It preserves inter-variable statistical dependencies (Pearson correlation and mutual information) better than traditional compressors [S1, unofficial]. On a 100-variable climate dataset (1568 MB), it achieved 49.68 dB PSNR at 1298:1 compression, outperforming TTHRESH (48.62 dB at 343:1) [S1, unofficial]. It has only been tested on static datasets [S1, unofficial].

### KD-INR
A two-stage pipeline for compressing time-varying data: first, compressing each timestep independently with a small INR with bottleneck layers and entropy-based adaptive sampling to concentrate training on high-information blocks; second, merging all per-timestep teacher models into a single student via offline knowledge distillation (an $O(n)$ procedure) [S1, unofficial]. On time-varying datasets up to 23.44 GB (argon bubble, 150 timesteps), it outperformed CoordNet, NeurComp, and SIREN at compression ratios from 733 to 10,000, with per-timestep compression requiring 1–6 minutes on a single GPU [S1, unofficial].

## Operating characteristics
*   **Compression Ratio:** neurcomp achieves 545:1 compression on scalar fields up to $512^3$ with 47.9 dB PSNR on the Rayleigh-Taylor dataset [S1, unofficial]. MVNet achieves 1298:1 on a 100-variable climate dataset [S1, unofficial]. KD-INR achieves ratios from 733 to 10,000 on time-varying data [S1, unofficial].
*   **Training Time:** neurcomp requires hours per volume on a V100 GPU [S1, unofficial]. KD-INR requires 1–6 minutes per timestep on a single GPU [S1, unofficial].
*   **Rendering Speed:** neurcomp cannot render interactively [S1, unofficial]. Laplacian pyramid variants achieve 13–30× faster decoding than neurcomp [S1, unofficial].
*   **Strengths:** High compression ratios; queryable data source; preserves statistical dependencies (MVNet); fast per-timestep training (KD-INR).
*   **Limitations:** High training time; slow rendering without caching or specialized decoders; limited to static data for some variants (MVNet) [S1, unofficial].

## Background
Neural Implicit Network Compression was introduced as the first INR compression method for scientific volumes [S1, unofficial]. It uses a coordinate-based SIREN MLP with residual connections and 9-bit weight quantization via k-means clustering [S1, unofficial]. The method was evaluated on scalar fields up to $512^3$ and compared against tensor decomposition methods like TTHRESH [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
