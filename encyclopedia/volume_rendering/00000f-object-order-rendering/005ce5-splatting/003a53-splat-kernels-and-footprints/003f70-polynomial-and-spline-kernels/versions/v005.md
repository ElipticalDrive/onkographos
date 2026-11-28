# Polynomial and Spline Kernels
## Overview
Polynomial and Spline Kernels replace the standard Gaussian radial basis function in splatting with piecewise polynomial or spline functions to achieve compact support, specific decay properties, or tunable shape parameters [G]. This family solves the problem of visual artifacts (such as Mach bands or infinite support tails) and performance bottlenecks associated with Gaussian kernels by providing finite extent and analytical derivatives [G]. What sets this family apart from its siblings is the reliance on algebraic polynomial segments (quadratic, cubic, parabolic) or special functions (Beta, Dirichlet) rather than exponential decay or geometric projection adjustments alone [G].
## Shared principles
The family shares the following mechanism and mathematical structure:
1. **Compact Support:** Unlike Gaussian kernels which have infinite support, polynomial and spline kernels are defined to be zero outside a finite radius $R$ [G]. This allows for exact bounding box culling during rasterization [G].
2. **Piecewise Definition:** Most kernels in this family are defined piecewise, with different polynomial expressions for the inner region (near the center) and the outer region (near the boundary), ensuring continuity of value and derivatives at the transition point [G].
3. **Radial Symmetry:** The kernel value $W$ depends only on the radial distance $r$ (or squared Mahalanobis distance $d_M^2$) from the splat center [G].
4. **Normalization:** Kernels are scaled such that the integral over the support equals the voxel density/weight, or the maximum value is normalized to 1 [G].
5. **Covariance Transformation:** For 3D splats projected to 2D, the 3D covariance matrix $\Sigma$ is transformed into a 2D conic matrix $\mathbf{C} = \Sigma'^{-1}$ (inverse covariance) to evaluate the distance metric efficiently [G]. A correction factor is often applied to the 3D covariance before projection to account for discretization or aliasing [G].
## Subtopics

### [Quadratic Spline Splatting](8112-quadratic-spline-splatting.md)
- **What it is:** Replaces the Gaussian kernel with a quadratic spline function.
- **Distinction:** Optimized for accuracy under perspective projection, where Gaussian footprints distort significantly.
- **When to choose:** When rendering under strong perspective projection and higher accuracy is required than standard Gaussian splatting provides.
- **Link:** [Quadratic Spline Splatting](8112-quadratic-spline-splatting.md)

### [Beta Function Splatting Kernel](8602-beta-function-splatting-kernel.md)
- **What it is:** A 2D footprint function using the Beta distribution to control kernel shape.
- **Distinction:** Offers independent control over openness, height, and width via parameters $a$ and $b$, resulting in smaller footprints for equivalent visual quality compared to Gaussians.
- **When to choose:** When fine-tuning visual parameters like opacity and zoom is needed, or when faster execution via smaller footprints is desired.
- **Link:** [Beta Function Splatting Kernel](8602-beta-function-splatting-kernel.md)

### [Raised Cosine Splatting](9916-raised-cosine-splatting.md)
- **What it is:** Uses a modified raised cosine kernel with compact support.
- **Distinction:** Reduces reconstruction error for high-frequency features (edges, small textures) compared to Gaussian splatting due to specific decay properties.
- **When to choose:** When reconstructing fine geometric details and high-frequency features is critical.
- **Link:** [Raised Cosine Splatting](9916-raised-cosine-splatting.md)

### [Ideal Reconstruction Function for 3D Splatting](14537-ideal-reconstruction-function-for-3d-splatting.md)
- **What it is:** A rotationally symmetric, tri-cubic kernel designed to eliminate Mach bands.
- **Distinction:** Minimizes reconstruction variance by optimizing the transition between inner and outer polynomial segments, providing smooth overlap from any viewing direction.
- **When to choose:** When visual quality is paramount and Mach bands are unacceptable, accepting the performance cost of larger, complex footprints.
- **Link:** [Ideal Reconstruction Function for 3D Splatting](14537-ideal-reconstruction-function-for-3d-splatting.md)

### [SPH Kernel Implementation](16549-sph-kernel-implementation.md)
- **Definition:** A polymorphic interface for evaluating Smoothed Particle Hydrodynamics (SPH) kernels, supporting both CPU and GPU execution for density calculation and voxelization [G].
- **Supported Kernels:** Cubic B-spline, Quintic B-spline, Wendland C2, C4, C6, and C8 [G].
- **Features:** Each kernel has closed-form expressions for the kernel value and its derivative in 3D [G]. It supports anisotropic variants for fluid surface rendering [G].

### 3D Point Splatting with Dirichlet Kernel
- **Definition:** A method that splats weighted 3D points onto a volumetric grid using a Dirichlet kernel (Lanczos/sinc filtering) [S1, unofficial].
- **Steps:** Initialize context with grid dimensions $(n_x, n_y, n_z)$ and integer radius $ir = \lceil \text{radius} \rceil$ [S1, unofficial]. For each point $(x, y, z)$ with weight $w$, compute integer floor coordinates $i_x, i_y, i_z$ [S1, unofficial]. Compute bounding box $[x_0, x_1), [y_0, y_1), [z_0, z_1)$ by clamping $[i_x - ir, i_x + ir]$ to grid boundaries $[0, n_x]$ etc. [S1, unofficial]. Iterate over indices $i, j, k$ in the bounding box [S1, unofficial]. Add computed weight $w_z$ to grid cell $(i, j, k)$ [S1, unofficial].
- **Parameters:** `out`: 3D ndarray target grid [S1, unofficial]. `points`: $[n, 3]$ ndarray of coordinates [S1, unofficial]. `weights`: $[n]$ ndarray of weights [S1, unofficial]. `radius`: Radius of the point splat [S1, unofficial].
- **Comparison:** If a different kernel is desired after splatting, a convolution via FFT can be applied [S1, unofficial].

### Parabolic Splatting
- **Definition:** A splatting method based on a 3D parabolic function $1 - \frac{1}{\xi}(d_M)^2$ for $d_M \ge 0$ [S2].
- **Steps:** Forward pass culling: If $ellipse < 0$ or $ellipse > 9.f$, skip the splat [S2].
- **Formulas:** 2D splat function: $w = \left[ 1 - \frac{1}{\xi} (x - \boldsymbol{\mu}')^T (\Sigma'_{2 \times 2})^{-1} (x - \boldsymbol{\mu}') \right]$ [S2]. Parabolic value: $parabola = 1. - \frac{ellipse}{9}$ [S2]. Alpha: $\alpha = \min(0.99f, con\_o.w \times parabola)$ [S2]. Gradient w.r.t mean $z$: $\frac{\partial w}{\partial z} = -\frac{2}{\xi}(\Sigma'_{2 \times 2})^{-1}z$ [S2]. Gradient w.r.t inverse covariance: $\frac{\partial w}{\partial (\Sigma'_{2 \times 2})^{-1}} = -\frac{1}{\xi} z z^T$ [S2].
- **Parameters:** `correctionFactor`: Scalar multiplier for covariance matrices. Value: $1.3f$ [S2].

### Tree and Forest Modeling with KRS
- **Definition:** Hierarchical modeling of trees and forests using Kernel-Reflection Sequences (KRS) with specific kernel sets and reflection planes [S3].
- **Steps:** Define leaf geometry and thinnest twig with the first few kernels and reflections [S3]. Double geometry at every new expansion level with a reflection so that two main branches start from the same point [S3]. Add a new, thicker branch ending at the junction as the next kernel [S3]. Use toroidal capsules (torus segment and two capping spheres) for branch kernels [S3]. Compute distance function coefficients from intuitive modeling parameters: forking positions, control points for curvature, and branch width [S3]. Define the generating circle of the torus using two forking positions and a control point [S3]. Set branch width as the section radius [S3]. Place planes of reflection at forking positions with editable normals [S3]. Generate a forest by adding more reflections (identical to terrain reflections if present) with empty extra kernel sets [S3].

### Fourier Boundary Evaluation (Horner's Method)
- **Steps:** Compute tangent plane coordinates $(u_j, v_j)$ via ray-plane intersection [S4]. Compute polar coordinates: $\rho_j = \sqrt{u_j^2 + v_j^2}$ and $\theta_j = \arctan(v_j/u_j)$ [S4].
- **Code:** Horner's method accumulation for Fourier polynomial:
  $$ z_{K-1} = c_{i,K-1} $$
  $$ z_{k} = z_{k+1} \cdot w + c_{i,k} $$
  yielding $r_i(\theta) = R_i |z_0|$ [S4].
- **Complexity:** Time complexity: $K-1$ complex multiply-adds [S4].

## Background
The use of polynomial and spline kernels in volume rendering and splatting evolved from the need to address the limitations of Gaussian kernels, particularly their infinite support and fixed shape [G]. Early splatting methods relied heavily on Gaussian functions for their smoothness and ease of integration [G]. The Ideal Reconstruction Function was developed to specifically target visual artifacts like Mach bands by optimizing polynomial transitions [G]. The Dirichlet kernel approach emerged from point cloud processing libraries, offering an alternative for grid-based accumulation [S1, unofficial]. Parabolic splatting was introduced in the context of differentiable rendering (DARB-Splatting) to provide simple, differentiable alternatives to complex radial basis functions [S2]. Kernel-Reflection Sequences (KRS) apply polynomial kernel principles to hierarchical geometric modeling, particularly for natural structures like trees [S3]. Fourier-based boundary evaluation uses polynomial accumulation (Horner's method) for efficient boundary representation in radiance fields [S4]. SPH kernels, including B-splines and Wendland functions, are widely used in physics-based simulation and visualization for their smoothness and compact support [G].

## Sources
[S1] mikolalysenko/splat-points-3d, https://github.com/mikolalysenko/splat-points-3d, fetched 2026-10-26 (unofficial)
[S2] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
[S3] Kernel-Reflection Sequences, https://doi.org/10.2312/stag.20161364, fetched 2026-10-26
[S4] Fourier Splatting: Generalized Fourier encoded primitives for scalable radiance fields, https://doi.org/10.48550/arXiv.2603.19834, fetched 2026-10-30
