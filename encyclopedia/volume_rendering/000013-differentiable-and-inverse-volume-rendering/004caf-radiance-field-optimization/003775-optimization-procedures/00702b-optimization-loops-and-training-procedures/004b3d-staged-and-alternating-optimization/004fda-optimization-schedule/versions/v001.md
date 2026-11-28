# Optimization Schedule

Also called: Staged Optimization

## Overview
Optimization Schedule refers to the structured progression of training parameters in differentiable volume and radiance field reconstruction, designed to mitigate local minima and ensure stable convergence. It addresses the problem of ambiguous gradients in inverse rendering by decoupling the optimization of geometric structure from material properties or fine-scale details. Unlike simultaneous optimization of all parameters, which often leads to instability in sparse-view settings, this method separates the process into distinct phases or scales, allowing the solver to first establish a coarse global alignment before refining local details [S1, S2].

## How it works
The method involves defining a sequence of optimization stages, each with specific constraints on the learnable parameters (such as rotation, scale, displacement, or texture resolution). The general workflow consists of initializing the scene representation, executing a coarse alignment phase, and proceeding to a regularized or multi-scale refinement phase.

### Inputs and Outputs
- **Inputs**: Initial scene representation (e.g., Gaussian primitives or mesh with displacement maps), set of target images with known camera poses, number of views, and optimization hyperparameters (learning rate, momentum, step counts).
- **Outputs**: Optimized scene parameters (positions, rotations, scales, densities, colors, or texture maps) that minimize the difference between rendered and target images.

### Data Structures
The specific data structures depend on the underlying representation (e.g., 3D Gaussians or parametric meshes), but generally include:
- **Gaussian Primitives**: Position $\mathbf{p}$, rotation matrix $\mathbf{R}$, scale vector $\mathbf{s}$, opacity $\alpha$, and spherical harmonic coefficients for color.
- **Mesh/Displacement**: Base mesh geometry, displacement map texture, reflectance/texture map.

### Numbered Steps
1. **Initialization**:
   - Initialize the scene representation. For Gaussian splatting, this involves placing primitives and setting initial opacity based on the number of views to ensure gradient propagation [S1]. For mesh-based methods, initialize with a coarse mesh having similar topology to the target [S2].
   - Set initial parameters: rotation matrices to identity, scales to a default value, and textures to constant or low-resolution values [S1, S2].

2. **Coarse Alignment (Stage 1)**:
   - Perform optimization for a fixed number of iterations (e.g., 1,000 iterations) [S1].
   - In some variants, this stage may involve free optimization of position and scale while keeping rotation fixed or using a simplified objective [S1].
   - The goal is to achieve a rough alignment of the scene geometry with the target views.

3. **Regularized/Multi-scale Optimization (Stage 2)**:
   - **Constrained Phase**: For a specified number of iterations (e.g., first 8,000 of 13,000 total in CoherentGS), keep certain parameters fixed or constrained. For example, keep the rotation matrix as identity and optimize scale according to a specific equation (Eq. 9 in [S1]) [S1].
   - **Free Optimization Phase**: For the remaining iterations (e.g., last 5,000), freely optimize all parameters including rotation and scale [S1].
   - **Multi-scale Refinement**: In mesh-based approaches, progressively increase the degrees of freedom by transitioning to finer resolutions of displacement and reflectance textures every 40 steps [S2].
   - Use gradient descent with momentum, learning rate decay, and maximum step size constraints [S2].
   - Minimize a distance metric, such as CIELAB $\Delta E$ for color accuracy [S2].

### Formulas
- **Opacity Initialization**: Opacity $\alpha$ is set based on the number of views $V$:
  - $V=2 \implies \alpha = 0.6$
  - $V=3 \implies \alpha = 0.5$
  - $V=4 \implies \alpha = 0.35$ [S1]
- **Scale Assignment**: In the constrained phase, scale is assigned according to Eq. 9 of [S1]. The exact formula is not provided in the source text, but it is used to initialize scale before free optimization [S1].
- **Rendering Equation**: The color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ is computed as:
  $$
  \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
  $$
  where $\sigma_i$ is density, $\delta_i$ is step size, $\mathbf{c}_i$ is color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is transmittance [44-radiance-field-optimization.md].

### Parameters
- **Optimizer**: Adam [S1].
- **Iterations**:
  - Coarse Alignment: 1,000 iterations [S1].
  - Regularized Optimization: 13,000 iterations total (8,000 constrained + 5,000 free) [S1].
  - Multi-scale steps: 300 to 500 optimization steps [S2].
- **Resolution Updates**: Transition to finer textures every 40 steps [S2].
- **Samples per Pixel**: 16 for optimization rendering [S2].
- **Views**: 4 to 7 different views [S2].
- **Path Pairs**: 8 pairs of paths per pixel [S2].

### Edge Cases and Pitfalls
- **Local Minima**: Naïve gradient-based optimization tends to get stuck in local minima, especially when reconstructing geometry and materials simultaneously [S2]. The multi-scale approach mitigates this by reconstructing low-frequency features (shape) before high-frequency details (texture) [S2].
- **Gradient Propagation**: Initial opacity must be adjusted based on the number of views to ensure sufficient gradient flow [S1].
- **Ambiguity**: Using high-resolution reflectance textures too early can lead to ambiguities where incorrect shapes are compensated by high-frequency texture details [S2]. Therefore, reflectance texture resolution should be lower than displacement map resolution at the beginning [S2].

## Subtopics

### Multi-scale shape optimization
A variant used for inverse rendering problems involving the reconstruction of both geometry and materials from photographs. It initializes the scene with a coarse mesh and constant textures, then progressively increases the resolution of displacement and reflectance textures every 40 steps. This ensures that shape details are reconstructed before texture details, reducing ambiguities. It uses CIELAB $\Delta E$ distance metric, momentum, learning rate decay, and maximum step size constraints. It typically uses 300-500 steps, 8 path pairs per pixel, 4-7 views, and 16 samples per pixel [S2].

## Operating characteristics
- **Time Complexity**: Depends on the number of iterations and the complexity of the rendering equation. Multi-scale methods may require more iterations due to resolution transitions, but each iteration at lower resolution is cheaper.
- **Memory Complexity**: Standard differentiable rendering memory usage, potentially reduced by explicit gradients or adjoint methods [44-radiance-field-optimization.md].
- **Strengths**: Improves convergence in sparse-view settings; avoids local minima by decoupling geometry and material optimization; handles ambiguous scenes better by prioritizing low-frequency features.
- **Limitations**: Requires careful tuning of stage transitions and hyperparameters; may be slower than single-stage optimization if not implemented efficiently.
- **When to Use**: Sparse novel view synthesis; inverse rendering with geometry and material reconstruction; scenes with complex shapes and specular reflections.
- **When Not to Use**: Dense-view settings where simultaneous optimization might converge sufficiently; real-time applications where training time is critical.

## Background
The concept of staged optimization in differentiable rendering originates from the need to handle the ill-posed nature of inverse rendering problems. CoherentGS [S1] introduced a specific schedule for 3D Gaussian splatting to improve sparse view synthesis, using a coarse alignment followed by regularized optimization. Reparameterizing discontinuous integrands for differentiable rendering [S2] proposed a multi-scale approach for mesh-based inverse rendering to avoid local minima in shape and texture reconstruction. These methods build on the differentiability of the volume rendering equation [44-radiance-field-optimization.md].

## Sources
[S1] CoherentGS: Sparse Novel View Synthesis with Coherent 3D Gaussians, https://doi.org/10.1007/978-3-031-73404-5_2, fetched 2026-10-27
[S2] Reparameterizing discontinuous integrands for differentiable rendering, https://doi.org/10.1145/3355089.3356510, fetched 2026-10-26
