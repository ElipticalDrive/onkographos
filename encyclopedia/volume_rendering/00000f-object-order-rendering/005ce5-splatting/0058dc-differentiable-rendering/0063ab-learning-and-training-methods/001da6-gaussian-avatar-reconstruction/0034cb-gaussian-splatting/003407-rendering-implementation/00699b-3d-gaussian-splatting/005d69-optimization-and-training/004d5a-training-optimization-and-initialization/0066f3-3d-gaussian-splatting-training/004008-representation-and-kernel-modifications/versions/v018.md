# Representation and Kernel Modifications


## Overview
This family of methods modifies the fundamental 3D Gaussian primitive representation or the rendering kernel function used in 3D Gaussian Splatting to address specific limitations in shape modeling, surface reconstruction, and blur handling. Unlike sibling methods that focus on training dynamics (e.g., [Densification Strategy](6234-densification-strategy.md)) or physical parameters ([Physical Parameter Optimization](4111-physical-parameter-optimization.md)), these techniques alter the mathematical definition of the splat itself—allowing negative colors, anchoring Gaussians to a grid, enforcing tangent-plane densification, replacing the Gaussian kernel with Radial Basis Functions, or modeling spatially-varying blur.

## Shared principles
The standard 3D Gaussian Splatting representation defines a primitive $i$ by its position $\mathbf{p}_i$, covariance $\boldsymbol{\Sigma}_i$ (parameterized by rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$), opacity $\alpha_i$, and color $\mathbf{c}_i$ (often represented via spherical harmonics). The rendering process projects these 3D ellipsoids to 2D, sorts them by depth, and accumulates color using alpha blending:
$$ C(x') = \sum_{i \in N} c_i \sigma_i \prod_{j=1}^{i-1} (1 - \sigma_j) $$
where $\sigma_i = \alpha_i G'_i(x')$ and $G'_i(x')$ is the 2D Gaussian kernel evaluated at pixel $x'$ [S2, Parent].

All methods in this family retain the differentiable rendering framework of 3D Gaussian Splatting [Parent]. They modify one or more of the following components:
1.  **Primitive Attributes:** Allowing $\mathbf{c}_i$ to take negative values [S1] or constraining $\mathbf{p}_i$ to a structured grid [S2].
2.  **Kernel Function:** Replacing the exponential Gaussian kernel $e^{-d_M^2}$ with other decaying functions of the Mahalanobis distance $d_M$ [S1].
3.  **Geometric Constraints:** Enforcing that new primitives lie on the tangent plane of existing ones [S1] or wrapping primitives around surfaces [S2].
4.  **Rendering Pipeline:** Injecting differentiable blur kernels into the rasterization step [S1].

## Subtopics

### [Negative Gaussian Splatting](9152-negative-gaussian-splatting.md)
[Negative Gaussian Splatting](9152-negative-gaussian-splatting.md) extends the primitive representation by incorporating "negative Gaussians"—ellipsoids with negative color values—into the optimization process. It solves the limitation of standard Gaussian Splatting, which relies on positive-only primitives and thus requires a high number of components to model complex nonlinear structures and high-frequency color transitions. By allowing Gaussians with inverse colors to cancel each other out, NegGS provides greater flexibility in shape representation and color modulation, enabling more accurate modeling of shadows and intricate geometries with fewer total primitives.

### [Scaffold-GS](9186-scaffold-gs.md)
[Scaffold-GS](9186-scaffold-gs.md) is a Gaussian-based scene representation that utilizes anchor points to establish a hierarchical and region-aware 3D scene structure, predicting neural Gaussian attributes on-the-fly based on viewing direction and distance. Unlike vanilla 3D-GS, which allows Gaussians to drift and split freely, Scaffold-GS exploits scene structure to guide and constrain Gaussian distribution while allowing local adaptation to varying viewing angles. It achieves comparable or superior reconstruction quality to 3D-GS with significantly reduced storage requirements and real-time rendering speeds.

### [GeoGaussian](9187-geogaussian.md)
[GeoGaussian](9187-geogaussian.md) is a training optimization and initialization strategy for 3D Gaussian Splatting that improves geometric consistency and surface smoothness by densifying primitives on the tangent plane of existing Gaussians and enforcing smoothness of geometric properties between neighbors. It distinguishes itself from sibling methods like Scaffold-GS, which anchors Gaussians to a grid, and Negative Gaussian Splatting, which introduces negative opacity, by focusing exclusively on the spatial distribution and geometric coherence of the Gaussian primitives during the densification phase.

### [DARBF-Splatting](9911-darbf-splatting.md)
[DARBF-Splatting](9911-darbf-splatting.md) is a method that replaces the standard Gaussian reconstruction kernel in 3D Gaussian Splatting with a class of Decaying Anisotropic Radial Basis Functions (DARBFs). It solves the limitation of being restricted to the exponential family by allowing non-exponential, decaying functions (e.g., half-cosine, inverse multiquadric) to serve as splats. It sets itself apart from its siblings by generalizing the kernel function itself rather than modifying the primitive representation or initialization strategy, achieving faster training convergence and lower memory usage while maintaining comparable visual quality.

### [Grid-based Blur Kernel](14244-grid-based-blur-kernel.md)
[Grid-based Blur Kernel](14244-grid-based-blur-kernel.md) replaces the multi-layer perceptron (MLP) used in prior works to generate spatially-varying blur kernels with a set of learnable, indexed grid tensors. This method models spatially-varying blurriness by assigning one of $N_k$ discrete blur kernels to pixels based on a pre-computed sharpness level. It sets itself apart from sibling methods like Negative Gaussian Splatting by explicitly addressing defocus blur in the rendering pipeline through differentiable convolution, rather than modifying the primitive geometry or opacity.

### Gaussian Shell Maps
- [S1] Combine CNN-based generators with 3DGS to recreate virtual humans with sophisticated details such as clothing and accessories.

### Gaussian Wrapping
- [S2] A framework that reinterprets 3D Gaussian primitives as oriented probabilistic surface elements representing geometry only within an oriented half-space, allowing derivation of closed-form occupancy, vacancy, and normal fields for watertight mesh extraction.
- [S2] Model surface points with asymmetric primitives where the Gaussian models density decay on the outward-facing side, while the inward-facing side is considered fully occupied.
- [S2] Identify geometric gaps where the oriented surface assumption is violated and densify the representation to obtain a watertight shell of oriented primitives.
- [S2] Introduce a learnable oriented normal at each Gaussian element and define an adapted attenuation formulation to enable closed-form expressions for normal and occupancy fields.
- [S2] Introduce a novel consistency loss and a dedicated densification strategy to enforce Gaussians to wrap the entire surface by closing geometric holes.
- [S2] Modify the differentiable rasterizer to output depth as an isosurface of the continuous model.
- [S2] Use Primal Adaptive Meshing for Region-of-Interest meshing at arbitrary resolution.

## Background
The standard 3D Gaussian Splatting representation [Parent] assumes symmetric, positive-definite Gaussians with positive opacity and color. Subsequent research identified limitations in modeling thin structures, complex shadows, and surface continuity. Negative Gaussian Splatting [S1, S2] and Gaussian Wrapping [S2] address the inability of standard Gaussians to represent voids or sharp surface boundaries. Scaffold-GS [S2] and GeoGaussian [S1] address the unstructured nature of standard Gaussians, which can lead to redundancy or geometric inconsistency. DARBF-Splatting [S1] and Grid-based Blur Kernels [S1] address the mathematical rigidity of the Gaussian kernel, allowing for more flexible shape modeling and blur handling. Gaussian Shell Maps [S1] and Gaussian Wrapping [S2] specifically target human avatar reconstruction and watertight mesh extraction, respectively.

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] From Blobs to Spokes: High-Fidelity Surface Reconstruction via Oriented Gaussians, https://doi.org/10.1007/978-3-032-37095-2_4, fetched 2026-10-29
