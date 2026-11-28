# Scene Representation Networks (SRN)


## Overview
Scene Representation Networks (SRN) represent a continuous scene as an opaque surface implicitly defined by a Multi-Layer Perceptron (MLP) that maps 3D coordinates to a feature vector [S1]. Unlike standard volume rendering methods that integrate density and color along a ray, SRN uses a Recurrent Neural Network (RNN) to march along the ray, predicting step sizes until the surface is found, then decoding the final feature vector into a single color [S1]. This approach is distinct from its siblings, such as [NeRF MLP Architecture](910-nerf-mlp-architecture.md), which models volumetric density and view-dependent color continuously, whereas SRN models a single opaque surface per ray [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A camera ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
*   **Output:** A single RGB color $\mathbf{c}$ for the point where the ray intersects the implicit surface.

**Data Structures:**
1.  **Scene MLP ($\Phi$):** Maps a 3D coordinate $\mathbf{x} \in \mathbb{R}^3$ to a feature vector $\mathbf{f} \in \mathbb{R}^k$. The exact dimension $k$ and network architecture are not specified in the primary source [S1], but it serves as the scene representation.
2.  **Step Prediction Network ($\Psi$):** An RNN (or recurrent module) that takes the current feature vector $\mathbf{f}$ and predicts the next step size $\Delta t$ along the ray.
3.  **Color Decoder ($\Omega$):** A network that maps the final feature vector $\mathbf{f}_{final}$ to the output color $\mathbf{c}$.

**Algorithm:**
The method synthesizes a view by marching along each camera ray. The process is iterative:

1.  Initialize the current position $\mathbf{p}_0 = \mathbf{o}$ and the initial feature vector $\mathbf{f}_0$ (often derived from $\Phi(\mathbf{p}_0)$ or a learned initial state).
2.  Iterate for $i = 0, 1, \dots, N-1$:
    a. Compute the feature vector at the current position: $\mathbf{f}_i = \Phi(\mathbf{p}_i)$.
    b. Use the recurrent step predictor to determine the step size: $\Delta t_i = \Psi(\mathbf{f}_i)$.
    c. Update the position along the ray: $\mathbf{p}_{i+1} = \mathbf{p}_i + \Delta t_i \cdot \mathbf{d}$.
3.  After $N$ steps (or when a convergence criterion is met), the final position $\mathbf{p}_N$ is considered the surface intersection.
4.  Decode the color from the final feature vector: $\mathbf{c} = \Omega(\mathbf{f}_N)$.

**Parameters:**
*   **Number of steps ($N$):** Not given in the sources.
*   **Feature dimension ($k$):** Not given in the sources.
*   **Network Architectures:** The specific layers, activation functions, and sizes for $\Phi$, $\Psi$, and $\Omega$ are not detailed in the provided text [S1].

**Edge Cases and Pitfalls:**
*   **Single Depth Assumption:** SRN assumes a single opaque surface per ray. It cannot handle transparent or multi-layered geometry [S1].
*   **Smoothing:** The method produces heavily smoothed geometry and texture, limiting its ability to reproduce fine details [S1].
*   **Step Prediction Stability:** The RNN must reliably predict step sizes that converge to the surface. If $\Delta t$ becomes too large, the ray may skip the surface; if too small, convergence is slow.

## Subtopics

### [Adaptively Placed Multi-Grid Scene Representation Network (APMGSRN)](11125-adaptively-placed-multi-grid-scene-representation-network-ap.md)
APMGSRN is a variant of SRN that represents continuous volumetric data using multiple spatially adaptive feature grids, dynamically allocating neural network resources to regions of high reconstruction error. Unlike sibling methods that rely on expensive octree structures or hash collisions to handle sparsity, APMGSRN uses learnable transformation matrices to migrate grid positions and scales, avoiding tree traversal and pruning while maintaining high accuracy for somewhat sparse scientific data.

### Signed Surface Approximation Network (SSAN)
SSAN is a module that creates a truncated signed distance field (TSDF) from NeRFs to enable the extraction of a unique and accurate surface and appearance [S2]. It is optimized from pretrained NeRFs, harnessing the rough 3D approximation of scene geometry learnt by the NeRF and useful priors such as rendered depths via volumetric rendering [S2].

**Layout and Formulas:**
SSAN is a neural network $\phi: \mathbb{R}^3 \rightarrow (\mathbb{R}, \mathbb{S}^2, \mathbb{R}^8)$ that takes a 3D coordinate $\mathbf{x}$ as input and predicts:
1.  Truncated signed distance approximation $\hat{t} \in [-0.1, 0.1]$.
2.  Normal $\hat{\mathbf{n}} \in \mathbb{S}^2$.
3.  8-dimensional appearance feature $\hat{\mathbf{f}} \in [0, 1]$.

$$ [\hat{t}, \hat{\mathbf{n}}, \hat{\mathbf{f}}] = \phi(\mathbf{x}) $$

The view-dependent RGB color $\hat{\mathbf{c}} \in [0, 1]$ is computed by an appearance network $\eta: (\mathbb{R}^8, \mathbb{S}^2, \mathbb{S}^2) \rightarrow \mathbb{R}^3$:

$$ \hat{\mathbf{c}} = \eta(\hat{\mathbf{f}}, \hat{\mathbf{n}}, \mathbf{d}) $$

where $\mathbf{d} \in \mathbb{S}^2$ is the viewing direction [S2].

## Operating characteristics
*   **Accuracy:** SRN captures low-frequency geometry and color variation but is unable to reproduce fine detail [S1]. It produces heavily smoothed geometry and texture [S1].
*   **Limitations:** The representational power is limited by selecting only a single depth and color per camera ray, making it unsuitable for transparent or complex multi-surface scenes [S1].
*   **Comparison:** It is a better-performing followup to DeepVoxels [S1]. It outperforms Neural Volumes (NV) and Local Light Field Fusion (LLFF) in some scenarios but lacks the fine detail reconstruction of later methods like NeRF [S1].

## Background
Scene Representation Networks (SRN) were introduced as a method to represent continuous scenes as opaque surfaces using an MLP [S1]. It is described as a better-performing followup to DeepVoxels by the same authors [S1]. The method was compared against Neural Volumes (NV) and Local Light Field Fusion (LLFF) in evaluations on real-world scenes, where it was noted for capturing low-frequency geometry but failing to reproduce fine details like fern leaves or rigging compared to other methods [S1].

## Sources
[S1] NeRF: Representing scenes as neural radiance fields for view synthesis, http://www.matthewtancik.com/nerf, fetched 2026-10-27
[S2] NeRFMeshing: Distilling Neural Radiance Fields into Geometrically-Accurate 3D Meshes, https://doi.org/10.1109/3dv62453.2024.00093, fetched 2026-10-29
