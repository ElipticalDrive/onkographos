# GauFRe

Also called: Gaussian Deformation Fields

## Overview
GauFRe is a method for real-time dynamic novel view synthesis using deformable 3D Gaussians [S2]. It solves the problem of reconstructing non-rigid dynamic scenes from monocular video by explicitly modeling temporal deformations of a canonical Gaussian primitive set [S2]. Unlike backward-warping deformation fields used in NeRFs, GauFRe uses a forward-warping deformation field, which is more efficient for explicit primitive-based systems because it predicts the deformed primitive state at a specific time rather than querying a canonical space for every ray sample [S2]. It further decouples static and dynamic regions using a GS-specific static component [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of monocular RGB images of a dynamic scene, along with camera poses (intrinsic $K$ and extrinsic $V$) and time stamps $t$ [S2].
*   **Outputs:** A rendered image for a given camera pose and time $t$, and the optimized parameters of the Gaussian primitives and deformation field [S2].

**Data Structures:**
The scene is represented by two sets of 3D Gaussians:
1.  **Deformable Set (Dynamic):** A set of $N_d$ Gaussians residing in a canonical space. Each Gaussian $i$ has attributes:
    *   Position: $x_i \in \mathbb{R}^3$ [S2].
    *   Covariance: $\Sigma_i \in \mathbb{R}^{3\times3}$, factored into rotation $R_i$ and scaling $s_i$ [S2].
    *   Density: $\sigma_i \in \mathbb{R}$ [S2].
    *   Color: $c_i$, represented by $m$-order spherical harmonics [S2].
2.  **Static Set (Non-deforming):** A set of $N_s$ Gaussians initialized around SfM-derived 3D points. These Gaussians do not deform over time [S2].

The deformation is modeled by a time-dependent forward-warping deformation field (DF), implemented as an MLP [S1][S2]. The MLP takes the canonical position $x_i$ and time $t$ as input and outputs delta values for scaling and rotation [S1][S2].

**Formulas:**
*   **Covariance Factorization:** To ensure positive semi-definiteness, the covariance matrix is factored as [S2]:
    $$ \Sigma_i = R_i \text{Exp}(s_i) \text{Exp}(s_i)^T R_i^T $$
    where $\text{Exp}(s_i)$ is a diagonal matrix with entries $e^{s_{i,k}}$, and $R_i$ is derived from a unit-length quaternion $q_i \in \mathbb{R}^4$ [S2].
*   **Deformation Field Output:** The MLP predicts deltas $\Delta s_i$ and $\Delta q_i$ (or rotation delta) for the scaling and rotation [S1][S2].
*   **Attribute Update:** The deformed scaling $s'_i$ and rotation $R'_i$ at time $t$ are computed by applying exponential and normalization operations to the deltas added to the canonical values [S1][S2]:
    $$ s'_i = \text{Exp}(s_i + \Delta s_i) $$
    $$ R'_i = \text{Normalize}(R_i + \Delta R_i) $$
    (Note: [S1] specifies exponential for scaling and normalization for rotation after adding deltas. [S2] confirms the forward-warping nature.)
*   **Screen-space Projection:** Given camera view transformation $V$ and projection matrix $K$, the 2D covariance $\Sigma'_i$ and projected mean $u_i$ are [S2]:
    $$ u_i = K V x'_i $$
    $$ \Sigma'_i = J V \Sigma'_i V^T J^T $$
    where $J$ is the Jacobian of the projection matrix $K$ [S2].

**Algorithm Steps:**
1.  **Initialization:**
    *   Initialize the position $x_i$ of the Gaussians using a 3D point cloud obtained from Structure-from-Motion (SfM) or random initialization [S2].
    *   Randomly initialize the point cloud into dynamic point clouds and static point clouds [S1][S3, unofficial].
    *   Initialize the static Gaussians around SfM-derived 3D points [S2].
2.  **Deformation Prediction:**
    *   For each Gaussian $i$ in the deformable set, input its canonical position $x_i$ and the current time $t$ into the deformation field MLP [S2].
    *   The MLP outputs delta values $\Delta s_i$ and $\Delta R_i$ (or quaternion delta) [S1][S2].
3.  **Attribute Update:**
    *   Compute the deformed scaling $s'_i$ by adding $\Delta s_i$ to the canonical scaling $s_i$ and applying the exponential operation [S1][S2].
    *   Compute the deformed rotation $R'_i$ by adding the rotation delta to the canonical rotation $R_i$ and applying normalization [S1][S2].
    *   Construct the deformed covariance $\Sigma'_i$ using the updated $s'_i$ and $R'_i$ [S2].
4.  **Rendering:**
    *   Project the deformed Gaussians (both dynamic and static) to screen space using the camera parameters $V$ and $K$ [S2].
    *   Rasterize the 2D Gaussians using elliptical weighted average splatting [S2].
    *   Composite the colors and densities to produce the final image [S2].
5.  **Optimization:**
    *   Optimize the entire system (canonical Gaussian attributes, static Gaussian attributes, and deformation field MLP) end-to-end using a self-supervised rendering loss [S2].
    *   Periodically clone, split, and prune Gaussians to balance quality and computational resources [S2].
    *   [S1] notes that for stable training, it first optimizes static 3DGS and then optimizes the deformation field, though [S2] states end-to-end optimization.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward_gaussian_deformation(canonical_gaussians, time, deformation_mlp, camera_params):
    """
    canonical_gaussians: List of Gaussian objects with attributes (x, s, q, sigma, c)
    time: float, current time step
    deformation_mlp: MLP taking (x, time) and returning (delta_s, delta_q)
    camera_params: dict with 'V' (view matrix) and 'K' (projection matrix)
    """
    deformed_gaussians = []
    
    # Process dynamic Gaussians
    for g in canonical_gaussians:
        # 1. Predict deltas
        delta_s, delta_q = deformation_mlp(g.x, time)
        
        # 2. Update scaling with exponential
        # s is a vector, delta_s is a vector
        new_s = exp(g.s + delta_s)
        
        # 3. Update rotation with normalization
        # q is a quaternion, delta_q is a quaternion delta
        new_q = normalize(g.q + delta_q)
        
        # 4. Construct covariance
        R = quaternion_to_rotation_matrix(new_q)
        S = diag(new_s)
        Sigma = R @ S @ S.T @ R.T
        
        # 5. Project to screen space
        # x is in canonical space, needs to be transformed? 
        # Note: Forward warping implies the position x might also be deformed or 
        # the deformation field outputs position delta. 
        # Sources [S2] say "forward-warping DF ... estimates the temporal set of deformed primitives".
        # It does not explicitly state position delta is output, but implies the primitive state is deformed.
        # Assuming position x is also deformed or the deformation field handles the full transform.
        # However, [S2] formula for projection uses x'_i. 
        # Let's assume the deformation field outputs position delta as well, or x is static in canonical space 
        # and the deformation is purely attribute-based? 
        # Re-reading [S2]: "forward-warping DF ... estimates the temporal set of deformed primitives".
        # And "parameterize the deformation for Gaussian attributes accordingly".
        # It is likely position is also deformed. Let's assume delta_x is also predicted or x is transformed.
        # Given the ambiguity, we will assume the deformation field outputs delta_x, delta_s, delta_q.
        # If not, x remains canonical? No, that would be static.
        # Let's assume delta_x is predicted.
        delta_x = deformation_mlp.get_position_delta(g.x, time) # Assumed component of MLP output
        
        new_x = g.x + delta_x
        
        # Project mean
        u = camera_params['K'] @ camera_params['V'] @ new_x
        
        # Project covariance
        J = jacobian_of_projection(camera_params['K'], new_x)
        Sigma_prime = J @ camera_params['V'] @ Sigma @ camera_params['V'].T @ J.T
        
        deformed_gaussians.append(Gaussian(x=new_x, Sigma=Sigma_prime, sigma=g.sigma, c=g.c))
        
    # Process static Gaussians (no deformation)
    for g in static_gaussians:
        # Project mean
        u = camera_params['K'] @ camera_params['V'] @ g.x
        
        # Project covariance
        J = jacobian_of_projection(camera_params['K'], g.x)
        Sigma_prime = J @ camera_params['V'] @ g.Sigma @ camera_params['V'].T @ J.T
        
        deformed_gaussians.append(Gaussian(x=g.x, Sigma=Sigma_prime, sigma=g.sigma, c=g.c))
        
    # Rasterize
    image = rasterize_splatting(deformed_gaussians, image_width, image_height)
    return image
```

**Parameters:**
*   Training time: Approximately 20 minutes for real-world scenes [S2].
*   Rendering performance: 96 FPS on an RTX 3090 GPU at 480x270 resolution [S2].

**Edge Cases and Pitfalls:**
*   **Static Regions:** Dynamic scenes often contain large static parts. Naively deforming all Gaussians can lead to artifacts in static regions. GauFRe addresses this by using a separate non-deforming set of Gaussians for static regions [S2].
*   **Optimization Stability:** Directly optimizing scaling and rotation can be unstable. Applying exponential to scaling and normalization to rotation after adding MLP deltas ensures convenient and reasonable optimization [S1].
*   **Backward vs. Forward Warping:** Using backward-warping in primitive-based systems is inefficient because the status of all primitives at a timestep must be known at once to render a ray. Forward-warping avoids this by explicitly predicting the deformed state [S2].

## Subtopics

### Foreground Gaussian Scale Residual Learning
A variant that initializes 3D Gaussian scales using KNN distances and learns scale residuals from volume latent features via a decoder to improve convergence and model capacity [S4]. It initializes the scale vector $\mathbf{s}_{init}$ using the average distance of $K$ nearest neighbors via the KNN algorithm, decodes the scale residual $\Delta \mathbf{s}$ from volume latent feature $\mathbf{f}_i$ using a decoder $\mathcal{D}_{cov}$, and computes the final scale $\mathbf{s} = \mathbf{s}_{init} + \Delta \mathbf{s}$ [S4]. It uses a 3D vector $\mathbf{s}$ for scaling and a quaternion $\mathbf{q}$ for rotation separately to allow independent optimization [S4].

## Operating characteristics
*   **Time Complexity:** Optimization takes approximately 20 minutes for real-world scenes, which is significantly faster than NeRF-based methods that take hours [S2].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Achieves 96 FPS real-time rendering on an RTX 3090 GPU at 480x270 resolution [S2].
*   **Quality:** On the D-NeRF dataset, GauFRe achieves a PSNR of 34.80, SSIM of 0.985, and LPIPS of 0.028 [S1].
*   **Strengths:** Real-time rendering, efficient optimization, explicit modeling of non-rigid transformations, decoupling of static and dynamic regions [S2].
*   **Limitations:** Requires SfM initialization for static components [S2]. Performance is measured at a specific resolution (480x270) [S2].

## Background
GauFRe (Gaussian Deformation Fields) was proposed to address the challenge of high-quality 3D reconstruction of dynamic scenes from monocular video [S2]. It builds on 3D Gaussian Splatting [S2] and introduces a forward-warping deformation field to model non-rigid transformations [S2]. The method was shown to achieve competitive results with higher efficiency compared to previous state-of-the-art NeRF and Gaussian-based methods [S2].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] GauFRe: Gaussian Deformation Fields for Real-Time Dynamic Novel View Synthesis, https://doi.org/10.1109/wacv61041.2025.00262, fetched 2026-10-26
[S3] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
[S4] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
