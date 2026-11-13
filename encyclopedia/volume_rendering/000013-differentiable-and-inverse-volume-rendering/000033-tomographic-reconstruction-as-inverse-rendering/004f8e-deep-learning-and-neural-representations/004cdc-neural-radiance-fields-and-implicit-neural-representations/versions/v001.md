# Neural Radiance Fields and Implicit Neural Representations

Also called: INR-based Tomography, Neural Volume Rendering for CT

## Overview
This family applies Implicit Neural Representations (INRs) to tomographic reconstruction, replacing discrete voxel grids with continuous functions parameterized by neural networks [S1, unofficial]. It solves the ill-posed inverse problem of recovering 3D density fields from sparse, noisy, or limited-angle X-ray projections by leveraging the differentiability of the rendering pipeline [S1, unofficial]. Unlike traditional discrete methods, these approaches optimize network weights directly, allowing for high-resolution reconstruction from minimal data and enabling simultaneous self-calibration of geometric and photometric parameters [S1].

## Shared principles
All methods in this family rely on the differentiability of the forward projection operator $\mathcal{P}$ and its adjoint (back-projection) $\mathcal{P}^T$ [G]. The core optimization problem is generally formulated as minimizing a loss function $\mathcal{L}$ between the measured sinogram $p$ and the rendered projection $\hat{p} = \mathcal{P}(x)$ of the current volume estimate $x$:

$$
\min_x \mathcal{L}(\mathcal{P}(x), p) + \lambda R(x)
$$

where $R(x)$ is a regularizer (e.g., Total Variation, neural prior) and $\lambda$ is a weighting factor [S1, unofficial][S2].

The gradient of the loss with respect to the volume $x$ is computed via the chain rule:

$$
\frac{\partial \mathcal{L}}{\partial x} = \mathcal{P}^T \left( \frac{\partial \mathcal{L}}{\partial \hat{p}} \right)
$$

The term $\frac{\partial \mathcal{L}}{\partial \hat{p}}$ is the residual gradient in projection space (e.g., $p - \hat{p}$ for MSE), and $\mathcal{P}^T$ is the back-projection operator [G].

Key shared components include:
1.  **Forward Projection:** Ray marching through the volume representation (voxels, octrees, or neural fields) to compute line integrals of density/absorption coefficients [S1, unofficial].
2.  **Backward Projection (Adjoint):** Distributing the error from the 2D detector plane back into the 3D volume. This is mathematically the transpose of the forward operator, ensuring stable convergence in iterative methods [G].
3.  **Optimization Loop:** An iterative process (e.g., SGD, Adam) that updates the volume parameters (voxel values, neural network weights, or Gaussian parameters) to reduce the projection error [S1, unofficial][S2].

## Subtopics

### [Neural Adaptive Tomography (NeAT)](1862-neural-adaptive-tomography-neat.md)
NeAT uses a hybrid explicit-implicit structure, combining an adaptive octree with neural feature grids stored in leaf nodes. It is chosen for sparse-view or limited-angle tomography where traditional methods fail, as it simultaneously optimizes the volume, camera poses, and photometric parameters (reference image). It outperforms purely implicit methods in compute time for scattered samples.

### [HoloINR](4745-holoinr.md)
HoloINR solves near-field holotomography by explicitly modeling Fresnel propagation physics between the object and detector, rather than assuming far-field propagation. It is chosen when near-field diffraction effects are significant, mapping 3D coordinates to complex-valued scattering potentials.

### [Canonical Neural Radiance Field for X-ray Tomography](11451-canonical-neural-radiance-field-for-x-ray-tomography.md)
This method decouples the static canonical volume from the deformation field, using rich static projections to train the volume and sparse dynamic projections to learn motion. It is chosen for dynamic tomography to avoid the poor performance of directly inputting time into conventional NeRFs.

### [VolumeNeRF](13110-volumenerf.md)
VolumeNeRF reconstructs 3D CT volumes from a single 2D X-ray projection by integrating anatomical priors (Mean CT and Likelihood Images) via Projection Attention Modules (PAMs). It is chosen for single-view reconstruction to mitigate ill-posedness using structural priors.

## Background
The application of Neural Radiance Fields (NeRF) to X-ray tomography emerged from the success of NeRF in novel view synthesis for visible light [S1, unofficial]. Traditional iterative reconstruction methods (e.g., SIRT, FDK) struggle with sparse data and noise [S1]. Neural inverse rendering methods were developed to leverage the continuous nature of INRs for higher resolution and robustness in ill-posed scenarios [S1, unofficial].

## Sources
