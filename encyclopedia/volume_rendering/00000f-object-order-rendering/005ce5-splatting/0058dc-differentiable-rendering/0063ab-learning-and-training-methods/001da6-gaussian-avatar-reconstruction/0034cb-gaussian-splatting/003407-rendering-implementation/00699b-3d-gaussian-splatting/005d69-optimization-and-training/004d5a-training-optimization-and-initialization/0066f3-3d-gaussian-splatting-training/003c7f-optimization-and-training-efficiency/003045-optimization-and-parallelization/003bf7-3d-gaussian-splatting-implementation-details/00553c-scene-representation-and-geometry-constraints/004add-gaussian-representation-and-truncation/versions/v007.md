# Gaussian Representation and Truncation

Also called: Gaussian Splatting Primitives, 3D Gaussian Representation, Gaussian Culling

## Overview
This family defines the spatial representation of 3D Gaussian primitives in differentiable volume rendering and the mechanisms for truncating their influence to optimize computational efficiency. It solves the problem of representing complex 3D scenes with continuous, differentiable primitives while managing the high memory and compute costs associated with rendering thousands of overlapping ellipsoids. It sets itself apart from sibling methods by focusing specifically on the mathematical definition of the Gaussian kernel (including variants like Inverse Multiquadric), the logic for spatial culling (truncation thresholds), and the structural constraints applied to the primitives (such as atomization or voxel-based fixation) rather than the general rendering pipeline or color modeling.

## Shared principles
All methods in this family represent a scene element as a 3D Gaussian distribution defined by a mean position $\mu$, a covariance matrix $\Sigma$, and an opacity scalar $\omega$. The covariance matrix is typically parameterized by a scale vector $s$ and a rotation quaternion $q$, such that $\Sigma = R S S^T R^T$, where $R$ is the rotation matrix derived from $q$ and $S$ is the diagonal scaling matrix.

The rendering contribution of a Gaussian to a pixel is determined by its 2D projection. The 3D covariance $\Sigma$ is projected to 2D screen space using the camera Jacobian $J$ to obtain the 2D covariance $\Sigma'$:
$$ \Sigma' = J \Sigma J^T $$
The inverse of this matrix, $C = \Sigma'^{-1}$, is known as the conic matrix. For a pixel at screen coordinates $p$, the displacement vector is $d = p - \mu_{2D}$. The spatial decay is governed by the squared Mahalanobis distance (ellipse term):
$$ \text{ellipse} = d^T C d = C_{11} d_x^2 + C_{22} d_y^2 + 2 C_{12} d_x d_y $$
Standard 3D Gaussian Splatting uses a Gaussian kernel where the unnormalized response is $R = \exp(-0.5 \times \text{ellipse})$. The final alpha value is computed as $\alpha = \omega \cdot R$ (or $1 - \exp(-\omega R)$ for alpha blending).

Truncation is applied to cull Gaussians that contribute negligibly to a pixel. In standard implementations, this is often coupled with opacity, but variants decouple this or modify the kernel function entirely.

## Subtopics

### [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md)
A variant that replaces the Gaussian radial basis function with an inverse multiquadric function. It uses a heavier-tailed decay profile defined by $w = \frac{1}{\text{ellipse} + 1}$. To maintain visual fidelity, it applies a correction factor of $1.38$ to the covariance matrix before projection. It is chosen when standard Gaussian tails are insufficient for certain scene geometries, requiring a specific correction to prevent training instability.

### [Opacity-Independent Gaussian Truncation](10266-opacity-independent-gaussian-truncation.md)
Modifies the culling logic to evaluate the spatial Gaussian response $R$ independently of the opacity parameter $\omega$. Truncation occurs if $R < \tau_\alpha$ (typically $1/255$), regardless of $\omega$. This allows low-opacity Gaussians to receive gradients and enables aggressive truncation (e.g., at $1\sigma$) without losing fine transparency details, addressing the issue where low-opacity primitives are culled entirely in standard implementations.

### [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md)
Replaces explicit Gaussian tracing with a neural field (Multiresolution Hash Grid + MLP) that maps ray origin and direction to distance and visibility. It serves as a constant-time ray oracle for secondary rays (shadows, AO), eliminating BVH traversal. It is chosen for applications requiring efficient global illumination or shadow casting in Gaussian scenes, offering fixed memory usage independent of scene size.

### [Atomized Proliferation](12924-atomized-proliferation.md)
A densification strategy that constrains fine-detail Gaussians into "Atom Gaussians" (isotropic spheroids with uniform size $s = S^{1/3}$, where $S$ is derived from SfM point distances). It uses a warm-up splitting schedule and merges large smooth surfaces. It is chosen to enhance geometric precision and stabilize densification, preventing the drift and redundancy common in standard 3DGS.

### [Voxel-based Gaussian Splatting](13071-voxel-based-gaussian-splatting.md)
Places fixed-position 3D Gaussians at voxel grid centroids, removing spherical harmonics and optimizing only opacity, scale, and rotation. It is designed for CT volume extraction and differentiable X-ray rendering. It is chosen when accurate volume extraction is required, solving the instability and inaccuracy of applying standard 3DGS to volumetric medical data.

### [Gaussian Voxel Kernel Functions (GVKF)](13979-gaussian-voxel-kernel-functions-gvkf.md)
Uses a sparse voxel grid where feature vectors are decoded into Gaussian attributes via MLPs. It replaces discrete opacity with a continuous kernel function, enabling direct mesh extraction. It is chosen for high-fidelity surface reconstruction in open scenes with reduced memory consumption compared to explicit 3DGS.

### [Vertex-level Pruning in Mixed-Primitive Gaussian Splatting](13983-vertex-level-pruning-in-mixed-primitive-gaussian-splatting.md)
Prunes vertices from mixed primitives (triangles, lines, ellipses) based on proximity and collinearity. It converts triangles to lines or ellipses if vertices are too close or collinear. It is chosen to reduce storage overhead and simplify representation in mixed-primitive systems, prioritizing memory efficiency over maximum geometric fidelity.

### [SplatQ](14415-splatq.md)
A post-training pruning framework that partitions Gaussians into blocks and uses Quadratic Unconstrained Binary Optimization (QUBO) to select retained Gaussians. It models pairwise interactions and allocates retention quotas per block. It is chosen for high-quality compression, outperforming global Top-K methods by considering spatial and appearance balance.

### [3D Gaussian Particle Approximation of VDB Datasets](14723-3d-gaussian-particle-approximation-of-vdb-datasets.md)
Converts OpenVDB volumetric datasets into non-overlapping 3D Gaussian particles for rendering via OptiX ray marching. It leverages the non-overlapping nature of VDB leaf nodes. It is chosen for real-time rendering of large sparse volumetric data with LOD support, differing from standard overlapping Gaussian splatting.

### POG (Potentially Over-large Gaussians)
A variant discussed in [S1] where Gaussians are allowed to grow potentially over-large. Ablation results on Tanks&Temples show POG yields PSNR 26.26, SSIM 0.879, LPIPS 0.121. It is compared against Hard Gaussians (HG), which yield PSNR 26.36, SSIM 0.880, LPIPS 0.126. POG is noted to degrade PSNR significantly because it overgrows already learned areas, making them susceptible to redundant Gaussians, whereas HG avoids this [S1].

### AtomGS
Described in [S2, unofficial], AtomGS employs Atomized Proliferation to constrain ellipsoid Gaussians into uniformly-sized Atom Gaussians, emphasizing densification for fine details. It retains larger anisotropic Gaussians for backgrounds. It integrates an Edge-Aware Normal Loss using 2D edge detector weights to constrain positioning on flat surfaces. It provides precise guidance for 3D geometry optimization, balancing visual and geometric accuracy better than standard 3DGS which freely drifts and splits [S2, unofficial].

### Vertex Pruning
Defined in [S3, unofficial] as a mechanism for mixed-primitive-based learning. It prunes vertices of mixed primitives (triangles, lines) to optimize representation, as standard pruning is incompatible with mixed primitives. It supports compositional splatting and surface reconstruction learning [S3, unofficial].

### Gaussian Reduction with Learnable Mask
Described in [S4], this method reduces Gaussian count using a learnable binary mask $M$ applied to opacities and scales during optimization. It targets small-sized kernels for removal. It uses residual vector quantization for geometric attributes and hash grids for view-dependent colors. The loss is $L = L_{ren} + \lambda_m L_m + L_r + L_s$, where $L_m$ is the masking loss. The mask is defined as $M_n = sg(1[\sigma(m_n) > \epsilon] - \sigma(m_n)) + \sigma(m_n)$, with $\epsilon$ as the threshold and $\lambda_m = 5 \times 10^{-4}$ [S4].

### 3DGS Visibility Pruning
Described in [S4], this applies visibility-based pruning to 3DGS using Gaussian centroids as reference points. It computes a visibility mask by projecting centroids to 2D and sorting by depth. It removes occluded kernels during the optimization pipeline, specifically when alpha values are reset [S4].

## Background
3D Gaussian Splatting (3DGS) was introduced by Kerbl et al. in 2023 [S1, G]. It revolutionized real-time novel view synthesis by using explicit 3D Gaussians instead of implicit neural fields. The standard representation uses Gaussian kernels with opacity, position, scale, rotation, and spherical harmonics. Subsequent research has focused on optimizing this representation through various truncation, pruning, and structural modification techniques to improve efficiency, geometry, and compatibility with other data formats like VDB and CT scans.

## Sources
[S1] Pushing Rendering Boundaries: Hard Gaussian Splatting, https://arxiv.org/abs/2412.04826v1, fetched 2026-10-26
[S2] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
[S3] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30 (unofficial)
[S4] Visibility-Based Geometry Pruning of Neural Plenoptic Scene Representations, https://doi.org/10.1109/tmm.2025.3618548, fetched 2026-10-26
