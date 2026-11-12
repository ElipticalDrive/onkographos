# Joint Pose Optimization

Also called: BARF, Unposed images methods

## Overview
Joint Pose Optimization (JPO) eliminates the dependency on external Structure-from-Motion (SfM) pipelines by simultaneously optimizing camera poses and the scene representation (neural or Gaussian) from unposed images [S1]. Unlike sibling methods that may rely on incremental depth priors or temporal continuity, JPO directly minimizes photometric reconstruction error to solve for both geometry and pose [S1]. It is distinct from feed-forward methods that predict poses via a separate network, as JPO treats poses as learnable parameters optimized via gradient descent [S1].

## Shared principles
All methods in this family rely on differentiable rendering to compute gradients from the rendered image to the Gaussian parameters (position, covariance, opacity, spherical harmonics) and/or network weights [S1, 6136-instantsplat.md, 6464-pixelsplat.md]. The rendering process remains differentiable with respect to the scene representation [S1, 6136-instantsplat.md, 6464-pixelsplat.md]. Loss functions typically combine photometric reconstruction error, such as $\mathcal{L}_{L1}$ or $\mathcal{L}_{MSE}$, with perceptual or structural metrics like $\mathcal{L}_{MS-SSIM}$ or $\mathcal{L}_{LPIPS}$ [6136-instantsplat.md][6464-pixelsplat.md].

## Subtopics

### [1D Signal Alignment for Joint Optimization Analysis](8423-1d-signal-alignment-for-joint-optimization-analysis.md)
A simplified analytical framework used to study the convergence behavior of joint camera pose and scene reconstruction optimization. It reduces the complex 3D joint optimization problem into a 1D signal alignment task to analyze how the signal spectrum affects the gradient landscape. Unlike the full 3D method, this approach isolates the "local phenomenon" of joint alignment, revealing that high-frequency signals cause gradient noise and local optima, which can be mitigated by Gaussian filtering.

## Background
Joint Pose Optimization was introduced to address the limitations of traditional pipelines that depend on off-the-shelf pose estimation methods, which increase inference time and fail in texture-less regions or when images lack overlap [S1]. Early works proposed joint optimization for both static [HJS∗ 23, WWX∗ 21, CL23a, LMTL21, SCW∗ 25] and dynamic settings [LGM∗ 23]. However, these initial methods required rough pose initialization and were limited to small motions [S1]. Subsequent research extended these ideas to generalizable representations, enabling joint pose prediction and representation learning from sparse image collections [CL23b, HJS∗ 23, XLC∗ 25]. Notably, Ye et al. [YLX∗ 25] demonstrated that pose estimation and novel-view synthesis could be performed from unposed images using only photometric loss, though this specific approach is limited to static scenes [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
