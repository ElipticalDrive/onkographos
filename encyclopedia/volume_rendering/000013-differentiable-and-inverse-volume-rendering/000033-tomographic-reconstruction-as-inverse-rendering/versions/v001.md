# Tomographic reconstruction as inverse rendering

Also called: Differentiable tomography, Inverse volume rendering for CT

## Overview
Tomographic reconstruction as inverse rendering treats computed tomography (CT) not as a linear algebra problem to be solved via closed-form filters or static iterative solvers, but as an optimization problem where a differentiable forward renderer (the projector) is minimized against measured projection data. This approach unifies classical algebraic reconstruction techniques (ART, SIRT) with modern neural representations and compressive sensing priors under a single gradient-based framework [S1, unofficial][S2]. It distinguishes itself from [Differentiable volume rendering](43-differentiable-and-inverse-volume-rendering.md) by focusing specifically on the tomographic geometry (parallel/fan/cone beam) and the reconstruction of density/absorption fields from sparse or noisy sinograms, and from [Radiance-field optimization](44-radiance-field-optimization.md) by explicitly modeling the X-ray physics (Beer-Lambert law) rather than just RGB appearance [S1, unofficial][S2].

## Shared principles
All methods in this family rely on the differentiability of the forward projection operator $\mathcal{P}$ and its adjoint (back-projection) $\mathcal{P}^T$. The core optimization problem is generally formulated as minimizing a loss function $\mathcal{L}$ between the measured sinogram $p$ and the rendered projection $\hat{p} = \mathcal{P}(x)$ of the current volume estimate $x$:

$$
\min_x \mathcal{L}(\mathcal{P}(x), p) + \lambda R(x)
$$

where $R(x)$ is a regularizer (e.g., Total Variation, neural prior) and $\lambda$ is a weighting factor [S1, unofficial][S2].

The gradient of the loss with respect to the volume $x$ is computed via the chain rule:

$$
\frac{\partial \mathcal{L}}{\partial x} = \mathcal{P}^T \left( \frac{\partial \mathcal{L}}{\partial \hat{p}} \right)
$$

The term $\frac{\partial \mathcal{L}}{\partial \hat{p}}$ is the residual gradient in projection space (e.g., $p - \hat{p}$ for MSE), and $\mathcal{P}^T$ is the back-projection operator [S2][S3, unofficial].

Key shared components include:
1.  **Forward Projection:** Ray marching through the volume representation (voxels, octrees, or neural fields) to compute line integrals of density/absorption coefficients [S1, unofficial][S2].
2.  **Backward Projection (Adjoint):** Distributing the error from the 2D detector plane back into the 3D volume. This is mathematically the transpose of the forward operator, ensuring stable convergence in iterative methods [S2][S3, unofficial].
3.  **Optimization Loop:** An iterative process (e.g., SGD, Adam) that updates the volume parameters (voxel values, neural network weights, or Gaussian parameters) to reduce the projection error [S1, unofficial][S2].

## Subtopics

### [Deep Image Prior Reconstruction](1221-deep-image-prior-reconstruction.md)
Uses a neural network (e.g., U-Net) as an implicit regularizer. Instead of optimizing voxel values directly, it optimizes the network weights $\theta$ and a latent variable $z$ to fit the sinogram data. It distinguishes itself by leveraging the architectural bias of CNNs toward natural image structures, allowing reconstruction from very sparse or noisy data without explicit priors.

### [DuDoTrans](1409-dudotrans.md)
A dual-domain transformer architecture for sparse-view CT. It uses Window-based Multi-Head Self-Attention (W-MSA) to reduce computational complexity from $O(N^2)$ to $O(N \cdot W_s^2)$ while maintaining long-range dependencies. It distinguishes itself from standard self-attention by restricting global context to local windows, making it efficient for high-resolution volumetric data.

### [Neural Adaptive Tomography (NeAT)](1862-neural-adaptive-tomography-neat.md)
An adaptive, hierarchical neural rendering pipeline for multi-view inverse rendering. It combines an explicit octree structure with neural features in leaf nodes. It distinguishes itself from purely implicit representations (like NeRF) by using an explicit hierarchy optimized via differentiable volume rendering, which is more efficient for tomographic tasks with widely scattered samples.

### [Compressive Volume Rendering Recovery Methods](1915-compressive-volume-rendering-recovery-methods.md)
Reconstructs volumes from sparse pixel subsets by treating the problem as image inpainting with sparsity priors (e.g., wavelet sparsity, Total Variation). It distinguishes itself from iterative algebraic methods (like SIRT) by using non-linear optimization of sparsity-promoting regularizers rather than simple linear updates.

### [Compressive Gradient Rendering](1948-compressive-gradient-rendering.md)
A compressive sensing technique that renders discrete gradient components (horizontal and vertical) instead of pixel intensities. It exploits sparsity in the Fourier domain and reconstructs the image using a Poisson solver. It distinguishes itself by operating in the gradient domain, allowing reconstruction from fewer samples than the Nyquist rate.

### [Back-projection](2450-back-projection.md)
The mathematical adjoint (transpose) of the forward projection operator. It accumulates 2D projection data into a 3D reconstruction volume. It distinguishes itself as the core gradient computation step in differentiable tomographic reconstruction, providing the exact transpose required for stable iterative optimization, unlike direct methods like FDK which may suffer from artifacts with limited projections.

### [Differentiable CT Execution Model](3066-differentiable-ct-execution-model.md)
A software framework providing per-view CT geometry, cell-constant point-ray Siddon projection, a matched adjoint, and first-order geometry/surface derivatives. It distinguishes itself by offering a complete differentiable execution model with spatial, view, and pixel block partitions for multi-GPU/CPU/disk scalability.

### [SIRT (Simultaneous Iterative Reconstruction Technique)](3101-sirt-simultaneous-iterative-reconstruction-technique.md)
An iterative method that normalizes ray residuals by total intersected length and backprojected updates by accumulated voxel sensitivity. It distinguishes itself by enforcing nonnegativity and providing stable convergence for algebraic reconstruction.

### [General Numerical Optimization Framework for Tomography](3251-general-numerical-optimization-framework-for-tomography.md)
A general model for solving inverse problems by minimizing a data fidelity term plus a regularizer. It distinguishes itself as a flexible framework that can incorporate various priors and optimizers beyond specific algorithms like SIRT or FDK.

### [FastFDK](3315-fastfdk.md)
A CPU-based implementation of the Feldkamp-Davis-Kress (FDK) algorithm, a filtered backprojection method for cone-beam CT. It distinguishes itself as a direct, non-iterative closed-form solution, intended for accuracy validation rather than iterative optimization.

### [Voxel Coloring](4418-voxel-coloring.md)
A volumetric reconstruction algorithm using color consistency to distinguish object surface voxels from background. It distinguishes itself by starting with an opaque volume and carving inconsistent voxels, rather than integrating density along rays.

### [HoloINR](4745-holoinr.md)
Solves the coupled inverse problem of holotomography (holography and tomography) using Implicit Neural Representations (INRs) for near-field setups. It distinguishes itself by modeling Fresnel propagation physics between object and detector, unlike far-field parallel/fan-beam assumptions.

### [EmissivityRenderer](4811-emissivityrenderer.md)
A differentiable renderer for multi-line neural tomography recovering 3-D electron density and temperature from spectral-intensity images. It distinguishes itself by performing Line-of-Sight (LOS) integration of emissivity, specific to plasma physics applications.

### [B-mode Image Reconstruction](5511-b-mode-image-reconstruction.md)
Not given in the sources.

### [R2-Gaussian](6090-r2-gaussian.md)
Adapts 3D Gaussian Splatting (3DGS) for tomography by addressing integration bias. It uses a local affine approximation to map 3D Gaussians to 2D Gaussians on the detector for alpha compositing. It distinguishes itself by enabling efficient, differentiable rendering of Gaussian primitives for CT reconstruction.

### [Differential Rendering for CT Reconstruction](6462-differential-rendering-for-ct-reconstruction.md)
An iterative method optimizing 3D volume parameters (absorption coefficient $\sigma$) by minimizing MSE between differentially rendered images and reference projections. It distinguishes itself by explicitly using the gradient of the rendering equation with respect to volume parameters for optimization.

### Density Reconstruction (Absorption-Only)
Reconstruction of a 3D density field from images using an optical absorption-only model, reducing the problem to a linear optimization problem solvable via DiffDVR [S2]. It uses multi-resolution optimization, starting at $32^3$ and doubling resolution every 10 iterations, performing 50 iterations at the highest resolution ($256^3$) [S2]. The loss function uses a smoothing prior on volume densities with $\lambda = 0.5$ [S2]. Parameters include 64 uniformly sampled views on a surrounding sphere, $512^2$ image resolution, Adam optimizer with batch size 8 and learning rate 0.3, and a step size of 0.2 voxels [S2]. Performance comparison for a human skull dataset shows ASTRA takes 53 seconds, DiffDVR takes ~12 minutes, and Mitsuba takes multiple hours [S2].

### Space-Time Image Formation Model
A model representing a continuously deforming 3D object as a 4D density function $h(x, y, z, t)$, discretized into $N_k$ frames, each represented by a voxel grid of $N_v$ voxels, assuming each frame is approximately static for reconstruction [S3, unofficial]. The tomography problem is expressed as $A \cdot f = p$, where $A$ is a block diagonal matrix with blocks $A_t \in \mathbb{R}^{(N_\theta M \times N_v)}$ modeling the Radon transform for $N_\theta$ projections contributing to frame $t$, $f$ is the vector of 3D volumes, and $p$ is the vector of measured sinogram data $p_t \in \mathbb{R}^{(N_\theta M)}$ [S3, unofficial]. Parameters: $N_k \le N_p$ (number of reconstructed frames), $N_\theta = N_p/N_k$ (projections per frame), $M$ (pixels per projection) [S3, unofficial]. The 4D density volume is described by a vector $f \in \mathbb{R}^{(N_v N_k)}$, composed of column vectors $f_t \in \mathbb{R}^{N_v}$ for each time step $t$, where $f = \{f_0, f_1, ..., f_{N_k-1}\}$ [S3, unofficial].

### Laplacian Projection
A physics-based tomographic reconstruction method employing Laplacian regularization for sparse view reconstruction, representing the state-of-the-art in physics-based smoke reconstruction [S4]. Parameters: Multi-view variant uses 16 views; single-view variant uses only one input view [S4].

### CPU Reference Reconstruction
A CPU-based implementation of CT reconstruction used as a ground truth reference for validating GPU implementations [S5, unofficial]. Parameters: Input includes projections (`std::vector<float>`), P (number of projections), W (width), H (height), NX (volume width), NY (volume height/depth), voxelSize (float), pixelSize (float), sdd (source-to-detector distance), sod (source-to-origin distance) [S5, unofficial]. Output is `std::vector<float>` representing the reconstructed volume [S5, unofficial].

## Background
Tomographic reconstruction has traditionally been dominated by analytical methods like Filtered Back Projection (FBP) and algebraic methods like ART and SIRT [S1, unofficial][S2]. The advent of differentiable rendering has allowed these problems to be framed as optimization problems, enabling the use of gradient-based optimizers and complex neural priors [S2]. Key developments include the use of DiffDVR for density reconstruction [S2], the application of Transformers for sparse-view reconstruction (DuDoTrans) [S1, unofficial], and the integration of neural implicit representations (NeAT, HoloINR) for adaptive and holographic tomography [S1, unofficial]. The field continues to evolve with hybrid methods combining explicit structures (octrees, Gaussians) with neural features for efficient and high-quality reconstruction [S1, unofficial][S4].

## Sources
[S1] ApaarRaina/DiffTomo, https://github.com/ApaarRaina/DiffTomo, fetched 2026-10-27 (unofficial)
[S2] Differentiable Direct Volume Rendering, https://arxiv.org/abs/2107.12672v1, fetched 2026-10-26
[S3] Space-Time Tomographic Reconstruction of Deforming Objects, https://vccimaging.org/Publications/Zang2020Thesis/Zang2020Thesis.pdf, fetched 2026-10-30 (unofficial)
[S4] A High-Efficiency Physics-Based Differentiable Renderer for Smoke Reconstruction, https://doi.org/10.26599/cvm.2026.9450546, fetched 2026-10-27
[S5] hardik0001/ct-reconstruction-gpu, https://github.com/hardik0001/ct-reconstruction-gpu, fetched 2026-10-27 (unofficial)
