# Joint Pose Optimization

Also called: BARF, Unposed images methods

## Overview
Joint Pose Optimization (JPO) refers to methods that simultaneously optimize camera poses and the scene representation (e.g., neural radiance fields or 3D Gaussians) to eliminate the dependency on pre-computed camera parameters from external Structure-from-Motion (SfM) pipelines [S1]. Unlike methods that assume known poses or rely on fixed priors, JPO treats camera extrinsics as learnable variables updated via gradient descent alongside the scene geometry and appearance [S1]. This approach is distinct from incremental methods that leverage mono-depth priors or temporal continuity to estimate poses [S1].

## Shared principles
All methods in this family output a set of 3D Gaussian primitives $\{G_i\}$ defined by mean $\mu_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and spherical harmonics coefficients $c_{i,lm}$ [6136-instantsplat.md][6464-pixelsplat.md]. The rendering process remains differentiable with respect to the scene representation [S1, 6136-instantsplat.md, 6464-pixelsplat.md]. In feed-forward variants, gradients flow through the network weights during training; in prior-based variants, gradients update Gaussian attributes while poses remain fixed or are ignored [S1, 6136-instantsplat.md, 6464-pixelsplat.md]. Loss functions typically combine photometric reconstruction error, such as $\mathcal{L}_{L1}$ or $\mathcal{L}_{MSE}$, with perceptual or structural metrics like $\mathcal{L}_{MS-SSIM}$ or $\mathcal{L}_{LPIPS}$ [6136-instantsplat.md][6464-pixelsplat.md].

## Subtopics

### [InstantSplat](6136-instantsplat.md)
A method that leverages sparse image collections to obtain generalizable representations, enabling joint pose prediction and representation learning.

### [pixelSplat](6464-pixelsplat.md)
A method that efficiently estimates pose and performs novel-view synthesis from unposed images and intrinsic camera parameters using only photometric loss.

## Background
The need for Joint Pose Optimization arises from the limitations of traditional pipelines that depend on off-the-shelf pose estimation methods, which increase inference time and may fail in texture-less regions or when images lack overlap [S1]. To eliminate the need for known camera parameters, several works propose to jointly optimize camera poses and neural representation, both in static [HJS∗ 23,WWX∗ 21,CL23a,LMTL21,SCW∗ 25] and dynamic settings [LGM∗ 23]. However, these methods still require rough pose initialization and are limited to small motions [S1]. Alternative solutions follow incremental approaches, leveraging mono-depth prior [BWL∗ 23] or temporal continuity [FLK∗ 24] to optimize camera parameters [S1]. Several works have extended this idea to obtain generalizable representations [CL23b,HJS∗ 23,XLC∗ 25], enabling joint pose prediction and representation learning from sparse image collections [S1]. Ye et al. [YLX∗ 25] demonstrated that it was possible to efficiently estimate pose and perform novel-view synthesis from unposed images and intrinsic camera parameters using only photometric loss [S1]. It is, however, limited to static scenes [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
