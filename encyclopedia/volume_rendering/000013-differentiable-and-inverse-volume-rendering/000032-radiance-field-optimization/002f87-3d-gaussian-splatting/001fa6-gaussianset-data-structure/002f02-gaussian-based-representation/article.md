# Gaussian-based Representation

Also called: Gaussian mixture model, Gaussian splatting

## Overview
Gaussian-based representation encodes signals, data, or geometric structures as an explicit, continuous sum (or mixture) of parameterized Gaussian functions [S1, unofficial]. It solves the problem of representing high-dimensional data with a structure that allows for efficient, closed-form operations such as differentiation and convolution, enabling fast rendering and analysis [S1, unofficial]. What sets it apart from its siblings and nearest alternatives is its use of explicit, analytically differentiable primitives rather than implicit neural fields or grid-based voxels; while [3D Gaussian Splatting (3DGS) Preliminaries](2257-3d-gaussian-splatting-3dgs-preliminaries.md) focuses on the specific rasterization pipeline for novel view synthesis, this method generalizes to 2D image compression, vector search, and physics simulation [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   A target signal or domain (e.g., a 2D image, a 3D scene, a vector dataset) [S1, unofficial].
*   A set of reference samples (e.g., pixel values, multi-view images, ground-truth vectors) for optimization [S1, unofficial].

**Outputs:**
*   A set of $N$ Gaussian primitives, each defined by location, covariance, and amplitude/attributes [S1, unofficial].
*   The reconstructed signal $f(\mathbf{x})$ evaluated at any point $\mathbf{x}$ [S1, unofficial].

### Data Structures
The representation consists of a list of $N$ Gaussians. The specific fields depend on the domain:

**2D Image Domain (e.g., GaussianImage):**
Each Gaussian is parameterized by an 8-tuple [S1, unofficial]:
1.  **Location** $\boldsymbol{\mu} \in \mathbb{R}^2$: 2 floats.
2.  **Covariance** $\boldsymbol{\Sigma} \in \mathbb{R}^{2 \times 2}$: Stored via Cholesky decomposition, requiring 3 floats (since $\boldsymbol{\Sigma}$ is symmetric positive-definite) [S1, unofficial].
3.  **Color** $\mathbf{c} \in \mathbb{R}^3$: 3 floats (RGB) [S1, unofficial].

**3D Scene Domain (e.g., 3DGS):**
Each Gaussian includes additional attributes to handle view-dependence and 3D geometry [S1, unofficial]:
1.  **Mean** $\boldsymbol{\mu}_n \in \mathbb{R}^3$: 3 floats.
2.  **Rotation** $\mathbf{q}_n \in \mathbb{R}^4$: Unit quaternion (4 floats) [S1, unofficial].
3.  **Scale** $\mathbf{s}_n \in \mathbb{R}^3$: 3 floats.
4.  **Opacity** $\alpha_n \in \mathbb{R}$: 1 float.
5.  **Spherical Harmonics (SH) Coefficients**: Variable size depending on order $L$ (e.g., $3(L+1)^2$ floats for RGB) [S1, unofficial].
6.  **Covariance** $\boldsymbol{\Sigma}_n$: Constructed from rotation and scale as $\boldsymbol{\Sigma}_n = \mathbf{R}_n \mathbf{S}_n \mathbf{S}_n^\top \mathbf{R}_n^\top$ [S1, unofficial].

### Formulas
The canonical primitive in $d$-dimensional Euclidean space is [S1, unofficial]:

$$
G_n(\mathbf{x}) = a_n \cdot \exp\left(-\frac{1}{2}(\mathbf{x}-\boldsymbol\mu_n)^\top \boldsymbol\Sigma_n^{-1} (\mathbf{x}-\boldsymbol\mu_n)\right)
$$

Where:
*   $\boldsymbol\mu_n \in \mathbb{R}^d$ is the center (mean) [S1, unofficial].
*   $\boldsymbol\Sigma_n \in \mathbb{R}^{d \times d}$ is the symmetric positive-definite covariance matrix (shape) [S1, unofficial].
*   $a_n$ is the amplitude or attribute vector (e.g., RGB color, opacity, semantic embedding) [S1, unofficial].

The total representation is the superposition:

$$
f(\mathbf{x}) = \sum_{n=1}^N G_n(\mathbf{x})
$$

The physical or semantic meaning of $f$ depends on context (image intensity, field value, feature embedding, etc.) [S1, unofficial].

### Parameters
*   $N$: The number of Gaussians in the mixture [S1, unofficial].
*   $d$: The dimensionality of the Euclidean space (e.g., 2 for images, 3 for scenes) [S1, unofficial].
*   **Loss Function**:
    *   **2D Image**: Typically L2 (MSE) or hybrid L2+D-SSIM over all pixels [S1, unofficial].
    *   **3D Scene**: Photometric losses over multiple views or ray samples, combined with sparsity, density, or geometric constraints [S1, unofficial].

### Numbered Steps

1.  **Initialization**:
    *   *Random/Iterative*: Initialize $N$ Gaussians with random locations and shapes [S1, unofficial].
    *   *Vectorized*: Use methods like EigenGS to map a principal-component eigenspace to Gaussian space for instant initialization [S1, unofficial].
    *   *Feedforward*: Use a predictive network to infer Gaussian parameters or densities in a single pass [S1, unofficial].

2.  **Optimization (Fitting)**:
    *   Define the loss function $\mathcal{L}$ between the rendered/evaluated sum $f(\mathbf{x})$ and the reference signal [S1, unofficial].
    *   Optimize the parameters ($\boldsymbol\mu_n, \boldsymbol\Sigma_n, a_n$, and optional attributes like rotation/SH) using gradient descent [S1, unofficial].

3.  **Adaptive Deployment (Optional)**:
    *   **Densification**: Where residuals concentrate, create additional Gaussians (e.g., via SplitNet or confidence-aware splitting) [S1, unofficial].
    *   **Pruning**: Remove Gaussians where coverage is redundant or negligible [S1, unofficial].
    *   **Hierarchical Fitting**: For large signals, establish a coarse, low-frequency backbone first, then add high-frequency residual Gaussians (Level-of-Gaussian fitting) [S1, unofficial].

4.  **Rendering/Evaluation**:
    *   **2D Image**: Compute the permutation-invariant summation $\sum G_n(\mathbf{x})$ for each pixel. Alpha blending is unnecessary as occlusion is irrelevant [S1, unofficial].
    *   **3D Scene**: Project each Gaussian to 2D, depth-sort, and alpha-blend front-to-back to yield correct occlusion and volumetric effects [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def gaussian_2d(x, mu, chol_sigma, color):
    """
    Evaluate a 2D Gaussian primitive.
    x: (2,) array, point location
    mu: (2,) array, mean
    chol_sigma: (3,) array, Cholesky factor of covariance matrix
    color: (3,) array, RGB color
    """
    # Reconstruct covariance from Cholesky factor
    L = np.array([
        [chol_sigma[0], 0],
        [chol_sigma[1], chol_sigma[2]]
    ])
    Sigma = L @ L.T
    
    # Mahalanobis distance
    diff = x - mu
    inv_Sigma = np.linalg.inv(Sigma)
    exponent = -0.5 * diff @ inv_Sigma @ diff
    
    # Amplitude (opacity/intensity) is implicitly 1.0 for standard image fitting 
    # or stored in color if normalized. Here we assume color scales the intensity.
    # In GaussianImage, the sum of Gaussians equals the image intensity.
    weight = np.exp(exponent)
    
    return weight * color

def render_2d_image(gaussians, image_shape):
    """
    Render a 2D image from a list of 2D Gaussians.
    gaussians: list of dicts, each with 'mu' (2,), 'chol_sigma' (3,), 'color' (3,)
    image_shape: (H, W) tuple
    """
    H, W = image_shape
    image = np.zeros((H, W, 3))
    
    # Create coordinate grid
    y, x = np.meshgrid(np.arange(H), np.arange(W), indexing='ij')
    points = np.stack([x, y], axis=-1).reshape(-1, 2)
    
    # Sum contributions from all Gaussians
    for g in gaussians:
        contributions = np.array([gaussian_2d(p, g['mu'], g['chol_sigma'], g['color']) 
                                  for p in points])
        image_flat = contributions.reshape(H, W, 3)
        image += image_flat
        
    return image

def optimize_gaussians_2d(gaussians, target_image, num_iterations=1000, lr=0.01):
    """
    Simple gradient descent optimizer for 2D Gaussian Image fitting.
    Note: In practice, automatic differentiation frameworks are used.
    """
    for i in range(num_iterations):
        # Forward pass
        rendered = render_2d_image(gaussians, target_image.shape)
        
        # Loss (L2/MSE)
        loss = np.mean((rendered - target_image) ** 2)
        
        # Backward pass (conceptual)
        # gradients = compute_gradients(loss, gaussians)
        # update_gaussians(gaussians, gradients, lr)
        
        if i % 100 == 0:
            print(f"Iter {i}, Loss: {loss}")
            
    return gaussians
```

### Edge Cases and Pitfalls
*   **High-Frequency Structures**: Very high-frequency structures may require many primitives or specialized learning schedules due to trade-offs between parameter count and signal complexity [S1, unofficial].
*   **Parameter Redundancy**: Potential for parameter redundancy or non-uniqueness if the base space is not managed (e.g., ambiguous quaternions or covariances in 3DGS) [S1, unofficial].
*   **2D vs 3D Aggregation**: Using alpha blending for 2D image fitting is incorrect; 2D uses permutation-invariant summation [S1, unofficial].

## Subtopics

### [GaussianSet Data Structure](2074-gaussianset-data-structure.md)
The foundational class storing means, orientations, scales, opacities, and SH coefficients. Choose this when you need a generic container for Gaussian primitives.

### [3D Gaussian Splatting (3DGS) Preliminaries](2257-3d-gaussian-splatting-3dgs-preliminaries.md)
Focuses on the specific rasterization and differentiable rendering pipeline for 3D scenes. Choose this for novel view synthesis.

### [ECoNGS Optimization and Coding](2087-econgs-optimization-and-coding.md)
Focuses on efficient coding and optimization strategies. Choose this when compression or bit-rate efficiency is the primary goal.

### [GaussianList Data Structure and Initialization](2214-gaussianlist-data-structure-and-initialization.md)
Details on list-based storage and initialization techniques. Choose this for memory-efficient sequential access.

### [Gaussian Property Loading](2221-gaussian-property-loading.md)
Mechanisms for loading pre-computed Gaussian properties. Choose this for inference-only pipelines.

### [3D Covariance Computation](2231-3d-covariance-computation.md)
Specific algorithms for computing 3D covariance matrices from rotation and scale. Choose this when implementing the 3D primitive math.

### [3D Gaussian Data Layout](2242-3d-gaussian-data-layout.md)
Memory layout optimizations for GPU rendering. Choose this for performance-critical rendering engines.

### [N-DGS Bounded Cholesky Parameterization](2811-n-dgs-bounded-cholesky-parameterization.md)
Advanced parameterization for stability. Choose this when training stability is an issue.

## Operating characteristics
*   **Time Complexity**: Rendering is highly parallelizable. 2D GaussianImage achieves 1000–2000 FPS, orders of magnitude faster than MLP-based Implicit Neural Representations (INRs) or classic codecs (10–150 FPS) [S1, unofficial].
*   **Memory Complexity**: Depends on $N$. For 2D, GaussianImage uses 0.4 GB for high-quality compression [S1, unofficial]. For 3D, 3DGS uses 746 MB for Mip-NeRF 360 scenes, while compact versions use 29 MB [S1, unofficial].
*   **Measured Performance**:
    *   **GaussianImage (2D)**: 2,000 FPS, 0.4 GB memory, 44.1 dB PSNR [S1, unofficial].
    *   **LIG (9K×9K Image)**: 20 FPS, 16.7–20.3 GB memory, 37.5–42.2 dB PSNR [S1, unofficial].
    *   **3DGS (Mip-NeRF 360)**: 120 FPS, 746 MB memory, 27.46 dB PSNR [S1, unofficial].
    *   **Compact 3DGS (+PP)**: 128 FPS, 29 MB memory, 27.03 dB PSNR [S1, unofficial].
    *   **Hybrid mesh–GS**: 231 FPS, 0.74M splats, 24.28 dB PSNR [S1, unofficial].
    *   **GARLIC (Vector Search)**: Recall@1=0.69 on SIFT1M 128D [S1, unofficial].
*   **Strengths**: Analytic differentiability, closed-form operators, massive parallelism, adaptivity via densification/pruning [S1, unofficial].
*   **Limitations**: High parameter count for very high-frequency signals; potential for non-uniqueness/redundancy [S1, unofficial].
*   **When to Use**: When fast decoding/rendering is required, or when analytic gradients are needed for optimization/simulation [S1, unofficial].
*   **When Not to Use**: When extreme compression is needed without adaptive quantization, or when the signal is extremely high-frequency and cannot be captured efficiently by a limited number of Gaussians [S1, unofficial].

## Background
Gaussian-based representations leverage the mathematical properties of Gaussian functions, which are closed under differentiation and convolution [S1, unofficial]. The approach has been applied to image coding, 3D scene reconstruction, and vector search [S1, unofficial]. Recent advancements include feedforward initialization (EigenGS), hybrid mesh-Gaussian decomposition, and graph-based relational learning [S1, unofficial]. The method is distinct from grid-based or MLP-based representations by its explicit, parametric nature [S1, unofficial].

## Sources
[S1] Gaussian-Based Representation, https://www.emergentmind.com/topics/gaussian-based-representation, fetched 2026-10-26 (unofficial)
