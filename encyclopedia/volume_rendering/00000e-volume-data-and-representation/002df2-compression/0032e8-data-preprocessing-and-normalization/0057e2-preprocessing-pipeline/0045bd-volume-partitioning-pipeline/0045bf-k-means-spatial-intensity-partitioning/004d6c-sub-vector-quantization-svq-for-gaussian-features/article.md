# Sub-Vector Quantization (SVQ) for Gaussian Features

Also called: Scalar Quantization (length-1 sub-vectors)

## Overview
Sub-Vector Quantization (SVQ) is a compression technique applied to per-Gaussian features (scale, rotation, appearance) that uses fixed K-means cluster indices and dynamically optimized codebooks to reduce storage footprint [S1]. It sets itself apart from sibling methods by decoupling cluster assignment from codebook optimization, allowing for feature-specific sub-vector lengths and dynamic range learning via 16-bit floating-point codebooks [S1]. Unlike static quantization, SVQ adapts the quantization range during the final training phase, offering a trade-off between storage efficiency and attribute representation accuracy [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Per-Gaussian feature vectors for Scale, Rotation, and Appearance [S1].
*   Pre-computed K-means cluster indices for each Gaussian (fixed from prior clustering) [S1].
*   Hyperparameters: Sub-vector lengths ($L$), codebook sizes ($C$), learning rate, and iteration count [S1].

**Outputs:**
*   Optimized codebook vectors for each feature type [S1].
*   Quantized feature representations stored as cluster indices and codebook lookups [S1].

### Data Structures
The method stores codebook entries with 16-bit floating-point (16-FP) precision [S1]. The structure for each feature type is defined by its sub-vector length and codebook size:

1.  **Scale:**
    *   Sub-vector length ($L_{scale}$): 1
    *   Codebook size ($C_{scale}$): $2^6 = 64$ codes per sub-vector [S1].
    *   Interpretation: Functions as scalar quantization [S1].
2.  **Rotation:**
    *   Sub-vector length ($L_{rot}$): 2
    *   Codebook size ($C_{rot}$): $2^9 = 512$ codes per sub-vector [S1].
3.  **Appearance:**
    *   Sub-vector length ($L_{app}$): 2
    *   Codebook size ($C_{app}$): $2^{10} = 1024$ codes per sub-vector [S1].
    *   Dimensionality ($D$): 3 for the appearance features vector [S1].

### Steps
1.  **K-Means Clustering:** Perform K-means clustering once on the target features (Scale, Rotation, Appearance) [S1].
2.  **Fix Indices:** Assign each Gaussian to a cluster based on the K-means result. Fix these assigned cluster indices for all remaining training iterations [S1].
3.  **Initialize Codebooks:** Initialize the codebook vectors for each feature type. For appearance features, view-dependent features are initialized as zero vectors, while static features are initialized using spherical harmonics DC coefficients trained up to that point [S1].
4.  **Optimize Codebooks:** Optimize only the codebook vectors for the remaining training iterations, keeping the cluster indices fixed [S1]. This allows the codebooks to dynamically learn the quantization range [S1].
5.  **Storage:** Store the optimized codebook entries with 16-FP precision [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pre-computed K-means indices are fixed for all Gaussians: scale_idx, rot_idx, app_idx
# Codebooks are initialized with 16-FP precision [S1].
# Appearance view-dependent features are initialized as zero vectors, static features use SH DC coefficients [S1].

def initialize_codebooks(feature_type, length, num_codes):
    """Initialize codebook vectors for a specific feature type."""
    if feature_type == "appearance":
        # Static features initialized from SH DC coefficients
        # View-dependent features initialized as zero vectors
        codebook = [[0.0] * length for _ in range(num_codes)]
    else:
        # Standard initialization for scale and rotation
        codebook = [[0.0] * length for _ in range(num_codes)]
    return codebook

def svq_train(features, codebook, indices, learning_rate, iterations):
    """Optimize codebook vectors while keeping cluster indices fixed."""
    for _ in range(iterations):
        # Forward pass: retrieve quantized features using fixed indices
        quantized = [codebook[i] for i in indices]
        # Compute reconstruction loss (MSE)
        loss = sum((f - q)**2 for f, q in zip(features, quantized)) / len(features)
        # Compute gradients w.r.t. codebook entries
        grad = [-2 * (f - q) / len(features) for f, q in zip(features, quantized)]
        codebook_grad = [grad[i] for i in indices]
        # Update codebook vectors only (gradient descent) [G]
        for k in range(len(codebook)):
            if k in indices:
                codebook[k] = [c - learning_rate * g for c, g in zip(codebook[k], codebook_grad[indices.index(k)])]
    return codebook

# Training loop for each feature type (standard training: 1K iterations)
scale_codebook = initialize_codebooks("scale", length=1, num_codes=64)
rot_codebook = initialize_codebooks("rotation", length=2, num_codes=512)
app_codebook = initialize_codebooks("appearance", length=2, num_codes=1024)

scale_codebook = svq_train(scale_features, scale_codebook, scale_idx, lr, 1000)
rot_codebook = svq_train(rot_features, rot_codebook, rot_idx, lr, 1000)
app_codebook = svq_train(app_features, app_codebook, app_idx, lr, 1000)
# Note: For Zip-NeRF dataset, iterations = 10000 [S1].
```

### Parameters
*   **$D=3$**: Dimensionality of the appearance features vector [S1].
*   **$\tau$**: Importance score threshold that determines the final model storage size and variant selection (XS to XL) [S1]. Typical values: 0.96, 0.98, 0.99, 0.999, and 0.999 [S1].
*   **Bit Allocations**:
    *   Scale: $2^6$ codes [S1].
    *   Rotation: $2^9$ codes [S1].
    *   Appearance: $2^{10}$ codes [S1].

### Edge Cases and Pitfalls
*   **Convergence Failure:** Directly training Compact-3DGS with Mini-Splatting-based pruning fails to converge due to inaccurate attribute representations [S1].
*   **Mitigation:** Halt Mini-Splatting training at 20K iterations (when pruning ends), then integrate Compact-3DGS representation (including SVQ) for the subsequent 10K iterations [S1].
*   **Visual Degradation:** Integrating Mini-Splatting without this staged approach results in significant visual quality degradation, primarily caused by imprecise attribute representations for irregular and sparse Gaussians [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Memory:** SVQ significantly reduces storage footprint. OMG-XS achieves 4.06 MB storage compared to LocoGS's 7.96 MB, effectively halving the storage size [S1].
*   **Time:** Rendering FPS is substantially higher due to the reduced number of Gaussians. On an NVIDIA GTX 1080 Ti, OMG-XS achieves 106 FPS compared to LocoGS's 73 FPS (1.5× faster) [S1]. On an NVIDIA RTX 4090, OMG-XS achieves 612 FPS vs LocoGS's 396 FPS [S1].

### Strengths and Limitations
*   **Strengths:** Drastically reduced storage and increased rendering speed while preserving high visual fidelity [S1]. The dynamic learning of quantization ranges via codebooks allows for efficient representation of feature distributions [S1].
*   **Limitations:** Requires a staged training approach to avoid convergence issues. The fixed K-means indices may not adapt to changes in feature distribution during the final optimization phase, relying entirely on codebook updates [S1].

### When to Use
*   Use SVQ when storage efficiency and rendering speed are critical, and the training pipeline can accommodate the staged integration of Compact-3DGS representation [S1].
*   Suitable for deployment on lower-end GPUs (e.g., GTX 1080 Ti) where real-time performance is desired [S1].

## Background
SVQ is part of the Optimized Minimal 3D Gaussian Splatting (OMG) method, introduced in "Optimized Minimal 3D Gaussian Splatting" [S1]. The method was implemented within the Mini-Splatting framework and trained for 30K iterations (or 150K for Zip-NeRF dataset) [S1]. SVQ is applied in the final phase of training (last 1K iterations for standard training, last 10K for Zip-NeRF) [S1]. The approach was developed to address the challenges of compressing a small number of Gaussians while maintaining visual quality [S1].

## Sources
[S1] Optimized Minimal 3D Gaussian Splatting, https://doi.org/10.52202/085713-4534, fetched 2026-10-30
