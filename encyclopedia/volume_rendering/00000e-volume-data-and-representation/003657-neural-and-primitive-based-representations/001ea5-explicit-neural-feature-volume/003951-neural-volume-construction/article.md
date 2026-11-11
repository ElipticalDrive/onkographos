# Neural Volume Construction

Also called: Sparse 3D ConvNet Feature Aggregation, Sparse Tensor Construction

## Overview
Neural Volume Construction is the process of aggregating sparse, initialized noisy primitives into a dense, structured 3D latent feature volume aligned with the world coordinate system [S1]. It solves the problem of integrating local and global geometric information from scattered primitives into a coherent representation that can be queried for decoding geometry and appearance parameters [S1]. It differs from sibling methods like the Decoder Network by providing the underlying spatial data structure, and from TensoRF by using a full 3D convolutional aggregation rather than low-rank factorization [S1][S9759-tensorf-vector-matrix-decomposition.md].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of initialized noisy 3D primitives $\mathcal{P}$ (e.g., Gaussian centers from a point cloud) [S1].
*   **Output:** A neural feature volume $\mathbf{F} \in \mathbb{R}^{H \times W \times D \times C}$, where $H \times W \times D$ is the spatial resolution and $C$ is the feature dimension [S1].

**Data Structures:**
*   **Sparse Tensor:** The input primitives $\mathcal{P}$ are stored as a sparse tensor to handle irregular distribution efficiently [S1].
*   **Feature Volume:** A dense 3D tensor $\mathbf{F}$ aligned with the world coordinate system [S1].

**Algorithm:**
1.  **Sparse Tensor Construction:** Construct a sparse tensor from the initialized noisy primitives $\mathcal{P}$ [S1].
2.  **Aggregation via Sparse 3D ConvNet:** Employ a sparse U-Net style 3D ConvNet $\psi^{3D}$ to aggregate the features [S1].
    *   The network follows an encoder-decoder structure [S1].
    *   A bottleneck with low spatial dimension is used to compress global context [S1].
    *   Skip connections are added to preserve both local and global information [S1].
3.  **Feature Volume Generation:** Compute the output feature volume $\mathbf{F}$ using the aggregation function:
    $$ \mathbf{F} = \psi^{3D}(\mathcal{P}) $$
    where $\mathbf{F}$ is aligned with the world coordinate system [S1].

**Parameters:**
*   **Spatial Resolution ($H \times W \times D$):** Not given in the sources.
*   **Feature Dimension ($C$):** Not given in the sources.
*   **Network Architecture:** Sparse U-Net style 3D ConvNet [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Trilinear interpolation is used to query the latent feature from $\mathbf{F}$ for each primitive [S1].
*   **Boundaries:** The output volume is aligned with the world coordinate system, implying the spatial extent must cover the scene bounds [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_neural_volume(primitives: List[Primitive], 
                            resolution: Tuple[int, int, int], 
                            feature_dim: int) -> Tensor:
    """
    Constructs a neural feature volume from noisy primitives using a sparse 3D ConvNet.
    
    Args:
        primitives: List of initialized noisy primitives (e.g., Gaussian centers).
        resolution: Spatial resolution (H, W, D) of the output feature volume.
        feature_dim: Dimensionality C of the feature vectors.
        
    Returns:
        F: A dense tensor of shape (H, W, D, C) aligned with the world coordinate system.
    """
    # Step 1: Construct sparse tensor from primitives
    # [G] Implementation detail: Map primitive positions to sparse tensor indices.
    sparse_tensor = create_sparse_tensor(primitives)
    
    # Step 2: Define Sparse U-Net style 3D ConvNet
    # [G] Implementation detail: Encoder with downsampling, bottleneck, decoder with upsampling, skip connections.
    conv_net = SparseUNet3D(
        input_channels=primitives.feature_channels,
        output_channels=feature_dim,
        resolution=resolution
    )
    
    # Step 3: Aggregate features
    # Formula: F = psi^3D(P)
    feature_volume = conv_net(sparse_tensor)
    
    return feature_volume
```

## Subtopics

### [Neural Inpainting for Holes](8918-neural-inpainting-for-holes.md)
Neural Inpainting for Holes is a method to resolve the "one-to-many" geometric ambiguity that arises during forward warping of voxel grids in dynamic volume rendering. When a scene expands, multiple source voxels map to a single target voxel, leaving empty "holes" in the target grid if warping is performed naively. This method employs a neural network to predict and fill these missing voxel values, enabling the reconstruction of a complete, high-resolution voxel grid from a lower-resolution warped input. It is distinct from sibling methods like Sparse Domain Decomposition and TensorBase by specifically addressing temporal warping artifacts through a dedicated 3D convolutional architecture rather than static representation or spatial partitioning.

### [Sparse Domain Decomposition](9993-sparse-domain-decomposition.md)
Sparse domain decomposition is a technique for representing large-scale volumetric scenes by partitioning the spatial domain into fixed-size subdomains, each modeled by a dedicated neural network (expert). It solves the performance degradation of single-network approaches in large domains, where normalized input coordinates $[0, 1]$ become incoherently clustered, causing poor feature mapping. Unlike single-model approaches, it enables parallel training and inference without inter-process communication, scaling accuracy and speed for massive datasets. It differs from standard Mixture-of-Experts (MoE) by using a fixed, non-learnable gate function rather than a learned gating mechanism.

### [TensorBase](10906-tensorbase.md)
TensorBase is a base class for tensor-based volume rendering that manages the axis-aligned bounding box (AABB), grid size, Singular Value Decomposition (SVD) volume initialization, and rendering module selection. It serves as the foundational infrastructure for neural volume construction, handling the geometric mapping from ray space to the feature grid and selecting the appropriate shading function (e.g., MLP, Spherical Harmonics) based on configuration. It distinguishes itself from sibling methods like Neural Inpainting for Holes and Sparse Domain Decomposition by providing the core tensor management and ray sampling logic upon which those specialized techniques build.

### Precomputed Sampling Features
Pre-calculated transmission and phase features stored in a table to accelerate neural network input generation [S2, unofficial]. Perform 4129 template samplings for each volume of medium using a uniform sampling strategy [S2, unofficial]. Generate 254 sampled transmission features and 254 phase transmission features per sample [S2, unofficial]. For phase features: Find volume phase of all template points using formula 6 (MHG/HG), set all possible angles between incident light and camera, find phase features using formulas 2, 3, 4, 5, pre-calculate all sampling points, and save to table [S2, unofficial]. For transmission features: Obtain graded density field by downsampling density field to get graded transmission field. Generate transmission features through 3D-CNN. Obtain sampling transmission features through sampling template. Pre-calculate and store in table [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Integrates local and global information in different network layers for faithful reconstructions [S1]. The 3D latent representation helps model prior knowledge across scenes [S1].
*   **Limitations:** Not given in the sources.
*   **When to Use:** When a generalizable 3D latent neural volume is needed for modeling prior knowledge across scenes [S1].
*   **When Not to Use:** Not given in the sources.

## Background
Originating from EVolSplat, this method was introduced in 2025 [S1]. It draws inspiration from previous works on generalizable 3D latent neural volumes [S1]. It is used in efficient volume-based Gaussian Splatting for urban view synthesis [S1].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S2] A real-time rendering method for high albedo anisotropic materials with multiple scattering, https://doi.org/10.48550/arxiv.2401.14051, fetched 2026-10-26 (unofficial)
