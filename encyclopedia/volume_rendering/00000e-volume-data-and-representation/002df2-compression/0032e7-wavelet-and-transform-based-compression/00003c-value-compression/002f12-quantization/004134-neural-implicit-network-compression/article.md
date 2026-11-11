# Neural Implicit Network Compression

Also called: Coordinate-based MLP Compression, SIREN-based INR Compression, neurcomp

## Overview
Neural Implicit Network (INR) Compression maps spatial coordinates directly to field values using a coordinate-based neural network, serving as simultaneously compressed storage and a queryable data source [S1, unofficial]. Unlike traditional lossy compressors (e.g., SZ, ZFP) that produce bitstreams requiring full decoding before visualization, INRs allow direct evaluation at arbitrary resolutions [S1, unofficial]. It sets itself apart from sibling quantization methods by replacing discrete coefficient storage with a continuous parametric function, enabling infinite resolution at the cost of significant training time and lack of interactive rendering capabilities in early implementations [S1, unofficial].

## How it works
**Inputs:** A scalar or multivariate scientific volume $V$ defined on a grid of size $N^3$ (e.g., $512^3$).
**Outputs:** A compressed neural network model (weights and architecture) that approximates $V(\mathbf{x})$ for $\mathbf{x} \in [0,1]^3$.

**Data Structures:**
1.  **Network Weights:** A set of matrices and bias vectors defining the MLP. In the baseline `neurcomp` method, these are quantized to 9-bit integers using k-means clustering [S1, unofficial].
2.  **Training Samples:** A set of coordinate-value pairs $\{(\mathbf{x}_i, v_i)\}$ sampled from the volume.

**Algorithm (neurcomp baseline):**
1.  **Architecture Selection:** Initialize a SIREN (Sinusoidal Representation Network) MLP with residual connections. SIRENs use sine activation functions $\sin(\omega_0 \mathbf{W}\mathbf{x} + \mathbf{b})$ to better represent high-frequency details common in scientific data compared to standard ReLU networks [S1, unofficial].
2.  **Sampling:** Sample spatial coordinates $\mathbf{x}$ from the volume domain. For uniform sampling, points are drawn from the grid. For adaptive sampling, entropy-based methods may concentrate samples in high-information regions [S1, unofficial].
3.  **Training:** Minimize the mean squared error (MSE) between the network output and the ground truth values:
    $$ \mathcal{L} = \frac{1}{M} \sum_{i=1}^{M} \| \text{MLP}(\mathbf{x}_i) - v_i \|^2 $$
    Optionally, add a gradient regularization term $\mathcal{L}_{grad}$ to improve isosurface fidelity:
    $$ \mathcal{L}_{total} = \mathcal{L} + \lambda \mathcal{L}_{grad} $$
    where $\lambda$ is a hyperparameter. This improves geometric accuracy at the cost of slight PSNR reduction [S1, unofficial].
4.  **Quantization:** After training, apply k-means clustering to the network weights to quantize them to 9-bit integers [S1, unofficial].
5.  **Storage:** Store the quantized weights and the network architecture definition.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def train_neurcomp(volume_data, grid_size, num_epochs=1000, learning_rate=1e-4):
    """
    Trains a SIREN-based INR for volume compression.
    
    Args:
        volume_data: 3D numpy array of shape (N, N, N) containing field values.
        grid_size: Integer N, the resolution of the volume.
        num_epochs: Number of training iterations.
        learning_rate: Learning rate for optimizer.
        
    Returns:
        quantized_weights: Quantized network weights.
    """
    # 1. Initialize SIREN MLP with residual connections
    # Note: Specific layer dimensions are not given in sources; 
    # assume a standard deep MLP with sine activations.
    model = SIREN_MLP(residual=True)
    optimizer = torch.optim.Adam(model.parameters(), lr=learning_rate)
    
    # 2. Prepare training data
    # Generate random coordinates in [0, 1]^3
    coords = np.random.rand(10000, 3)
    # Map to grid indices and sample values
    # Assuming volume_data is normalized to [0,1] spatially
    grid_indices = (coords * (grid_size - 1)).astype(int)
    # Clamp indices to valid range
    grid_indices = np.clip(grid_indices, 0, grid_size - 1)
    values = volume_data[grid_indices[:, 0], grid_indices[:, 1], grid_indices[:, 2]]
    
    # Convert to tensors
    X = torch.tensor(coords, dtype=torch.float32)
    Y = torch.tensor(values, dtype=torch.float32)
    
    # 3. Training Loop
    for epoch in range(num_epochs):
        optimizer.zero_grad()
        
        # Forward pass
        predictions = model(X)
        
        # Compute MSE loss
        loss = torch.mean((predictions - Y) ** 2)
        
        # Optional: Gradient regularization for isosurface fidelity
        # This requires computing gradients of the network output w.r.t input
        # Not explicitly detailed in sources, so omitted for core definition [G]
        
        # Backward pass
        loss.backward()
        optimizer.step()
        
    # 4. Quantization
    # Apply k-means clustering to weights to 9-bit precision
    quantized_weights = kmeans_quantize(model.get_weights(), bits=9)
    
    return quantized_weights

def kmeans_quantize(weights, bits=9):
    """
    Quantizes weights using k-means clustering.
    """
    # Flatten weights
    flat_weights = weights.flatten()
    # Determine number of clusters (2^bits)
    k = 2 ** bits
    # Perform k-means clustering
    # Note: Standard k-means implementation required [G]
    centroids, labels = perform_kmeans(flat_weights, k)
    # Map weights to nearest centroid
    quantized_flat = centroids[labels]
    # Reshape back to original weight shapes
    quantized_weights = quantized_flat.reshape(weights.shape)
    return quantized_weights
```

**Parameters:**
*   **Weight Quantization Bits:** 9-bit (baseline neurcomp) [S1, unofficial].
*   **Hardware:** V100 GPU [S1, unofficial].
*   **Training Time:** Hours per volume [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Interactive Rendering:** The method could not render interactively from the volume due to the computational cost of evaluating the network for every ray [S1, unofficial].
*   **Precision:** Quantization to 9-bit may introduce artifacts if the network weights have high variance; k-means clustering is used to mitigate this [S1, unofficial].

## Subtopics

### Time-varying INR compression via Laplacian pyramid
A method that decomposes time-varying volumes via a Laplacian pyramid with per-block MLPs to improve decoding speed [S1, unofficial]. It is chosen when fast decoding of large time-series data is required. It achieves decoding speeds 13–30× faster than neurcomp on datasets up to $1,728^3$ [S1, unofficial].

### MVNet
A method for compressing multivariate data by formulating it as multi-task learning, using a Residual SIREN with a $v$-dimensional output layer (where $v$ is the number of physical variables) and sharing all intermediate parameters across variables [S1, unofficial]. It is chosen for multivariate scientific data to preserve inter-variable statistical dependencies (Pearson correlation and mutual information) better than traditional compressors [S1, unofficial]. On a 100-variable climate dataset (1568 MB), it achieved 49.68 dB PSNR at 1298:1 compression, outperforming TTHRESH (48.62 dB at 343:1) [S1, unofficial]. Pitfall: Only tested on static datasets [S1, unofficial].

### KD-INR
A two-stage pipeline for compressing time-varying data: first, compressing each timestep independently with a small INR with bottleneck layers and entropy-based adaptive sampling; second, merging all per-timestep teacher models into a single student via offline knowledge distillation (an $O(n)$ procedure) [S1, unofficial]. It is chosen for large time-varying datasets where training from scratch is too slow. On datasets up to 23.44 GB (150 timesteps), it outperformed CoordNet, NeurComp, and SIREN at compression ratios from 733 to 10,000, with training time of 1–6 minutes per timestep on a single GPU [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training time is significant, requiring hours per volume on a V100 GPU for the baseline neurcomp method [S1, unofficial]. KD-INR reduces per-timestep training to 1–6 minutes [S1, unofficial].
*   **Memory Complexity:** Storage is determined by the size of the quantized network weights. For neurcomp, 9-bit quantization allows high compression ratios (e.g., 545:1 on $512^3$ volumes) [S1, unofficial].
*   **Measured Performance:** On scalar fields up to $512^3$, neurcomp achieved 47.9 dB PSNR at 545:1 compression on the Rayleigh-Taylor dataset, outperforming tensor decomposition (TTHRESH) at high compression ratios [S1, unofficial]. MVNet achieved 49.68 dB PSNR at 1298:1 compression on a 100-variable climate dataset [S1, unofficial].
*   **Strengths:** Simultaneously compressed storage and queryable data source; enables infinite resolution; preserves inter-variable dependencies in multivariate cases [S1, unofficial].
*   **Limitations:** Cannot render interactively from the volume in baseline implementations; training time is high; quality is upper-bounded by the teacher model in distillation methods [S1, unofficial].
*   **When to Use:** When storage is constrained and interactive rendering is not required during the compression phase, or when post-compression evaluation at arbitrary resolutions is needed.
*   **When Not to Use:** When interactive rendering is required immediately after compression, or when training resources are limited.

## Background
Neural Implicit Network Compression originated as a method for scientific volumes, with `neurcomp` being the first INR compression method for this domain, using a coordinate-based SIREN MLP with residual connections and 9-bit weight quantization via k-means clustering [S1, unofficial]. It was developed to address the strain on storage systems and I/O pipelines caused by full-resolution scientific volumes [S1, unofficial]. Traditional lossy compressors like SZ and ZFP produce bitstreams that must be fully decoded before visualization, whereas INRs offer a compact network that maps spatial coordinates to field values [S1, unofficial]. The field has since expanded to include multivariate data (MVNet) and time-varying data (KD-INR, Laplacian pyramid variants) [S1, unofficial].

## Sources
[S1] Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
