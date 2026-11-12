# Neural Implicit Network Compression

Also called: Coordinate-based MLP Compression, SIREN-based INR Compression, neurcomp

## Overview
Neural Implicit Network Compression (INR Compression) replaces traditional grid-based storage of scientific volumes with a compact coordinate-based neural network that maps spatial coordinates directly to field values [S1, unofficial]. This approach serves as simultaneously compressed storage and a queryable data source, offering an alternative to lossy compressors like SZ and ZFP that require full decoding before visualization [S1, unofficial]. It sets itself apart from siblings like [Quantization](5935-quantization.md) by encoding the data structure itself into network weights rather than discretizing values, and from [Differentiable Quantization for Neural Fields](10195-differentiable-quantization-for-neural-fields.md) by focusing on weight quantization and architectural efficiency for scientific data rather than differentiable quantization layers for training stability [S1, unofficial].

## How it works
The core method, exemplified by **neurcomp**, involves training a coordinate-based Multi-Layer Perceptron (MLP) to approximate a scalar field.

**Inputs and Outputs:**
*   **Input:** Spatial coordinates $\mathbf{x} = (x, y, z)$ within the volume domain.
*   **Output:** Field value $v$ at $\mathbf{x}$.
*   **Training Data:** A set of sampled coordinates $\{(x_j, y_j, z_j)\}$ and their corresponding ground truth values $v_j$ from the original volume grid.

**Data Structures:**
*   **Network Architecture:** A SIREN (Sinusoidal Representation Network) MLP with residual connections.
*   **Weights:** The network parameters (weights and biases) are the compressed representation.
*   **Quantization:** Weights are quantized to 9 bits using k-means clustering [S1, unofficial].

**Steps:**
1.  **Sampling:** Sample coordinates from the original volume. For high-frequency details, optional entropy-based adaptive sampling can concentrate training on high-information blocks [S1, unofficial].
2.  **Training:** Train the SIREN MLP to minimize the Mean Squared Error (MSE) between the network output and the sampled ground truth values.
3.  **Regularization (Optional):** Apply a gradient regularization term to improve isosurface fidelity. This may result in a slight reduction in PSNR [S1, unofficial].
4.  **Quantization:** Apply k-means clustering to the trained network weights to reduce precision to 9 bits [S1, unofficial].
5.  **Storage:** Store the quantized weights. The network serves as the compressed file.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
from sklearn.cluster import KMeans

def train_neurcomp(volume_grid, num_samples=100000, num_clusters=512):
    """
    Trains a SIREN-based INR and quantizes weights.
    
    Args:
        volume_grid: 3D numpy array of shape (Nx, Ny, Nz) containing field values.
        num_samples: Number of coordinate-value pairs to sample for training.
        num_clusters: Number of clusters for k-means weight quantization.
        
    Returns:
        quantized_weights: List of numpy arrays representing quantized layer weights.
    """
    # 1. Sample coordinates and values
    # Flatten grid to get all voxel centers
    coords_3d = np.argwhere(np.ones(volume_grid.shape))
    values = volume_grid[coords_3d[:, 0], coords_3d[:, 1], coords_3d[:, 2]]
    
    # Randomly sample indices
    sample_indices = np.random.choice(len(coords_3d), num_samples, replace=False)
    sampled_coords = coords_3d[sample_indices]
    sampled_values = values[sample_indices]
    
    # Normalize coordinates to [0, 1] for better training stability
    max_coord = np.max(sampled_coords, axis=0)
    sampled_coords_normalized = sampled_coords / max_coord
    
    # 2. Define and Train SIREN MLP
    # Note: Actual SIREN implementation requires specific initialization (e.g., SIREN init)
    # and sinusoidal activation functions. This is a conceptual placeholder.
    model = SIREN_MLP(input_dim=3, hidden_dim=256, output_dim=1, num_layers=4)
    
    # Training loop (conceptual)
    optimizer = torch.optim.Adam(model.parameters(), lr=1e-4)
    for epoch in range(1000): # Number of epochs not specified in sources
        optimizer.zero_grad()
        predictions = model(sampled_coords_normalized)
        loss = torch.mean((predictions - sampled_values) ** 2)
        
        # Optional gradient regularization for isosurface fidelity [G]
        # gradients = torch.autograd.grad(predictions.sum(), sampled_coords_normalized, create_graph=True)[0]
        # grad_loss = torch.mean(gradients ** 2)
        # loss += lambda_grad * grad_loss
        
        loss.backward()
        optimizer.step()
        
    # 3. Quantize Weights
    quantized_weights = []
    for param in model.parameters():
        # Flatten weights for clustering
        w_flat = param.detach().cpu().numpy().flatten()
        
        # K-means clustering
        kmeans = KMeans(n_clusters=num_clusters, random_state=42)
        labels = kmeans.fit_predict(w_flat.reshape(-1, 1))
        
        # Replace weights with cluster centers
        cluster_centers = kmeans.cluster_centers_.flatten()
        quantized_w_flat = cluster_centers[labels]
        
        # Reshape back to original shape
        quantized_w = quantized_w_flat.reshape(param.shape)
        quantized_weights.append(quantized_w)
        
    return quantized_weights

def query_neurcomp(quantized_weights, coords_normalized):
    """
    Queries the compressed INR for field values.
    
    Args:
        quantized_weights: List of numpy arrays from training.
        coords_normalized: Numpy array of shape (N, 3) with normalized coordinates.
        
    Returns:
        values: Numpy array of shape (N,) with predicted field values.
    """
    # Reconstruct model or forward pass through quantized weights
    # This step depends on the specific SIREN implementation
    model = SIREN_MLP_from_weights(quantized_weights)
    values = model(coords_normalized).detach().cpu().numpy()
    return values
```

**Parameters:**
*   **Bit Depth:** 9 bits per weight [S1, unofficial].
*   **Training Time:** Hours per volume on a V100 GPU [S1, unofficial].
*   **Cluster Count:** Not explicitly given for neurcomp, but 512 is used in the pseudocode example for k-means [G].

**Edge Cases and Pitfalls:**
*   **Interactive Rendering:** The method could not render interactively from the volume due to the computational cost of querying the network [S1, unofficial].
*   **Precision:** Quantization introduces error; 9-bit quantization is a balance between compression and fidelity [S1, unofficial].

## Subtopics

### Time-varying INR compression via Laplacian pyramid
A method that decomposes time-varying volumes via a Laplacian pyramid with per-block MLPs to improve decoding speed [S1, unofficial]. It is chosen when fast decoding of large time-varying datasets is required. It achieves decoding speeds 13–30× faster than neurcomp on datasets up to $1,728^3$ [S1, unofficial].

### MVNet
A method for compressing multivariate data by formulating it as multi-task learning, using a Residual SIREN with a v-dimensional output layer (where v is the number of physical variables) and sharing all intermediate parameters across variables [S1, unofficial]. It is chosen for multivariate scientific data to preserve inter-variable statistical dependencies (Pearson correlation and mutual information) better than traditional compressors [S1, unofficial]. It has only been tested on static datasets [S1, unofficial]. On a 100-variable climate dataset (1568 MB), it achieved 49.68 dB PSNR at 1298:1 compression, outperforming TTHRESH (48.62 dB at 343:1) [S1, unofficial].

### KD-INR
A two-stage pipeline for compressing time-varying data: first compressing each timestep independently with a small INR with bottleneck layers, then merging all per-timestep teacher models into a single student via offline knowledge distillation (an $O(n)$ procedure) [S1, unofficial]. It is chosen for large time-varying datasets where training each timestep from scratch is too slow. It uses entropy-based adaptive sampling to concentrate training on high-information blocks [S1, unofficial]. On time-varying datasets up to 23.44 GB (argon bubble, 150 timesteps), it outperformed CoordNet, NeurComp, and SIREN at compression ratios from 733 to 10,000, with per-timestep compression requiring 1–6 minutes on a single GPU [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time is significant, requiring hours per volume on a V100 GPU for neurcomp [S1, unofficial]. KD-INR reduces per-timestep training to 1–6 minutes [S1, unofficial].
*   **Memory Complexity:** The compressed size is determined by the number of network parameters and their bit depth. neurcomp achieves 545:1 compression on scalar fields up to $512^3$ [S1, unofficial].
*   **Measured Performance:** neurcomp achieved 47.9 dB PSNR at 545:1 compression on the Rayleigh-Taylor dataset [S1, unofficial]. MVNet achieved 49.68 dB PSNR at 1298:1 compression on a 100-variable climate dataset [S1, unofficial].
*   **Strengths:** Simultaneously compressed storage and queryable data source; preserves inter-variable dependencies (MVNet); fast decoding for time-varying data (Laplacian pyramid, KD-INR).
*   **Limitations:** Cannot render interactively from the volume (neurcomp); training time can be high; limited to static datasets for some variants (MVNet) [S1, unofficial].
*   **When to Use:** When interactive rendering is not required, and storage/query efficiency is prioritized over decoding speed. Suitable for scientific volumes where traditional compressors produce bitstreams that must be fully decoded.
*   **When Not to Use:** When interactive rendering is required; when training time is a critical constraint.

## Background
Neural Implicit Network Compression emerged as an alternative to traditional lossy compressors like SZ and ZFP for scientific volumes [S1, unofficial]. The first INR compression method for scientific volumes, neurcomp, used a coordinate-based SIREN MLP with residual connections and 9-bit weight quantization via k-means clustering [S1, unofficial]. It was developed to address the strain on storage systems and I/O pipelines caused by full-resolution scientific volumes [S1, unofficial]. Subsequent variants like MVNet, KD-INR, and Laplacian pyramid methods extended the approach to multivariate and time-varying data [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
