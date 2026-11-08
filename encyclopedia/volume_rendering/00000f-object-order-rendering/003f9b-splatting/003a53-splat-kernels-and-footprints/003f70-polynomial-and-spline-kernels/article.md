# Polynomial and Spline Kernels
## Overview
This family defines splatting kernels using piecewise polynomial functions and spline basis functions to approximate volumetric density. It addresses the need for kernels with compact support and specific smoothness properties (continuity of derivatives) that differ from the standard Gaussian or EWA splats. It sets itself apart from [EWA Splatting](1069-ewa-splatting.md) by focusing on the algebraic form of the radial basis function rather than just the covariance-based footprint approximation, and from [Splat Extent and Projection](1792-splat-extent-and-projection.md) by defining the internal weight decay function.
## Shared principles
The splatting family shares the fundamental mechanism of treating volumetric data as a set of geometric or parametric primitives that are transformed and rasterized [S1, S3, S4, unofficial].

1.  **Primitive Representation:** The volume is decomposed into primitives. These may be explicit geometric shapes (cubes, cells, splats) or implicit functions sampled via texture hardware [S1, S3, S4, unofficial]. Each primitive $k$ is defined by its position $\mathbf{p}_k$, density/extinction coefficient $g_k$ or $\rho_k$, and a reconstruction kernel $h$ (typically a Gaussian) [S1, unofficial][S6][S10, unofficial].
2.  **Transformation:** Primitives are transformed from volume space to screen space using the Model-View-Projection (MVP) matrix or specific factorizations (e.g., shear-warp) [S2, S3, S4, unofficial]. For perspective projections, the kernel shape may be distorted (e.g., EWA splatting) to account for foreshortening [S6][S10, unofficial].
3.  **Rasterization:** The transformed primitives are projected onto the 2D image plane. This step leverages hardware-accelerated scan-conversion, depth testing, and blending [S1, S2, S3, S4, unofficial]. The contribution of a voxel $D$ to a screen point $(x,y)$ is determined by the footprint function $F(x,y)$, which is the integral of the 3D kernel along the view direction [S2][S6][S10, unofficial]:
    $$ F(x,y) = \int h(x-x_D, y-y_D, z) \, dz $$
    The intensity written to the pixel is modulated by the voxel density:
    $$ P(x,y) = \text{weight}(x,y) \cdot \rho(D) $$
    where $\text{weight}(x,y) = F(x-x_D, y-y_D)$ [S10, unofficial].
4.  **Compositing:** Contributions are accumulated to form the final image. This is typically done using alpha blending in back-to-front or front-to-back order to handle transparency correctly [S1, unofficial][S2, S3][S7, book]. Splatting compositing is performed along sheets (all pixels at once for a set of voxels), whereas ray-casting composites along rays (one sample at a time) [S10, unofficial].
5.  **Hardware Acceleration:** Most methods in this family rely on GPU rasterization pipelines for performance, enabling real-time interactivity [S1, unofficial][S2, S3].

## Subtopics

### [Quadratic Spline Splatting](8112-quadratic-spline-splatting.md)
Uses quadratic spline basis functions for kernel definition.

### [Beta Function Splatting Kernel](8602-beta-function-splatting-kernel.md)
A 2D kernel function used for splatting that replaces the Normal distribution. It uses a Beta function to control footprint shape, offering smaller footprints and easier tuning for visual parameters like opacity and zoom.

### [Raised Cosine Splatting](9916-raised-cosine-splatting.md)
A splatting technique using a raised cosine function as the radial basis function, requiring a correction factor to scale the 3D covariance matrix.

### 3D Point Splatting with Dirichlet Kernel
- **Definition:** A method that splats a collection of weighted 3D points onto a volumetric grid using a Dirichlet kernel (described in the text as Lanczos/sinc filtering) [S1, unofficial].
- **Step:**
    1. Initialize the computation context with the output grid dimensions $(n_x, n_y, n_z)$ and the integer radius $ir = \lceil \text{radius} \rceil$ [S1, unofficial].
    2. For each point $(x, y, z)$ with weight $w$, compute the integer floor coordinates $i_x = \lfloor x \rfloor$, $i_y = \lfloor y \rfloor$, $i_z = \lfloor z \rfloor$ [S1, unofficial].
    3. Compute the bounding box $[x_0, x_1)$, $[y_0, y_1)$, $[z_0, z_1)$ for the splat by clamping $[i_x - ir, i_x + ir]$ to the grid boundaries $[0, n_x]$, $[0, n_y]$, $[0, n_z]$ [S1, unofficial].
    4. Iterate over the bounding box indices $i, j, k$ [S1, unofficial].
    5. Add the computed weight $w_z$ to the grid cell at $(i, j, k)$ [S1, unofficial].
- **Parameter:**
    - `out`: a 3D ndarray representing the target volumetric grid [S1, unofficial].
    - `points`: a $[n, 3]$ shape ndarray encoding the x/y/z grid coordinates of the points to splat [S1, unofficial].
    - `weights`: a $[n]$ shape ndarray containing the weight for each point [S1, unofficial].
    - `radius`: the radius of the point splat to draw, used to determine the bounding box of the kernel support [S1, unofficial].
- **Comparison:** The text notes that if a different kernel is desired after splatting, a convolution via FFT can be applied [S1, unofficial].

### Parabolic Splatting
- **Definition:** A splatting method based on a 3D parabolic function, defined as $1 - \frac{1}{\xi}(d_M)^2$ for $d_M \geq 0$ [S2].
- **Step:** Forward pass culling: If $ellipse < 0$ or $ellipse > 9.f$, the splat is skipped [S2].
- **Formula:**
    - The 2D splat function is: $w = \left[ 1 - \frac{1}{\xi} (x - \boldsymbol{\mu}')^T (\Sigma'_{2 \times 2})^{-1} (x - \boldsymbol{\mu}') \right]$ [S2]. Here $z = (x - \mu')$.
    - The parabolic value is calculated as: $parabola = 1. - \frac{ellipse}{9}$ [S2]. The alpha is $\alpha = \min(0.99f, con\_o.w \times parabola)$ [S2].
    - Backward pass gradients for mean: $\frac{\partial w}{\partial z} = -\frac{2}{\xi}(\Sigma'_{2 \times 2})^{-1}z$ [S2]. In code, $dP_{d\delta x} = (con\_o.x \times d.x + con\_o.y \times d.y) \times \frac{-2}{9}$ [S2].
    - Backward pass gradients for covariance: $\frac{\partial w}{\partial (\Sigma'_{2 \times 2})^{-1}} = -\frac{1}{\xi} z z^T$ [S2]. In code, gradients are scaled by $\frac{-1}{9}$ [S2].
- **Parameter:**
    - `correctionFactor`: Scalar multiplier for covariance matrices. Value: $1.3f$ [S2].
- **Code:**
    - Forward propagation core loop snippet:
      ```python
      correctionFactor = 1.3f
      cov *= correctionFactor
      ellipse = (con_o.x * d.x * d.x + con_o.z * d.y * d.y + 2.f * con_o.y * d.x * d.y)
      if ellipse < 0 or ellipse > 9.f:
          continue
      parabola = 1.0 - ellipse / 9.0
      alpha = min(0.99f, con_o.w * parabola)
      ```
    - Backward propagation gradient updates for mean and covariance:
      ```python
      correctionFactor = 1.3f
      cov2D *= correctionFactor
      # Gradients of loss w.r.t. entries of 2D covariance matrix,
      # given gradients of loss w.r.t. conic matrix (inverse covariance matrix).
      # e.g., dL / da = dL / d_conic_a * d_conic_a / d_a
      dL_da = denom2inv * (-c * c * dL_dconic.x + 2 * b * c * dL_dconic.y + (denom - a * c) * dL_dconic.z)
      dL_dc = denom2inv * (-a * a * dL_dconic.z + 2 * a * b * dL_dconic.y + (denom - a * c) * dL_dconic.x)
      dL_db = ...
      ```

### Tree and Forest Modeling with KRS
- **Definition:** Hierarchical modeling of trees and forests using KRS with specific kernel sets and reflection planes [S3].
- **Step:**
    1. Define leaf geometry and thinnest twig with the first few kernels and reflections [S3].
    2. Double geometry at every new expansion level with a reflection placed so that two main branches start from the same point [S3].
    3. Add a new, thicker branch ending at the junction as the next kernel [S3].
    4. Use toroidal capsules (torus segment and two capping spheres) for branch kernels [S3].
    5. Compute distance function coefficients from intuitive modeling parameters: forking positions, control points for curvature, and branch width [S3].
    6. Define the generating circle of the torus using two forking positions and a control point [S3].
    7. Set branch width as the section radius [S3].
    8. Place planes of reflection at forking positions with editable normals [S3].
    9. Generate a forest by adding more reflections (identical to terrain reflections if present) with empty extra kernel sets [S3].

## Background
Kernel-Reflection Sequences (KRS) were introduced in 2016 [S3]. They are used for hierarchical modeling of natural features like trees, forests, coral, terrain, and rocks [S3]. The method leverages the self-similar nature of these structures, where individual kernels become indistinguishable at large scales, resembling a 3D equivalent of a Lévy C-curve [S3].

## Sources
[S1] mikolalysenko/splat-points-3d, https://github.com/mikolalysenko/splat-points-3d, fetched 2026-10-26 (unofficial)
[S2] DARB-Splatting: Generalizing Splatting with Decaying Anisotropic Radial Basis Functions, https://doi.org/10.1109/WACV61042.2026.00438, fetched 2026-10-30
[S3] Kernel-Reflection Sequences, https://doi.org/10.2312/stag.20161364, fetched 2026-10-26
