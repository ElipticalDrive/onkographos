# Neural Implicit Network Compression

Also called: Coordinate-based MLP Compression, SIREN-based INR Compression, neurcomp

## Overview
Neural Implicit Network (INR) Compression maps spatial coordinates directly to field values using a coordinate-based neural network, serving as simultaneously compressed storage and a queryable data source [S1, unofficial]. Unlike traditional lossy compressors (e.g., SZ, ZFP) that produce bitstreams requiring full decoding before visualization, INRs allow direct evaluation of the field at arbitrary resolutions [S1, unofficial]. It sets itself apart from siblings like [Scalar Quantization](5726-scalar-quantization.md) by learning a continuous parametric representation rather than discretizing static values, and from [Differentiable Quantization for Neural Fields](10195-differentiable-quantization-for-neural-fields.md) by focusing on weight quantization and architectural efficiency for scientific volumes rather than differentiable quantization of activations or weights during training for general neural fields [S1, unofficial].

## How it works
**Inputs:** A scalar or multivariate volumetric dataset $V$ defined on a 3D grid (e.g., $N \times N \times N$).
**Outputs:** A neural network model (weights and architecture) that approximates the field $f(x, y, z) \approx V(x, y, z)$.

**Data Structures:**
The core data structure is a Multi-Layer Perceptron (MLP). For the baseline method (neurcomp), the network is a SIREN (Sinusoidal Representation Network) with residual connections.
*   **Weights:** Quantized to 9 bits using k-means clustering [S1, unofficial].
*   **Architecture:** Coordinate-based MLP. The input is a 3D coordinate vector $\mathbf{x} = [x, y, z]$. The output is the field value $v$.

**Steps:**
1.  **Sampling:** Sample spatial coordinates $\{(x_j, y_j, z_j)\}$ from the volume. For high-frequency features, entropy-based adaptive sampling can be used to concentrate training on high-information blocks [S1, unofficial].
2.  **Encoding:** Apply positional or hash encoding $\gamma(\mathbf{x})$ to the input coordinates to map them to a higher-dimensional space suitable for MLP processing [S1, unofficial].
3.  **Network Forward Pass:** Pass the encoded coordinates through the SIREN MLP. The network uses sinusoidal activation functions (e.g., $\sin(\mathbf{W}\mathbf{x} + \mathbf{b})$) to capture high-frequency details [S1, unofficial].
4.  **Training:** Minimize the reconstruction loss (e.g., Mean Squared Error) between the network output and the ground truth values at the sampled coordinates.
    *   *Optional:* Add a gradient regularization term to improve isosurface fidelity. This improves geometric accuracy at the cost of a slight reduction in PSNR [S1, unofficial].
5.  **Quantization:** Apply k-means clustering to the network weights to quantize them to 9 bits [S1, unofficial].
6.  **Storage:** Store the quantized weights and network architecture. The model size constitutes the compressed file.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from sklearn.cluster import KMeans

def train_neurcomp(volume_data, grid_shape, num_samples=1000000, epochs=1000, lr=1e-4):
    """
    Trains a SIREN-based INR for volume compression.
    volume_data: 3D numpy array of shape (N, N, N)
    grid_shape: tuple (N, N, N)
    """
    # 1. Initialize SIREN MLP
    # Input dim: 3 (x, y, z)
    # Hidden dims: e.g., [256, 256, 256, 256]
    # Output dim: 1 (scalar value)
    # Activation: sin()
    model = SIRENMLP(input_dim=3, hidden_dims=[256, 256, 256, 256], output_dim=1)
    
    # 2. Sample coordinates
    # Generate random indices within the grid bounds
    indices = np.random.randint(0, grid_shape[0], size=(num_samples, 3))
    # Map indices to normalized coordinates [0, 1] or physical coordinates
    coords = indices / grid_shape[0] # Assuming uniform grid
    
    # Get ground truth values
    values = volume_data[indices[:, 0], indices[:, 1], indices[:, 2]]
    
    # 3. Training Loop
    optimizer = torch.optim.Adam(model.parameters(), lr=lr)
    for epoch in range(epochs):
        optimizer.zero_grad()
        
        # Forward pass
        # Note: SIREN uses sin activation, no ReLU
        predictions = model(coords)
        
        # Loss calculation
        loss = torch.mean((predictions - values) ** 2)
        
        # Optional gradient regularization for isosurface fidelity
        # This requires computing gradients of the output w.r.t input
        # grad_loss = compute_gradient_regularization(model, coords, predictions)
        # total_loss = loss + lambda_grad * grad_loss
        
        loss.backward()
        optimizer.step()
        
    # 4. Weight Quantization
    # Flatten all weight tensors
    all_weights = [p.data.view(-1) for p in model.parameters()]
    all_weights_flat = torch.cat(all_weights)
    
    # K-means clustering for 9-bit quantization (512 clusters)
    kmeans = KMeans(n_clusters=512, random_state=42)
    kmeans.fit(all_weights_flat.reshape(-1, 1))
    
    # Replace weights with cluster centers
    quantized_centers = kmeans.cluster_centers_
    labels = kmeans.predict(all_weights_flat.reshape(-1, 1))
    
    # Map labels back to centers
    quantized_weights = quantized_centers[labels].reshape(all_weights_flat.shape)
    
    # Update model weights (implementation detail: assign back to model parameters)
    # This step is [G] as the exact tensor assignment syntax depends on the framework
    assign_quantized_weights(model, quantized_weights, labels, kmeans.cluster_centers_)
    
    return model

def assign_quantized_weights(model, flat_quantized, labels, centers):
    """
    [G] Helper to assign quantized weights back to the model.
    Iterates through model parameters, reshapes flat_quantized chunks,
    and assigns them.
    """
    idx = 0
    for param in model.parameters():
        num_elements = param.numel()
        chunk = flat_quantized[idx : idx + num_elements]
        # Reshape to match parameter shape
        param.data = chunk.view(param.shape)
        idx += num_elements

class SIRENMLP(torch.nn.Module):
    def __init__(self, input_dim, hidden_dims, output_dim):
        super().__init__()
        layers = []
        prev_dim = input_dim
        for h_dim in hidden_dims:
            layers.append(torch.nn.Linear(prev_dim, h_dim))
            layers.append(torch.nn.Sine()) # Sinusoidal activation
            prev_dim = h_dim
        layers.append(torch.nn.Linear(prev_dim, output_dim))
        self.network = torch.nn.Sequential(*layers)

    def forward(self, x):
        return self.network(x)
```

**Parameters:**
*   **Weight Quantization Bits:** 9 bits (512 levels) [S1, unofficial].
*   **Training Time:** Hours per volume on a V100 GPU [S1, unofficial].
*   **Grid Size:** Tested on scalar fields up to $512^3$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Interactive Rendering:** The method could not render interactively from the volume due to the computational cost of evaluating the network for every ray [S1, unofficial].
*   **Precision:** Quantization to 9 bits may introduce artifacts if the clustering is not optimal; however, it was shown to be effective for scientific volumes [S1, unofficial].

## Subtopics

### Time-varying INR compression via Laplacian pyramid
A method that decomposes time-varying volumes via a Laplacian pyramid with per-block MLPs to improve decoding speed [S1, unofficial]. It is chosen when fast decoding of large time-series data is required. It achieves decoding speeds 13–30× faster than neurcomp on datasets up to $1,728^3$ [S1, unofficial].

### MVNet
A method for compressing multivariate data by formulating it as multi-task learning, using a Residual SIREN with a v-dimensional output layer (v = number of physical variables) sharing intermediate parameters across variables [S1, unofficial]. It is chosen when compressing correlated multivariate fields (e.g., climate data) to preserve inter-variable statistical dependencies (Pearson correlation and mutual information) better than traditional compressors [S1, unofficial]. On a 100-variable climate dataset (1568 MB), it achieved 49.68 dB PSNR at 1298:1 compression, outperforming TTHRESH (48.62 dB at 343:1) [S1, unofficial]. It has only been tested on static datasets [S1, unofficial].

### KD-INR
A two-stage pipeline for compressing time-varying data: first compressing each timestep independently with a small INR with bottleneck layers, then merging all per-timestep teacher models into a single student via offline knowledge distillation (an $O(n)$ procedure) [S1, unofficial]. It uses entropy-based adaptive sampling to concentrate training on high-information blocks [S1, unofficial]. It is chosen for large time-varying datasets where training each timestep from scratch is too slow. On datasets up to 23.44 GB (argon bubble, 150 timesteps), it outperformed CoordNet, NeurComp, and SIREN at compression ratios from 733 to 10,000, with per-timestep training requiring 1–6 minutes on a single GPU [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time is significant, requiring hours per volume on a V100 GPU for the baseline neurcomp method [S1, unofficial]. KD-INR reduces per-timestep training to 1–6 minutes [S1, unofficial].
*   **Memory Complexity:** The storage size is determined by the number of network weights. For neurcomp, 9-bit quantization reduces the weight size significantly [S1, unofficial].
*   **Performance:** On scalar fields up to $512^3$, neurcomp outperformed tensor decomposition (TTHRESH) at high compression ratios, achieving 47.9 dB PSNR at 545:1 compression on the Rayleigh-Taylor dataset [S1, unofficial].
*   **Strengths:** Simultaneously compressed storage and queryable data source; high compression ratios; ability to preserve high-frequency details via SIREN; multivariate dependency preservation (MVNet); fast decoding for time-varying data (Laplacian pyramid) [S1, unofficial].
*   **Limitations:** Cannot render interactively from the volume without additional caching or acceleration structures (e.g., GPU cache for INR-compressed volumes achieves 5× rendering speedup) [S1, unofficial]. Training time can be long for large volumes [S1, unofficial].

## Background
Neural Implicit Network Compression was introduced as a method for compressing scientific volumes, addressing the strain on storage systems and I/O pipelines caused by full-resolution data [S1, unofficial]. The first INR compression method for scientific volumes, neurcomp, used a coordinate-based SIREN MLP with residual connections and 9-bit weight quantization via k-means clustering [S1, unofficial]. It was developed to provide an alternative to traditional lossy compressors like SZ and ZFP, which produce bitstreams that must be fully decoded before visualization [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
