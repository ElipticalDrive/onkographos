# Non-Pinhole Camera Training

Also called: non-pinhole 3DGS

## Overview
Non-Pinhole Camera Training extends the 3D Gaussian Splatting optimization framework to handle camera models that deviate from the standard pinhole assumption, such as those found in wide-angle or fisheye lenses. It solves the problem of geometric distortion and projection errors that cause standard 3D Gaussian Splatting to fail when applied to datasets captured with non-pinhole optics. It differs from standard training by replacing the linear perspective projection with a differentiable non-pinhole projection model, allowing the rasterizer to correctly map 3D Gaussians to image pixels for distorted views.

## How it works
The method relies on a differentiable rasterizer that supports arbitrary camera projection functions. The core modification is the replacement of the standard view-projection matrix with a non-pinhole projection pipeline.

**Inputs:**
- 3D Gaussian parameters: mean $\mu$, covariance $\Sigma$ (or rotation $R$ and scale $S$), opacity $\alpha$, spherical harmonics coefficients $C$.
- Camera parameters: intrinsic matrix $K$ (potentially including distortion coefficients), extrinsic matrix $W$ (world-to-camera), and the specific non-pinhole projection function $P_{np}$.
- Image resolution $H \times W$.

**Outputs:**
- Rendered color image $I$.
- Gradients of the loss with respect to Gaussian parameters.

**Data Structures:**
- Gaussian attributes are stored in standard 3DGS format (position, scale, rotation, opacity, SH coefficients).
- Camera state includes distortion parameters if applicable.

**Steps:**
1. **Transform Gaussians:** Transform Gaussian means $\mu$ from world space to camera space using the extrinsic matrix $W$: $\mu_c = W \mu$.
2. **Non-Pinhole Projection:** Project the 3D camera-space point $\mu_c$ to 2D image coordinates $(u, v)$ using the non-pinhole projection function $P_{np}$. For a standard pinhole camera, this is $u = f_x \frac{x_c}{z_c} + c_x$. For non-pinhole models, $P_{np}$ may involve radial distortion correction or a specific fisheye projection model.
   $$
   \begin{bmatrix} u \\ v \end{bmatrix} = P_{np}\left(\frac{x_c}{z_c}, \frac{y_c}{z_c}\right)
   $$
   [G] The function $P_{np}$ must be differentiable with respect to $x_c, y_c, z_c$ to allow gradient flow back to the Gaussian positions.
3. **Covariance Projection:** Compute the 2D covariance matrix $\Sigma'$ in screen space. This requires the Jacobian $J$ of the projection function $P_{np}$ with respect to the 3D coordinates, and the view-space covariance $\Sigma_c$.
   $$
   \Sigma' = J W_{view} \Sigma_c W_{view}^T J^T
   $$
   [G] For non-pinhole models, $J$ is the Jacobian of the non-linear projection function, not just the linear perspective division.
4. **Rasterization:** Rasterize the projected Gaussians using the standard alpha-blending accumulation, but using the distorted 2D positions and covariances.
5. **Loss Computation:** Compute the photometric loss (e.g., L1 or SSIM) between the rendered image and the ground truth non-pinhole image.
6. **Backpropagation:** Compute gradients through the non-pinhole projection function $P_{np}$ to update Gaussian positions and other parameters.

**Parameters:**
- Distortion coefficients: Specific to the camera model (e.g., $k_1, k_2$ for radial distortion). Not given in the sources.
- Optimization learning rate: Standard 3DGS values. Not given in the sources.

**Edge Cases:**
- **Degenerate Depth:** Points with $z_c \le 0$ must be clipped or ignored to avoid division by zero or invalid projections.
- **Distortion Singularities:** Extreme distortion near the image center or edges may require careful handling of the Jacobian to prevent numerical instability.

## Subtopics

## Operating characteristics
- **Time Complexity:** Similar to standard 3DGS rasterization, but with potentially higher per-pixel cost due to the non-linear projection function evaluation.
- **Memory Complexity:** Same as standard 3DGS.
- **Performance:** On the Matterport dataset, which contains non-pinhole camera captures, the proposed method successfully reconstructs the scene, whereas standard 3D-GS fails entirely [S1].
- **Strengths:** Enables high-fidelity reconstruction of scenes captured with wide-angle or fisheye lenses without requiring pre-distortion correction or specialized preprocessing.
- **Limitations:** Requires a differentiable implementation of the specific non-pinhole projection model. Performance may degrade if the projection function is not smooth or differentiable everywhere.

## Background
The method was introduced in "On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy" [S1]. It addresses the limitation of standard 3D Gaussian Splatting, which assumes a pinhole camera model, by demonstrating that adapting the rasterizer to support non-pinhole projections allows for successful training on datasets like Matterport, where standard 3DGS fails.

## Sources
[S1] On the Error Analysis of 3D Gaussian Splatting and an Optimal Projection Strategy, https://doi.org/10.1007/978-3-031-72643-9_15, fetched 2026-10-29
