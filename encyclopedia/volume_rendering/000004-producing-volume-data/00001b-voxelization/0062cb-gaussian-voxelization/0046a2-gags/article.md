# GaGS
Also called: Gaussian Voxelization with Sparse Convolution for Deformation

## Overview
GaGS is a method for learning spatial-temporal deformation of 3D Gaussians in dynamic scenes by voxelizing Gaussian distributions and extracting geometry-aware features via sparse convolutions [S1][S2, unofficial]. It sets itself apart from sibling methods by explicitly combining positional features (via MLP on point clouds) with local geometric features (via 3D U-Net on voxels) to predict deformation, rather than relying solely on implicit deformation fields or 4D Gaussian formulations [S1][S2, unofficial].

## How it works
**Inputs:** A set of 3D Gaussians $G_k$ (initialized from the first frame), each defined by mean $\mu_k$, covariance $\Sigma_k$, opacity $\alpha_k$, and features $F_k$. A target timestamp $t$ for deformation prediction.
**Outputs:** Deformed 3D Gaussians at timestamp $t$, specifically updated positions and rotations.

**Data Structures:**
1.  **Point Cloud:** A list of 3D coordinates $\{\mu_k\}$ representing the Gaussian centers.
2.  **Voxel Grid:** A sparse 3D grid where Gaussians are discretized. The exact voxelization resolution and sparsification threshold are not given in the sources [G: Assume standard sparse voxel grid implementation with coordinate hashing].
3.  **Feature Vectors:**
    *   Positional features $F_{pos}$: Output of the positional MLP.
    *   Geometric features $F_{geo}$: Output of the 3D U-Net.
    *   Geometry-aware features $F_{geom}$: Concatenation of $F_{pos}$ and $F_{geo}$.
4.  **Rotation Representation:** 6D rotation representation (instead of quaternions) for each Gaussian.

**Steps:**
1.  **Initialization:** Initialize 3D Gaussians from the first frame of the dynamic scene [S2, unofficial].
2.  **Branch 1 (Positional Features):**
    *   Extract the 3D coordinates $\{\mu_k\}$ of the Gaussians.
    *   Treat these coordinates as a point cloud.
    *   Pass the point cloud through a Multi-Layer Perceptron (MLP) to extract positional features $F_{pos}$ [S2, unofficial].
3.  **Branch 2 (Geometric Features):**
    *   Voxelize the point cloud of Gaussian centers into a 3D voxel grid [S1][S2, unofficial].
    *   Extract local geometric features from the voxel grid using a 3D U-Net to produce $F_{geo}$ [S2, unofficial].
4.  **Feature Fusion:**
    *   Concatenate $F_{pos}$ and $F_{geo}$ to form the geometry-aware features $F_{geom}$ [S2, unofficial].
5.  **Deformation Prediction:**
    *   Feed $F_{geom}$ into an MLP to predict the deformation parameters for the Gaussians at the target timestamp $t$ [S2, unofficial].
    *   The MLP outputs updates for position and rotation.
6.  **Rotation Handling:**
    *   Convert the predicted rotation updates from the 6D representation back to the standard representation (e.g., quaternion) for the final Gaussian state [S2, unofficial].

**Parameters:**
*   **Rotation Representation:** 6D rotation representation is used internally to facilitate smooth learning of rotation variations [S2, unofficial].
*   **Network Architectures:** MLPs for positional features and deformation prediction; 3D U-Net for geometric feature extraction. Specific layer counts, hidden dimensions, and kernel sizes are not given in the sources [G: Assume standard U-Net and MLP configurations].

**Edge Cases and Pitfalls:**
*   **Rotation Smoothness:** Using quaternions directly can lead to learning difficulties due to non-linearity and singularities. The 6D representation mitigates this [S2, unofficial].
*   **Voxelization Loss:** Discretizing continuous Gaussians into voxels may lose fine geometric details. The 3D U-Net aims to capture local geometry to compensate [S1][S2, unofficial].
*   **Timestamp Dependency:** The deformation prediction is conditioned on the target timestamp $t$. The MLP must generalize across time [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians $N$, the voxel grid resolution, and the depth of the MLPs and 3D U-Net. Sparse convolution reduces complexity compared to dense 3D convolutions. Not given in the sources.
*   **Memory Complexity:** Proportional to the number of non-empty voxels and the feature dimensions. Sparse voxel storage helps manage memory. Not given in the sources.
*   **Strengths:** Explicitly captures local geometric information via 3D U-Net, which may improve deformation accuracy in complex dynamic scenes. The 6D rotation representation aids in learning smooth rotations.
*   **Limitations:** Requires voxelization, which introduces discretization artifacts. The two-branch architecture increases computational overhead compared to single-branch methods. Not given in the sources.
*   **When to use:** When accurate deformation learning is critical and local geometric context is important.
*   **When not to use:** When computational resources are limited and simpler implicit deformation fields suffice.

## Background
GaGS was proposed by [110] (cited in [S2, unofficial]) and [161] (cited in [S1]) as a method to handle dynamic scenes by learning deformation in a canonical space. It is part of the broader effort to extend 3D Gaussian Splatting to 4D (space-time) representations [S1]. The method addresses the limitation of static 3D Gaussians by introducing a geometry-aware feature extraction network that leverages sparse convolutions [S1][S2, unofficial].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] Dynamic Scene Reconstruction: Recent Advance in Real-time Rendering and Streaming, https://doi.org/10.48550/arXiv.2503.08166, fetched 2026-10-29 (unofficial)
