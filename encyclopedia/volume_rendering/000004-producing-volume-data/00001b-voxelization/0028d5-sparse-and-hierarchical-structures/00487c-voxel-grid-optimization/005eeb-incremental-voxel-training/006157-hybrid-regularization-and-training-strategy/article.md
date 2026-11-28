# Hybrid Regularization and Training Strategy

Also called: Coarse-to-fine Hybrid Training, Occupancy Mask Extraction and Implicit-Explicit Optimization

## Overview
Hybrid Regularization and Training Strategy is a two-stage optimization method for voxel-based neural radiance fields that mitigates overfitting in explicit feature grids by leveraging an implicit neural network for initialization [S1, S2]. It solves the problem of deterministic integration being prone to overfitting at specific voxels when features are directly optimized, which leads to degenerate solutions like floaters or holes in sparse input scenarios [S1, S2]. It sets itself apart from pure explicit regularization by using an intermediate implicit phase to provide a powerful initialization, and from pure implicit methods by discarding the network to allow direct grid optimization for higher final quality [S1, S2].

## How it works
The method consists of three sequential phases: Coarse Geometry Search, Implicit Optimization, and Explicit Optimization.

### Inputs and Outputs
**Inputs:**
*   Training images and camera poses [S1, S2].
*   Target voxel grid resolution $R$ (e.g., $128^3$ or $160^3$) [S1].
*   Sparsity regularization weight $\lambda_s = 1e-5$ [S1, S2].
*   Alpha threshold for pruning $\tau_\alpha = 0.01$ [S1, S2].

**Outputs:**
*   Optimized feature grid values for occupied voxels [S1, S2].
*   Binary occupancy mask [S1, S2].
*   Weights for density and color decoders [S1, S2].

### Data Structures
1.  **Implicit MLP:**
    *   Architecture: 1 input layer, 8 hidden layers (512 neurons each), 1 output layer (32 neurons) [S1, S2].
    *   Activations: ReLU for all layers except the last one (linear) [S1, S2].
    *   Input: Vertex coordinates [S1, S2].
    *   Modification: Positional encoding of vertex coordinates is concatenated to the output of the 4th hidden layer [S1, S2].
2.  **Density Decoder:**
    *   Architecture: 1 input layer, 1 hidden layer (32 neurons), 1 output layer (33 neurons) [S1].
    *   Activations: ReLU for hidden layer, linear for output [S1].
3.  **Color Decoder:**
    *   Architecture: 1 input layer (32 neurons), 1 output layer (3 neurons) [S1].
    *   Activations: ReLU for input layer, sigmoid for output [S1].
4.  **Voxel Grid:**
    *   Stores feature vectors (dimension 32) for occupied voxels [S1].
    *   Binary occupancy mask indicating occupied vs. empty space [S1, S2].
    *   Parameters stored in half-precision (float16) to optimize model size [S1, S2].

### Numbered Steps

**Phase 1: Coarse Geometry Search**
1.  Initialize a low-resolution voxel grid with resolution $R/4$ (e.g., $32^3$ if target is $128^3$) [S1, S2].
2.  Initialize features randomly [S1].
3.  Train the coarse model for 5 epochs using a single density decoder [S1, S2].
4.  Extract an unsupervised occupancy mask from the density outputs of the coarse model [S1, S2].
5.  Discard the weights and features of the coarse model [S1, S2].
6.  Prune voxels based on the occupancy mask using $\tau_\alpha = 0.01$ [S1, S2].

**Phase 2: Implicit Optimization**
1.  Initialize the implicit MLP with random weights [S1, S2].
2.  Train the implicit MLP for 1000 epochs [S1, S2].
3.  Use the Adam optimizer with learning rate $5e-4$ for the fine stage [S1, S2].
4.  Sample rays with batch size 16384 (NeRF-synthetic) or 8192 (Tanks and Temples) [S1, S2].
5.  Compute photometric loss and sparsity regularization loss [S1, S2].

**Phase 3: Explicit Optimization**
1.  Initialize the explicit feature grid using values from the trained implicit MLP [S1, S2].
2.  Discard the implicit MLP [S1, S2].
3.  Train the explicit feature grid for 1000 epochs [S1, S2].
4.  Use the Adam optimizer with learning rate $5e-4$ [S1, S2].
5.  Apply sparsity regularization loss [S1, S2].
6.  Prune voxels again using $\tau_\alpha = 0.01$ after the fine stage [S1, S2].

### Formulas
**Sparsity Regularization Loss:**
$$ L_{sparsity} = \lambda_s \sum_i \frac{\sigma_i^2}{\log(1 + \frac{\sigma_i^2}{0.5})} $$
Where:
*   $\sigma_i$ is the $i$-th accumulated density [S1, S2].
*   $\lambda_s$ is the regularization weight, set to $1e-5$ [S1, S2].
*   This loss favors sparsity and prevents the model from predicting background color in empty space [S1, S2].

### Parameters
*   **Implicit MLP Layers:** 8 hidden layers, 512 neurons each [S1, S2].
*   **Feature Dimension:** 32 [S1, S2].
*   **Training Epochs:** 1000 for implicit stage, 1000 for explicit stage [S1, S2].
*   **Coarse Epochs:** 5 [S1, S2].
*   **Optimizer:** Adam [S1, S2].
*   **Learning Rate:** $1e-3$ for coarse geometry searching, $5e-4$ for fine stage (implicit and explicit) [S1, S2].
*   **Batch Size:** 16384 for NeRF-synthetic, 8192 for Tanks and Temples [S1, S2].
*   **Sparsity Weight ($\lambda_s$):** $1e-5$ [S1, S2].
*   **Pruning Threshold ($\tau_\alpha$):** $0.01$ [S1, S2].
*   **Rendering Threshold ($\tau_t$):** $0.01$ [S1, S2].

### Edge Cases and Pitfalls
*   **Overfitting:** Deterministic integration is prone to overfitting at certain voxels when features are explicitly trained directly [S1, S2]. The hybrid strategy solves this by using the implicit MLP to provide a powerful initialization that avoids overfitting [S1, S2].
*   **Empty Space:** Without the occupancy mask, the model may optimize features in empty space, leading to artifacts. The unsupervised mask extraction prevents this [S1, S2].
*   **Precision:** Parameters are stored in half-precision (float16) to optimize model size, which may introduce precision issues if not handled carefully during optimization [S1, S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def positional_encoding(x, num_freq_bands=10):
    """Standard positional encoding for coordinates [G]."""
    scales = [2**i for i in range(num_freq_bands)]
    xb = np.reshape((x * scales), (-1, int(num_freq_bands * x.shape[1])))
    freq_bands = np.sin(xb)
    return np.concatenate([x, freq_bands], axis=1)

def mlp_forward(params, x):
    """Forward pass for the implicit MLP [S1, S2]."""
    h = x
    for i in range(8):
        h = params['W'][i] @ h + params['b'][i]
        if i < 7:  # ReLU for hidden layers
            h = np.maximum(0, h)
        if i == 3:  # Concatenate positional encoding to 4th hidden layer output
            pe = positional_encoding(x)
            h = np.concatenate([h, pe], axis=1)
    # Output layer: linear
    h = params['W_out'] @ h + params['b_out']
    return h

def density_decoder_forward(params, features):
    """Forward pass for density decoder [S1]."""
    h = params['W_d1'] @ features + params['b_d1']
    h = np.maximum(0, h)  # ReLU
    density = params['W_d2'] @ h + params['b_d2']  # Linear output
    return density

def color_decoder_forward(params, features):
    """Forward pass for color decoder [S1]."""
    h = params['W_c1'] @ features + params['b_c1']
    h = np.maximum(0, h)  # ReLU
    color = 1.0 / (1.0 + np.exp(-params['W_c2'] @ h - params['b_c2']))  # Sigmoid output
    return color

def compute_sparsity_loss(densities, lambda_s):
    """Computes sparsity regularization loss [S1, S2]."""
    term = (densities ** 2) / np.log(1 + (densities ** 2) / 0.5)
    return lambda_s * np.sum(term)

def extract_occupancy_mask(densities, threshold):
    """Extracts unsupervised occupancy mask from density outputs [S1, S2]."""
    return densities > threshold

def prune_voxels(grid, mask, threshold):
    """Prunes voxels based on occupancy mask and threshold [S1, S2]."""
    grid.occupied = mask
    grid.features = grid.features[mask]

def hybrid_regularization_training(training_data, target_resolution):
    """Implements the Hybrid Regularization and Training Strategy [S1, S2]."""
    # Phase 1: Coarse Geometry Search
    coarse_res = target_resolution // 4
    coarse_grid = VoxelGrid(resolution=coarse_res)
    coarse_features = np.random.randn(coarse_grid.num_voxels, 32)
    coarse_mlp_params = init_random_mlp_params()
    coarse_decoder_params = init_random_decoder_params()
    
    for epoch in range(5):
        rays = sample_rays(training_data, batch_size=16384)
        features = coarse_features[rays.voxel_indices]
        densities = density_decoder_forward(coarse_decoder_params, features)
        occupancy_mask = extract_occupancy_mask(densities, threshold=0.01)
        prune_voxels(coarse_grid, occupancy_mask, threshold=0.01)
        # Backpropagate photometric loss with lr=1e-3
        coarse_mlp_params = adam_update(coarse_mlp_params, loss, lr=1e-3)
        
    # Discard coarse model
    coarse_features = None
    coarse_mlp_params = None
    coarse_decoder_params = None
    
    # Phase 2: Implicit Optimization
    implicit_mlp_params = init_random_mlp_params()
    fine_decoder_params = init_random_decoder_params()
    
    for epoch in range(1000):
        rays = sample_rays(training_data, batch_size=16384)
        features = mlp_forward(implicit_mlp_params, rays.vertex_coordinates)
        densities = density_decoder_forward(fine_decoder_params, features)
        colors = color_decoder_forward(fine_decoder_params, features)
        photometric_loss = np.mean((colors - rays.target_colors) ** 2)
        sparsity_loss = compute_sparsity_loss(densities, lambda_s=1e-5)
        total_loss = photometric_loss + sparsity_loss
        implicit_mlp_params = adam_update(implicit_mlp_params, total_loss, lr=5e-4)
        fine_decoder_params = adam_update(fine_decoder_params, total_loss, lr=5e-4)
        
    # Phase 3: Explicit Optimization
    explicit_grid = VoxelGrid(resolution=target_resolution)
    explicit_features = mlp_forward(implicit_mlp_params, explicit_grid.corners)
    implicit_mlp_params = None  # Discard MLP
    
    for epoch in range(1000):
        rays = sample_rays(training_data, batch_size=16384)
        features = interpolate_features(explicit_features, rays.vertex_coordinates)
        densities = density_decoder_forward(fine_decoder_params, features)
        colors = color_decoder_forward(fine_decoder_params, features)
        photometric_loss = np.mean((colors - rays.target_colors) ** 2)
        sparsity_loss = compute_sparsity_loss(densities, lambda_s=1e-5)
        total_loss = photometric_loss + sparsity_loss
        explicit_features = adam_update(explicit_features, total_loss, lr=5e-4)
        fine_decoder_params = adam_update(fine_decoder_params, total_loss, lr=5e-4)
        
        # Prune after fine stage
        occupancy_mask = extract_occupancy_mask(densities, threshold=0.01)
        prune_voxels(explicit_grid, occupancy_mask, threshold=0.01)
        
    return explicit_features, explicit_grid.occupancy_mask
```

## Subtopics

### [Feature Grid Optimization (Explicit vs Implicit Regularization)](18380-feature-grid-optimization-explicit-vs-implicit-regularizatio.md)
Explicit regularization directly optimizes grid values via gradient-based methods, requiring less time and GPU memory but producing lower quality metrics. Implicit regularization uses an MLP to predict features from spatial coordinates, offering higher quality but requiring more resources. Hybrid regularization combines both: the implicit stage provides a powerful initialization to avoid overfitting, and the explicit stage yields better final quality.

## Operating characteristics
*   **Time Complexity:** The method requires two sequential training stages (implicit and explicit), each running for 1000 epochs [S1, S2].
*   **Memory Complexity:** Peak GPU memory usage is approximately 40 GB on an NVIDIA A100 GPU during training [S1, S2].
*   **Performance:** The method achieves high-quality results and enables real-time rendering by leveraging sparsity and half-precision storage [S1, S2].
*   **Strengths:** Prevents overfitting in explicit grids, handles sparse inputs well, and enables real-time rendering through sparsity [S1, S2].
*   **Limitations:** Requires two distinct training phases and significant GPU memory [S1, S2].
*   **When to use:** When high-quality novel view synthesis is required and computational resources are sufficient [S1, S2].
*   **When not to use:** When training time or GPU memory is constrained [S1, S2].

## Background
The method was introduced in the NeRFLight paper by Rivas-Manzaneque et al. in 2023 [S1, S2]. It builds on the idea of hybrid regularization proposed in [48] (DIVeR-64) to address overfitting in deterministic integration [S1, S2]. The approach is part of the NeRFLight framework, which aims to provide fast and light neural radiance fields using a shared feature grid [S1, S2].

## Sources
[S1] NeRFLight: Fast and Light Neural Radiance Fields using a Shared Feature Grid, https://doi.org/10.1109/cvpr52729.2023.01195, fetched 2026-10-26
[S2] NeRFLight: Fast and Light Neural Radiance Fields using a Shared Feature Grid, https://openaccess.thecvf.com/content/CVPR2023/html/Rivas-Manzaneque_NeRFLight_Fast_and_Light_Neural_Radiance_Fields_Using_a_Shared_CVPR_2023_paper.html, fetched 2026-10-27
