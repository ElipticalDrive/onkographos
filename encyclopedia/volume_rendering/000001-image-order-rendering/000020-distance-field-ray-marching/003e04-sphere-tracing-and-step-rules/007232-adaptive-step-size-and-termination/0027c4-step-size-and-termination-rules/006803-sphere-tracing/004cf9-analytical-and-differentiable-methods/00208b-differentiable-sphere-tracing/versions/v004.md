# Differentiable Sphere Tracing

Also called: Differentiable Sphere Tracing Renderer, DIST

## Overview
Differentiable Sphere Tracing (DIST) is a rendering algorithm for implicit Signed Distance Functions (SDFs) represented by neural networks, designed to enable efficient forward and backward passes for inverse graphics optimization [S1]. It solves the problem of non-differentiable or computationally prohibitive ray marching in neural implicit representations by optimizing the rendering pipeline for GPU execution [S1]. Unlike naive sphere tracing, which requires recursive computational graphs for every query, DIST uses a coarse-to-fine strategy and a gradient approximation to reduce memory and computation costs [S1].

## How it works
**Inputs:**
- Neural SDF function $f: \mathbb{R}^3 \to \mathbb{R}$ (or grid-based SDF) [S1][S4, unofficial].
- Camera intrinsics $K$ and extrinsics (position $c$, rotation $R$) [S1][S2, unofficial].
- Ray marching parameters: `num_iterations` (max steps), `convergence_threshold` [S1][S3, unofficial].
- Optional bounding volume: spherical `bounding_radius` or axis-aligned `bounding_box` [S2, S4, unofficial].

**Outputs:**
- Rendered image (color/depth), surface normals, opacity, and gradients w.r.t. SDF parameters [S1][S4, unofficial].

**Data Structures:**
- `ray_positions`: Tensor of shape $(H, W, 3)$ storing current ray origins [S1][S3, unofficial].
- `ray_directions`: Tensor of shape $(H, W, 3)$ storing normalized ray vectors [S1][S2, unofficial].
- `foreground_masks`: Boolean tensor $(H, W, 1)$ indicating valid pixels [S1][S3, unofficial].
- `signed_distances`: Tensor $(H, W, 1)$ storing $f(p^{(n)})$ [S1][S3, unofficial].

**Algorithm Steps:**

1. **Ray Initialization:**
   Generate a 2D homogeneous grid for image dimensions $(H, W)$ and project to camera space using $K^{-1}$ to create `homo_calib` $(3, H \times W)$ [S1][S2, unofficial].
   Compute ray directions by rotating `homo_calib` with $R^T$ and normalizing:
   $$ \mathbf{v} = \text{normalize}(R^T @ \text{homo\_calib}) $$
   Set initial positions $\mathbf{p}^{(0)} = \mathbf{c}$ (camera center) and accumulated distance $d^{(0)} = 0$ [S1].
   Initialize `foreground_masks` to `True` for all finite positions [S1][S3, unofficial].

2. **Bounding Volume Check (Optional):**
   If `bounding_radius` is set, compute intersection with the sphere:
   $$ a = \|\mathbf{v}\|^2, \quad b = 2(\mathbf{v} \cdot \mathbf{p}), \quad c = \|\mathbf{p}\|^2 - r^2 $$
   $$ \Delta = b^2 - 4ac $$
   If $\Delta \ge 0$, update positions to the first hit:
   $$ t = \frac{-b - \sqrt{\Delta}}{2a} $$
   $$ \mathbf{p} = \text{where}(\Delta \ge 0, \mathbf{p} + \mathbf{v}t, \mathbf{p}) $$
   Update `foreground_masks` to exclude rays outside the radius [S1][S3, unofficial].

3. **Ray Marching Loop:**
   For $n = 0$ to `num_iterations`:
   a. **Query SDF:** Compute $\mathbf{b}^{(n)} = f(\mathbf{p}^{(n)})$ [S1][S3, unofficial].
   b. **Convergence Check:** Determine if rays have hit the surface:
      $$ \text{converged} = |\mathbf{b}^{(n)}| < \text{convergence\_threshold} $$
      If all rays are converged or out of foreground, break [S1][S3, unofficial].
   c. **Update Position:**
      For $n=0$, update all foreground rays:
      $$ \mathbf{p}^{(n+1)} = \mathbf{p}^{(n)} + \mathbf{v} \cdot \mathbf{b}^{(n)} $$
      For $n>0$, update only unconverged foreground rays:
      $$ \mathbf{p}^{(n+1)} = \text{where}(\text{foreground\_masks} \land \neg \text{converged}, \mathbf{p}^{(n)} + \mathbf{v} \cdot \mathbf{b}^{(n)}, \mathbf{p}^{(n)}) $$
      [S1][S3, unofficial].

4. **Normal and Shading Computation:**
   Compute surface normals $\mathbf{n}$ as the gradient of the SDF at the converged positions [S1][S4, unofficial].
   For grid-based SDFs, interpolate normals from voxel corners [S4, unofficial].
   Apply shading model (e.g., point light at camera):
   $$ L = \frac{2 \max(\mathbf{l} \cdot \mathbf{n}, 0)}{\|\mathbf{p}_{light} - \mathbf{p}_{hit}\|^2} $$
   [S4, unofficial].

5. **Backward Pass (Gradient Approximation):**
   To avoid recursive back-propagation through the marching loop [S1]:
   a. Detach `ray_positions` and enable gradients: `ray_positions.requires_grad_(True)` [S3, unofficial].
   b. Re-compute SDF values $\mathbf{b}$ at these positions [S3, unofficial].
   c. Compute gradient of SDF w.r.t. positions:
      $$ \nabla_p \mathbf{b} = \frac{\partial \mathbf{b}}{\partial \mathbf{p}} $$
      [S3, unofficial].
   d. Compute output gradient approximation:
      $$ \frac{\partial \mathcal{L}}{\partial \mathbf{b}} = -\frac{\frac{\partial \mathcal{L}}{\partial \mathbf{p}} \cdot \mathbf{v}}{\nabla_p \mathbf{b} \cdot \mathbf{v}} $$
      Zero gradients for unconverged points [S3, unofficial].
   e. Compute gradients w.r.t. SDF parameters using standard autograd [S3, unofficial].

**Parameters:**
- `num_iterations`: Max steps. Typical: 2000 [S3, unofficial].
- `convergence_threshold`: SDF threshold for hit. Typical: $10^{-3}$ [S3, unofficial].
- `march_step`: Steps per ray. Default: 50 [S2, unofficial].
- `bounding_radius`: Optional spherical bound [S2, unofficial].
- `grid_res_x/y/z`: Voxel grid dimensions [S4, unofficial].

**Pitfalls:**
- Naive sphere tracing is prohibitive for neural networks due to memory/computation [S1].
- Division by zero in gradient calculation: clamp `grad_positions_dot_directions` to $\pm 10^{-6}$ [S3, unofficial].
- Non-intersecting pixels must be handled by masking to avoid artifacts [S4, unofficial].
- Normalization requires epsilon ($10^{-12}$) to prevent division by zero [S2, unofficial].

## Subtopics

### [Instant-NSR-PL Forward Pass](2540-instant-nsr-pl-forward-pass.md)
Renders scenes by marching rays through a neural SDF, accumulating color, normals, depth, and opacity. It computes composite RGB, normals, depth, and opacity by querying the geometry network for SDF values and gradients, computing opacity from SDF and ray direction, and accumulating results.

### [Differentiable Silhouette Rendering](4322-differentiable-silhouette-rendering.md)
Renders binary silhouettes of SDF shapes with gradient back-propagation. It solves non-differentiability of standard silhouette detection by computing a "soft" silhouette value based on the minimum absolute SDF encountered along a ray. It differs from DIST by focusing on silhouette supervision rather than full photometric rendering, and from Instant-NSR-PL by using minimum absolute distance rather than normal-based edge detection.

### [SDF Direct Diffuse Integrator](4467-sdf-direct-diffuse-integrator.md)
Computes direct illumination for SDF scenes, extending path tracing with a boundary integral term at grazing points for accurate gradients on thin surfaces. Unlike DIST, it explicitly handles visibility discontinuities at object boundaries using a relaxed boundary formulation.

### [Multi-View Photometric Shape Prediction](5427-multi-view-photometric-shape-prediction.md)
Predicts 3D shape from multiple color images by minimizing cross-view photometric consistency loss without segmentation masks. It differs from encoder-based methods by using random initialization and implicit representation, robust to noise and focal length changes.

### [Camera Pose Optimization via Depth and Silhouette](5667-camera-pose-optimization-via-depth-and-silhouette.md)
Optimizes camera pose by minimizing gradients from 2D depth and silhouette observations. It builds on DIST to render scenes using Fuzzy Metaballs, enabling gradient descent on camera parameters. Unlike global pose optimization, it uses direct gradient backpropagation through the rendering primitive to pose parameters. It distinguishes itself from learnable local warp methods by using analytical gradients rather than a dedicated MLP.

#### Single-View Depth Shape Completion
Recovers complete 3D shape from a single depth image by optimizing a latent code through a differentiable renderer, requiring no additional training [S1]. It fixes pre-trained DeepSDF decoder parameters, initializes latent code as mean shape, renders depth and silhouette, and backpropagates to optimize the latent code [S1]. The loss is $\arg \min_{\mathbf{z}} \mathcal{L}_d(\mathcal{R}_d(f(\mathbf{z})), I_d) + \mathcal{L}_s(\mathcal{R}_s(f(\mathbf{z})), I_s)$ [S1]. It outperforms DeepSDF on dense depth because it samples adaptively via differentiable sphere tracing rather than fixed rules [S1].

#### Reparameterization for Differentiable SDF Rendering
Uses a reparameterization function $T$ for SDFs that satisfies the boundary requirement $\partial_\pi T(\omega_b, \pi) = \partial_\pi \omega_b$ [S5, unofficial]. It constructs a 3D vector field $\mathbf{V}(x, \pi) := -\frac{\partial_x \phi(x, \pi_0)}{\phi(x, \pi) \|\partial_x \phi(x, \pi_0)\|_2}$ [S5, unofficial]. The evaluation distance $t$ is computed as a weighted average of intermediate sphere tracing distances: $t := \frac{\sum_{i=1}^N w^{(i)} t_i}{\sum_{i=1}^N w^{(i)}}$ [S5, unofficial].

## Operating characteristics
- **Time Complexity:** Linear in the number of ray marching steps and pixels. Optimized by coarse-to-fine strategy and early termination [S1].
- **Memory Complexity:** Proportional to the number of active rays and SDF network size. Gradient approximation reduces memory compared to full back-propagation [S1].
- **Performance:** Efficient on commodity GPUs. DIST uses aggressive ray marching and safe convergence criteria to save computation [S1].
- **Strengths:** Fully differentiable, supports inverse graphics, handles neural SDFs efficiently [S1].
- **Limitations:** Requires careful tuning of convergence threshold and bounding volumes. Gradient approximation may introduce errors [S1][S3, unofficial].

## Background
DIST was proposed by Liu et al. in 2020 to bridge the gap between inverse graphics and deep learning-based implicit SDFs [S1]. It addresses the computational and memory prohibitions of naive sphere tracing in neural networks [S1]. The method has been implemented in PyTorch and used for single-view and multi-view inverse optimization [S2, unofficial]. It builds on sphere tracing algorithms [14] and integrates with deep learning frameworks for end-to-end training [S1].

## Sources
[S1] DIST: Rendering Deep Implicit Signed Distance Function With Differentiable Sphere Tracing, https://doi.org/10.1109/cvpr42600.2020.00209, fetched 2026-10-26
[S2] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
[S3] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S4] YueJiang-nj/CVPR2020-SDFDiff, https://github.com/YueJiang-nj/CVPR2020-SDFDiff, fetched 2026-10-27 (unofficial)
[S5] A Survey on Physics-based Differentiable Rendering - arXiv.org, https://arxiv.org/pdf/2504.01402, fetched 2026-10-29 (unofficial)
