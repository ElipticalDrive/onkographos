# Gaussian Splatting for Microscopy

Also called: Gaussian Splatting for Microscopy Volumes, nD Gaussian splatting

## Overview
Gaussian Splatting for Microscopy decomposes multidimensional fluorescence microscopy volumes into a mixture of oriented, anisotropic Gaussian primitives, optimizing them against the data to place representational capacity only where signal exists [S1, unofficial]. This approach solves the problem of data transfer and interactive viewing of large n-dimensional datasets by converting voxel grids into compact, GPU-rasterizable geometry, enabling scalable web visualization without intermediate voxel grids [S1, unofficial]. Unlike standard 3D Gaussian Splatting which uses view-dependent RGB color, this method uses view-independent scalar amplitudes, making it suitable for quantitative scientific data rather than artistic rendering [S1, unofficial].

## How it works

### Inputs and Outputs
*   **Input:** A preprocessed n-dimensional microscopy volume $V: \mathbb{R}^D \rightarrow \mathbb{R}_{\ge 0}$, typically a fused and deconvolved stack from multi-view light-sheet or confocal microscopy [S1, unofficial].
*   **Output:** A set of $K$ Gaussian splats, each defined by position, amplitude, and covariance, stored in a format suitable for streaming (e.g., Zarr archives) [S1, unofficial].

### Data Structures
Each Gaussian splat $k$ is parameterized by:
*   $\boldsymbol{\mu}_k \in \mathbb{R}^D$: Position (mean) of the Gaussian [S1, unofficial].
*   $a_k \in \mathbb{R}_{\ge 0}$: Amplitude (scalar intensity) [S1, unofficial].
*   $\mathbf{L}_k \in \mathbb{R}^{D \times D}$: Lower-triangular Cholesky factor of the covariance matrix $\mathbf{\Sigma}_k$, such that $\mathbf{\Sigma}_k = \mathbf{L}_k \mathbf{L}_k^\top$ [S1, unofficial].

The Cholesky parameterization stores $D(D+1)/2$ elements per splat (6 for 3D), guaranteeing positive-definite covariance by construction [S1, unofficial]. The diagonal of $\mathbf{L}_k$ is parameterized as $\sigma_{\min} + \text{softplus}(\cdot)$, where $\sigma_{\min} = 1/\sqrt{12}$ voxels (the standard deviation of a uniform voxel), ensuring no splat is narrower than a voxel's own spread [S1, unofficial]. Amplitudes are made non-negative by a softplus activation and are upper-bounded by a configurable maximum $a_{\max}$ [S1, unofficial].

### Preprocessing
1.  **Registration and Fusion:** Multi-view images are registered and fused with intensity equalization [S1, unofficial].
2.  **Deconvolution:** The fused volume is deconvolved (e.g., ten iterations against the point-spread function) to produce a single fused volume per timepoint [S1, unofficial].
3.  **Normalization:** Source volumes (stored as 16-bit integers or float32) are normalized to $[0, 1]$ using an affine map of the intensity range [S1, unofficial].
4.  **Clipping:** Outlier voxels are clipped based on dataset-specific percentiles to prevent noise from dominating the fit [S1, unofficial]:
    *   Plain min-max: Kidney, Cells3D, Blastocyst, Tribolium, Mouse heart.
    *   1st–99.5th: OpenCell.
    *   1st–99.999th: C. elegans.
    *   0th–99.999th: Sparse Fly brain.
    *   0th–99.9th: Zebrafish, Drosophila.
    *   Neuromast: Background floors (105.99 and 103.89 grey levels) are subtracted, then clipped to 0th–99.9th [S1, unofficial].
5.  **Background Floor Handling:** For quality benchmarks, no floor suppression is applied. For compression/timing benchmarks, automatic floor suppression subtracts a constant background pedestal before fitting [S1, unofficial].

### Initialization
Initial positions are seeded automatically from the volume using gradient-weighted samples of the volume’s edges, complemented by a coarse uniform grid [S1, unofficial].

### Optimization
The model is optimized using the Adam optimizer [S1, unofficial]. The reconstruction loss is L1 with regularization on splat amplitudes and extents [S1, unofficial].

**Fixed-Pool Optimization:**
Unlike standard 3D Gaussian Splatting which grows and prunes the Gaussian set, this method uses a fixed-pool optimizer. Periodically, the least informative splats are relocated to regions of high reconstruction residual. This acts as a fixed-pool analogue to pruning-and-seeding, keeping optimizer tensor shapes constant [S1, unofficial].

**Stopping Criterion (Blind-Spot Cross-Validation):**
The number of splats $K$ is determined by blind-spot cross-validation to prevent overfitting noise [S1, unofficial]:
1.  Mask 5% of voxels.
2.  Replace masked voxels with donut-median neighborhood values [S1, unofficial].
3.  Score reconstruction on held-out voxels.
4.  Stop fitting at the held-out PSNR peak. If the peak is interior or cull-limited, use the peak or the point where cull holds count constant [S1, unofficial].

### Rendering Formula
The volume prediction at voxel $\mathbf{x}$ is:
$$ \hat{V}(\mathbf{x}) = \sum_{k=1}^{K} a_k \exp\left(-\frac{1}{2} \| \mathbf{L}_k^{-1} (\mathbf{x} - \boldsymbol{\mu}_k) \|^2 \right) $$
where $\boldsymbol{\mu}_k$ is the position, $a_k$ is the amplitude, and $\mathbf{L}_k$ is the Cholesky factor [S1, unofficial].

**Viewer Evaluation:**
The viewer uses a truncated Gaussian to ensure compact support:
$$ \hat{V}_{\text{viewer}}(\mathbf{x}) = \sum_{k=1}^{K} a_k \max\left(0, \exp\left(-\frac{1}{2} \| \mathbf{L}_k^{-1} (\mathbf{x} - \boldsymbol{\mu}_k) \|^2 \right) - \frac{c_T}{1 - c_T} \right) $$
where $c_T = \exp(-T/2)$ at $T = 2.75$ standard deviations [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def preprocess_volume(volume, dataset_name):
    """
    Preprocesses a microscopy volume for fitting.
    volume: numpy array, raw intensity data.
    dataset_name: str, to determine clipping strategy.
    Returns: normalized volume in [0, 1].
    """
    # Step 1: Registration and Fusion (Assumed done externally)
    # Step 2: Deconvolution (Assumed done externally)
    
    # Step 3: Normalization and Clipping
    if dataset_name in ['Kidney', 'Cells3D', 'Blastocyst', 'Tribolium', 'Mouse heart']:
        v_min, v_max = np.min(volume), np.max(volume)
    elif dataset_name == 'OpenCell':
        v_min, v_max = np.percentile(volume, 1), np.percentile(volume, 99.5)
    elif dataset_name == 'C. elegans':
        v_min, v_max = np.percentile(volume, 1), np.percentile(volume, 99.999)
    elif dataset_name == 'Fly brain':
        v_min, v_max = np.percentile(volume, 0), np.percentile(volume, 99.999)
    elif dataset_name in ['Zebrafish', 'Drosophila']:
        v_min, v_max = np.percentile(volume, 0), np.percentile(volume, 99.9)
    elif dataset_name == 'Neuromast':
        # Subtract background floors
        floor1, floor2 = 105.99, 103.89
        volume = volume - floor1 # Simplified for single channel example
        volume = np.clip(volume, 0, None)
        v_min, v_max = np.percentile(volume, 0), np.percentile(volume, 99.9)
    else:
        v_min, v_max = np.min(volume), np.max(volume)
        
    # Affine map to [0, 1]
    normalized = (volume - v_min) / (v_max - v_min)
    return normalized

def seed_initial_splats(volume, num_splats):
    """
    Seeds initial Gaussian positions.
    """
    # Gradient-weighted samples of edges
    gradients = np.gradient(volume)
    edge_strength = np.linalg.norm(gradients, axis=0)
    # Sample positions proportional to edge strength
    positions = sample_weighted(volume.shape, edge_strength, num_splats // 2)
    
    # Coarse uniform grid
    grid_positions = generate_uniform_grid(volume.shape, num_splats // 2)
    
    return np.vstack([positions, grid_positions])

def fit_gaussians(volume, initial_positions, max_splats, max_iterations=20000):
    """
    Fits Gaussian splats to the volume using fixed-pool optimization.
    """
    # Initialize parameters
    # mu: [K, D]
    # L: [K, D, D] (Lower triangular Cholesky)
    # a: [K] (Amplitudes)
    
    K = max_splats
    mu = initial_positions
    L = initialize_cholesky(K, volume.shape[0]) # Diagonal init with sigma_min
    a = np.ones(K)
    
    # Optimizer state (Adam)
    m_mu, v_mu = np.zeros_like(mu), np.zeros_like(mu)
    m_L, v_L = np.zeros_like(L), np.zeros_like(L)
    m_a, v_a = np.zeros_like(a), np.zeros_like(a)
    
    best_held_out_psnr = -np.inf
    optimal_K = K
    patience = 0
    
    for t in range(max_iterations):
        # 1. Forward Pass: Compute reconstruction error
        # Reconstruct volume at sampled voxels
        # loss = L1(volume, reconstructed) + regularization(a, L)
        
        # 2. Backward Pass: Update parameters
        # gradients = compute_gradients(loss, mu, L, a)
        # update_adam(mu, m_mu, v_mu, gradients_mu)
        # update_adam(L, m_L, v_L, gradients_L)
        # update_adam(a, m_a, v_a, gradients_a)
        
        # 3. Fixed-Pool Relocation (Periodically)
        if t % 100 == 0:
            # Identify least informative splats (e.g., low amplitude or high error contribution)
            # Relocate them to regions of high reconstruction residual
            pass
            
        # 4. Blind-Spot Cross-Validation (Periodically)
        if t % 500 == 0:
            held_out_psnr = evaluate_cross_validation(volume, mu, L, a)
            if held_out_psnr > best_held_out_psnr:
                best_held_out_psnr = held_out_psnr
                optimal_K = K # Or current active count if culling is involved
                patience = 0
            else:
                patience += 1
                
            if patience > 500:
                break
                
    return mu, L, a

def evaluate_cross_validation(volume, mu, L, a):
    """
    Masks 5% of voxels, replaces with donut-median, and scores PSNR on held-out voxels.
    """
    mask = np.random.rand(*volume.shape) < 0.05
    masked_volume = volume.copy()
    masked_volume[mask] = donut_median(volume, mask)
    
    # Reconstruct masked volume
    reconstructed = reconstruct_volume(masked_volume, mu, L, a)
    
    # Calculate PSNR on held-out voxels (original values)
    mse = np.mean((volume[mask] - reconstructed[mask]) ** 2)
    psnr = 10 * np.log10(1.0 / mse) # Assuming normalized [0,1]
    return psnr

def donut_median(volume, mask):
    """
    Replaces masked voxels with the median of their neighborhood, excluding the center.
    """
    # Implementation of donut median filter
    pass

def reconstruct_volume(volume, mu, L, a):
    """
    Reconstructs the volume from Gaussian splats.
    Uses truncated Gaussian for viewer evaluation.
    """
    # Initialize output volume
    output = np.zeros_like(volume)
    
    # For each voxel, sum contributions from all splats
    # Optimization: Only consider splats within bounding box
    for x in voxels:
        val = 0.0
        for k in range(len(mu)):
            diff = x - mu[k]
            # Compute Mahalanobis distance
            dist_sq = np.dot(diff, np.linalg.solve(L[k], diff)) # L[k] is lower triangular
            # Truncated Gaussian
            c_T = np.exp(-2.75 / 2)
            term = np.exp(-0.5 * dist_sq) - c_T / (1 - c_T)
            val += a[k] * max(0, term)
        output[x] = val
    return output
```

### Edge Cases and Pitfalls
*   **Overfitting:** Adding more splats beyond the optimal budget causes the model to memorize noise rather than signal, indicated by a growing gap between training PSNR and held-out PSNR [S1, unofficial].
*   **Noise Assumption:** Blind-spot cross-validation assumes noise is independent across voxels. On fused or deconvolved volumes, a held-out peak is not guaranteed. If a noise floor is estimable, it caps the budget from above to avoid reproducing noise [S1, unofficial].
*   **Resolution Limits:** Structures thinner than the local Gaussian scale (e.g., individual microtubules) blur. The representation is for looking, not measuring [S1, unofficial].
*   **Convergence:** Convergence is rapid; at 2,000 iterations, training PSNR is within 0.5 dB of the 20,000-iteration value for most volume-capacity cells [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Fitting Time:** Median 2.4 minutes per volume at a 32K splat budget on an NVIDIA RTX 3070 (8 GiB) [S1, unofficial].
*   **Scaling:** Adding capacity (splats) outperforms adding iterations. 16K splats × 500 iterations beat 4K splats × 20,000 iterations by an average of +3.03 dB across 17 volumes at 10× fewer splat-iterations [S1, unofficial].
*   **Memory:** Fits store volumes 6–340× smaller (median 99.3×) at 26–67 dB PSNR [S1, unofficial].

### Measured Performance
*   **Tribolium embryo nuclei:** 51.4 dB PSNR at 32K splats, 54.9 dB at 512K, 56.0 dB at 2M splats [S1, unofficial].
*   **Kidney nuclei:** 34.8 dB PSNR at 32K splats, 45.2 dB at 512K splats [S1, unofficial].
*   **Rendering:** On an RTX 3070 at 720p HDR, Gaussian splats maintain 60 FPS (16.7 ms budget) through $10^6$ elements [S1, unofficial].

### Strengths and Limitations
*   **Strengths:** Direct GPU rasterization without voxel grids; suitable for streaming where bandwidth and memory are constrained; fast fitting; high compression ratios [S1, unofficial].
*   **Limitations:** Structures thinner than the local Gaussian scale blur; not suitable for precise quantification or measurement; fitting is asymmetric (expensive fitting, cheap viewing) [S1, unofficial].

### When to Use
*   Use for interactive web visualization and sharing of large n-dimensional microscopy datasets.
*   Use when bandwidth and client-side memory are constrained.
*   Do not use for precise quantitative analysis, segmentation, or tracking; use source voxels for those tasks [S1, unofficial].

## Background
Gaussian Splatting for Microscopy is part of the Luxar framework, an open-source Python system designed for scalable interactive web visualization of multidimensional scientific data [S1, unofficial]. It adapts nD Gaussian splatting for microscopy, validating the approach on 17 volumes from 12 datasets spanning spinning-disk, confocal, light-sheet, and structured-illumination modalities [S1, unofficial]. The method uses blind-spot cross-validation to determine the optimal splat count, balancing PSNR and overfitting [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
