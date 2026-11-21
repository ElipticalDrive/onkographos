# Variable Basis Mapping (VBM)


## Overview
Variable Basis Mapping (VBM) is a framework that transforms volumetric fields into 3D Gaussian Splatting (3DGS) representations through wavelet-domain analysis, enabling real-time volumetric visualization by avoiding the prohibitively high computational costs of direct volume rendering [S1, unofficial]. Unlike iVR-GS, which uses randomly initialized Gaussians, or VEG, which uses heuristic density-based spheres, VBM establishes a direct, mathematical bridge using wavelet analysis to exploit structural and statistical information [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A physical volumetric field $F$ discretized into a uniform grid of resolution $G^3$, where $G$ is the volumetric data resolution [S1, unofficial].
*   **Intermediate:** A radiance field $RF \in \mathbb{R}^{G \times G \times G \times 4}$ obtained by applying a transfer function $T_F: \mathbb{R}^r \to \mathbb{R}^4$ to $F$, producing RGBA vectors (color $c$, opacity $\alpha$) [S1, unofficial].
*   **Output:** A set of $N$ anisotropic 3D Gaussian primitives, each defined by center $\mu_i \in \mathbb{R}^3$, covariance matrix $\Sigma_i \in \mathbb{R}^{3\times3}$, color $c_i$, and opacity $\alpha_i$ [S1, unofficial].

**Data Structures**
1.  **Wavelet-to-Gaussian Transition Bank:** A precomputed lookup structure mapping canonical wavelet atoms to optimal Gaussian surrogates. It stores the parameters $(\mu, \Sigma, c, \alpha)$ for the canonical kernel $\psi_{j,k=(0,0,0)}$ at each decomposition level $j$.
2.  **Gaussian Primitive:** Defined by the tuple $(\mu_i, \Sigma_i, c_i, \alpha_i)$. The radiance field is approximated by:
    $$ RF(x) \approx \sum_{i=1}^{N} c_i \frac{\alpha_i}{\sqrt{\det(\Sigma_i)}} \exp\left(-\frac{1}{2}(x - \mu_i)^\top \Sigma_i^{-1} (x - \mu_i)\right) $$
    where $\mu_i$ is the center, $\Sigma_i$ is the covariance matrix, $c_i$ is the color, and $\alpha_i$ is the opacity [S1, unofficial].

**Steps**
1.  **Precompute Transition Bank:** Compute optimal Gaussian surrogates for canonical wavelet atoms across multiple scales. To avoid cubic complexity ($J \times G^3$ fittings), only the canonical kernel $\psi_{j,k=(0,0,0)}$ is computed for each level $j$. Remaining kernels are derived analytically via translation and scaling, guaranteed by the group representation theorem [S1, unofficial].
2.  **Wavelet Decomposition:** Express the radiance field $RF$ in a compact multi-scale form using the 3D Discrete Wavelet Transform (DWT). The field is decomposed into low-frequency coefficients $c_{0,k}$ and high-frequency coefficients $d_{j,k}$:
    $$ RF(x) \approx \sum_{k} c_{0,k} \phi(x - k) + \sum_{j=0}^{J-1} \sum_{k} d_{j,k} \psi_{j,k}(x) $$
    where $J$ is the number of decomposition levels, $\phi$ is the scaling function, and $\psi_{j,k}$ are dyadic wavelet atoms defined as $\psi_{j,k}(x) = 2^{3j/2} \psi(2^j x - k)$ [S1, unofficial].
3.  **Analytical Gaussian Construction:** Map the discrete wavelet coefficients directly to 3DGS parameters using the precomputed Transition Bank. Each wavelet basis $\psi_{j,k}$ is mapped to a Gaussian basis, and the associated coefficients $c_{j,k}$ (color) and $\alpha_{j,k}$ (opacity) are translated into the corresponding 3DGS parameters $(\mu, \Sigma, c, \alpha)$ [S1, unofficial].
4.  **Image-Space Fine-Tuning:** Apply a lightweight optimization stage in image space to refine the representation and improve rendering fidelity [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def variable_basis_mapping(physical_field_F, resolution_G, num_levels_J, transfer_function):
    """
    Transforms a volumetric field into 3DGS using VBM.
    
    Args:
        physical_field_F: ndarray of shape (G, G, G, r)
        resolution_G: int, dimension of the cubic grid
        num_levels_J: int, number of wavelet decomposition levels
        transfer_function: callable, maps physical values to RGBA
        
    Returns:
        gaussians: list of Gaussian primitives (mu, Sigma, c, alpha)
    """
    # Step 1: Convert physical field to radiance field
    # RF shape: (G, G, G, 4)
    RF = transfer_function(physical_field_F)
    
    # Step 2: Precompute Wavelet-to-Gaussian Transition Bank
    # Only compute for canonical kernel at k=(0,0,0) for each scale j
    transition_bank = {}
    for j in range(num_levels_J):
        # Compute optimal Gaussian surrogate for canonical wavelet atom psi_j,0
        # This involves statistical estimation of the best-fit Gaussian parameters
        # for the shape defined by psi_j,0(x) = 2^(3j/2) * psi(2^j * x)
        canonical_gaussian = compute_canonical_gaussian_surrogate(j)
        transition_bank[j] = canonical_gaussian
        
    # Step 3: Perform 3D Wavelet Decomposition on RF
    # Decompose each channel (R, G, B, A) separately
    low_freq_coeffs = {} # c_0,k
    high_freq_coeffs = {} # d_j,k
    
    for channel_idx in range(4):
        channel_data = RF[:, :, :, channel_idx]
        # Perform DWT
        coeffs = perform_dwt_3d(channel_data, num_levels_J)
        low_freq_coeffs[channel_idx] = coeffs['low']
        high_freq_coeffs[channel_idx] = coeffs['high']
        
    # Step 4: Map Wavelet Coefficients to Gaussian Primitives
    gaussians = []
    
    # Map low-frequency scaling function coefficients
    # phi(x-k) maps to a Gaussian centered at k with parameters derived from transition bank
    # Note: Scaling function phi is not in the wavelet bank, so a separate mapping or 
    # approximation is required. [G: Assume a standard Gaussian approximation for phi or 
    # treat low-freq as a single large Gaussian or set of Gaussians based on grid spacing]
    for k in grid_indices(resolution_G):
        # Extract coefficients for all channels at position k
        c_low = [low_freq_coeffs[ch][k] for ch in range(4)]
        # Map to Gaussian parameters. 
        # Since phi is not in the wavelet bank, we approximate or use a default mapping.
        # [G: Use a fixed Gaussian shape for the scaling function component, 
        # centered at k, with width related to grid resolution]
        mu = k
        Sigma = compute_scaling_function_covariance(resolution_G)
        c = c_low[:3]
        alpha = c_low[3]
        gaussians.append((mu, Sigma, c, alpha))
        
    # Map high-frequency wavelet coefficients
    for j in range(num_levels_J):
        for k in grid_indices_at_scale(j, resolution_G):
            # Get canonical Gaussian parameters for this scale j
            base_gaussian = transition_bank[j]
            
            # Apply translation and scaling to base_gaussian to match psi_j,k
            # Due to translation covariance, mu and Sigma are transformed based on k and j
            mu = transform_translation(base_gaussian.mu, k, j)
            Sigma = transform_scaling(base_gaussian.Sigma, j)
            
            # Extract wavelet coefficients for this atom
            c_high = [high_freq_coeffs[ch][j][k] for ch in range(4)]
            
            # Scale the color and opacity by the wavelet coefficient magnitude
            # The wavelet coefficient acts as a weight for the basis function
            # [G: The paper states coefficients are "systematically translated". 
            # Typically, c_i = c_high * weight, alpha_i = alpha_high * weight or similar]
            # Assuming linear mapping for simplicity as exact formula not provided:
            color = c_high[:3]
            opacity = c_high[3]
            
            gaussians.append((mu, Sigma, color, opacity))
            
    # Step 5: Image-Space Fine-Tuning
    # Optimize the parameters of gaussians to minimize rendering error
    gaussians = fine_tune_gaussians(gaussians, RF)
    
    return gaussians

def compute_canonical_gaussian_surrogate(j):
    """
    Computes the optimal Gaussian (mu, Sigma, c, alpha) that best approximates
    the canonical wavelet atom psi_j,0(x) = 2^(3j/2) * psi(2^j * x).
    This is done via statistical estimation (e.g., moment matching or least squares).
    """
    # [G: Implementation detail omitted. Requires numerical optimization 
    # to fit a Gaussian to the wavelet shape.]
    pass

def perform_dwt_3d(data, levels):
    """
    Performs 3D Discrete Wavelet Transform.
    Returns low-frequency and high-frequency coefficient arrays.
    """
    # [G: Standard DWT implementation. Returns nested arrays or flattened indices]
    pass

def transform_translation(mu, k, j):
    """
    Translates the Gaussian center based on wavelet index k and scale j.
    """
    # [G: k is the spatial translation index. The wavelet atom is shifted by k/2^j in physical space]
    return mu + k / (2**j)

def transform_scaling(Sigma, j):
    """
    Scales the Gaussian covariance based on wavelet scale j.
    """
    # [G: The wavelet atom is scaled by 2^j. Covariance scales by (2^j)^2]
    return Sigma * (2**j)**2

def compute_scaling_function_covariance(resolution_G):
    """
    Computes covariance for the scaling function phi approximation.
    """
    # [G: Approximation of the scaling function's spatial extent]
    pass

def fine_tune_gaussians(gaussians, target_RF):
    """
    Refines Gaussian parameters using image-space optimization.
    """
    # [G: Standard 3DGS optimization loop (e.g., Adam) minimizing L1/L2 loss 
    # between rendered images and target RF slices]
    pass
```

**Parameters**
*   $G$: Volumetric data resolution. The dataset is discretized into a uniform grid of resolution $G^3$ [S1, unofficial].
*   $J$: Number of decomposition levels in the wavelet transform [S1, unofficial].
*   $N$: Number of Gaussian primitives in the 3DGS representation [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Representational Gap:** There is a substantial representational gap between wavelet and Gaussian bases, making direct conversion nontrivial [S1, unofficial].
*   **Computational Complexity:** A naïve implementation of Wavelet-to-Gaussian transitions incurs cubic computational complexity ($J \times G^3$ fittings), resulting in prohibitive computational cost. The VBM method avoids this by computing only canonical kernels and deriving others analytically [S1, unofficial].

## Subtopics

### [DeSplat](12925-desplat.md)
A method for dynamic scene representation.

### [360 Gaussian Splatting](12929-360-gaussian-splatting.md)
Extends 3DGS to 360-degree panoramic scenes.

### [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)
Techniques to reduce the storage footprint of Gaussian primitives.

### [Bézier-boundary gradient approximation](13137-b-zier-boundary-gradient-approximation.md)
Approximates gradients at Gaussian boundaries using Bézier curves.

### [SOGS (Second-Order Anchor for Advanced 3D Gaussian Splatting)](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md)
Uses second-order optimization for better convergence.

### [3D Gaussian Splatting Training Loop](15139-3d-gaussian-splatting-training-loop.md)
The standard iterative training process for 3DGS.

### [Scene Representation and Geometry Constraints](15421-scene-representation-and-geometry-constraints.md)
Incorporates geometric priors into the representation.

### [Perceptual Wrapper](15438-perceptual-wrapper.md)
Uses perceptual loss functions to improve visual quality.

### [TurboGS](15697-turbogs.md)
Optimizes training speed for 3DGS.

### [Error-Guided Sparse Pixel Sampling](15708-error-guided-sparse-pixel-sampling.md)
Samples pixels based on rendering error to focus optimization.

### [Frequency Guided Resolution Scheduler](15751-frequency-guided-resolution-scheduler.md)
Adapts rendering resolution based on frequency content.

### [Inverse and Internal Rendering](15764-inverse-and-internal-rendering.md)
Renders from internal viewpoints or inverts the rendering process.

### [Neural SDF Inference via Gaussian Pulling](15776-neural-sdf-inference-via-gaussian-pulling.md)
Infers Signed Distance Fields from Gaussians.

### [Multi-View Regulated Gaussian Splatting](15783-multi-view-regulated-gaussian-splatting.md)
Uses multi-view consistency constraints.

### [Gaussian Frosting](15863-gaussian-frosting.md)
Adds a lightweight neural network on top of Gaussians for detail.

### [FisherRF](15865-fisherrf.md)
Uses Fisher Information for efficient optimization.

### [Semantic and Language Integration](15930-semantic-and-language-integration.md)
Integrates semantic labels or language queries.

### [3D Difference-of-Gaussians (3D-DoG)](16198-3d-difference-of-gaussians-3d-dog.md)
Uses DoG primitives for sharper edges.

### [Hardware and Computational Acceleration](16214-hardware-and-computational-acceleration.md)
GPU/ASIC optimizations for rendering.

### [Pruning and Density Control](16215-pruning-and-density-control.md)
Removes redundant Gaussians during training.

### [QuantumGS](16253-quantumgs.md)
Explores quantum computing for Gaussian optimization.

### [Human Gaussian Splatting](16261-human-gaussian-splatting.md)
Specialized for human avatars.

### [Dynamic and Spacetime Modeling](16342-dynamic-and-spacetime-modeling.md)
Extends 3DGS to 4D dynamic scenes.

### [Gaussian Mixture Tree Construction](16717-gaussian-mixture-tree-construction.md)
Hierarchical structure for Gaussians.

### [Closed-Form Triangle Splatting (CF-Splat)](16773-closed-form-triangle-splatting-cf-splat.md)
Uses triangles instead of Gaussians.

### [Generalizable Semantic Gaussian Splatting](16774-generalizable-semantic-gaussian-splatting.md)
Generalizes to unseen scenes with semantics.

### [Softmax-GS](16854-softmax-gs.md)
Uses softmax blending for opacity.

### [Distributed 3D Gaussian Splatting](16979-distributed-3d-gaussian-splatting.md)
Scales rendering across multiple machines.

### [Student’s t-Primitive Representation](17072-student-s-t-primitive-representation.md)
Uses t-distributions for heavier tails.

### [Hierarchical, Graph-Based, and Adaptive Structures](17106-hierarchical-graph-based-and-adaptive-structures.md)
Advanced data structures for Gaussians.

### [Anisotropic Reflective Spherical 3D Gaussian Splatting](17311-anisotropic-reflective-spherical-3d-gaussian-splatting.md)
Adds reflection models.

## Operating characteristics
*   **Time Complexity:** The precomputation step is reduced from $O(J \times G^3)$ to $O(J)$ by leveraging translation covariance and computing only canonical kernels [S1, unofficial]. The mapping step is $O(N)$ where $N$ is the number of coefficients. Fine-tuning is comparable to standard 3DGS optimization [S1, unofficial].
*   **Memory Complexity:** Stores $N$ Gaussian primitives. The Transition Bank is small ($O(J)$) [S1, unofficial].
*   **Performance:** Significantly accelerates convergence and enhances rendering quality compared to heuristic initialization methods like iVR-GS and VEG [S1, unofficial]. Enables real-time volumetric visualization [S1, unofficial].
*   **Strengths:** Principled mathematical bridge between volumetric analysis and 3DGS; exploits structural and statistical information; efficient precomputation [S1, unofficial].
*   **Limitations:** Representational gap between wavelet and Gaussian bases requires careful approximation; relies on the quality of the wavelet transform and transfer function [S1, unofficial].
*   **When to use:** Real-time visualization of large-scale volumetric data where direct volume rendering is too slow [S1, unofficial].
*   **When not to use:** Scenes where the volumetric data does not have strong structural regularities exploitable by wavelets, or when extremely high-fidelity direct volume rendering is required without approximation [S1, unofficial].

## Background
Variable Basis Mapping (VBM) was proposed in 2026 by Cai Yucheng et al. from USTC [S1, unofficial]. It addresses the challenge of real-time visualization of large-scale volumetric data, which suffers from prohibitively high computational costs with direct volume rendering and voxel-based methods [S1, unofficial]. The method builds on 3D Gaussian Splatting (3DGS) [S1, unofficial] and integrates concepts from wavelet analysis [S1, unofficial]. It is the first framework to establish a direct, mathematical bridge between volumetric analysis and 3DGS [S1, unofficial].

## Sources
[S1] Variable Basis Mapping for Real-Time Volumetric Visualization, https://doi.org/10.48550/arxiv.2601.09417, fetched 2026-10-26 (unofficial)
