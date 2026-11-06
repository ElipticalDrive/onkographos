# Plenoxels regularization ablation

Also called: Plenoxels TV and sparsity regularization

## Overview
This article documents the regularization strategies used in the Plenoxels framework to stabilize the optimization of sparse voxel-based radiance fields. It addresses the problem of visual artifacts (floaters, holes) and memory inefficiency inherent in high-dimensional, non-convex optimization of explicit representations [S1]. It distinguishes itself from sibling methods by combining Total Variation (TV) regularization on both voxel opacities and spherical harmonic coefficients with scene-specific sparsity and beta losses, rather than relying solely on geometric constraints or tomographic projections [S1, 3265-space-time-tomographic-projection-stp-regularizer.md, 3273-anisotropic-total-variation.md].

## Shared principles
The core mechanism involves adding penalty terms to the reconstruction loss to enforce smoothness and sparsity [S1]. The total loss is defined as:

$$
\mathcal{L} = \mathcal{L}_{recon} + \lambda_{TV} \mathcal{L}_{TV} + \mathcal{L}_{aux}
$$

where $\mathcal{L}_{recon}$ is the reconstruction loss computed over sampled rays [1132-plenoxel-optimization-with-tv-regularization.md]. The primary regularization term $\mathcal{L}_{TV}$ is the isotropic Total Variation computed over sampled voxels and SH dimensions [S1]:

$$
\mathcal{L}_{TV} = \frac{1}{|\mathcal{V}|} \sum_{\substack{\mathbf{v} \in \mathcal{V} \\ d \in [D]}} \sqrt{\Delta_x^2(\mathbf{v}, d) + \Delta_y^2(\mathbf{v}, d) + \Delta_z^2(\mathbf{v}, d)}
$$

Here, $D$ is the dimensionality of the SH coefficients, $\mathcal{V}$ is a stochastic batch of active voxels, and $\Delta^2_x(\mathbf{v}, d)$ is the squared difference between the $d$-th value in voxel $\mathbf{v}=(i,j,k)$ and its neighbor $(i+1,j,k)$, normalized by the voxel resolution [S1]. The terms $\Delta^2_y$ and $\Delta^2_z$ are computed analogously for the y and z directions [G]. The TV weight $\lambda_{TV}$ is fixed per scene type (bounded, forward-facing, 360°) and differs for SH coefficients and $\sigma$ [S1]. Auxiliary losses $\mathcal{L}_{aux}$ include sparsity and beta regularizers applied to specific scene types [S1].

## Subtopics

### [Plenoxel Optimization with TV Regularization](1132-plenoxel-optimization-with-tv-regularization.md)
This method trains sparse voxel-based radiance fields by combining MSE reconstruction loss with TV regularization on voxel opacities ($\sigma$) and spherical harmonic (SH) coefficients [1132-plenoxel-optimization-with-tv-regularization.md]. It explicitly penalizes high-frequency spatial variations in the voxel grid to prevent visual artifacts and ill-conditioning [1132-plenoxel-optimization-with-tv-regularization.md]. It builds on [Plenoxels](52-plenoxels.md) and [Plenoxels regularization ablation](901-plenoxels-regularization-ablation.md).

### Sparsity and Beta Loss Regularization
These are auxiliary regularizers used in addition to TV regularization [S1].

**Sparsity Loss** is applied to forward-facing and 360° scenes to encourage voxels to be empty, reducing memory footprint and quality loss during upsampling [S1]. It is evaluated at each sample on each active ray in each minibatch [S1]. The formula is:

$$
\mathcal{L}_s = \lambda_s \sum_{i,k} \log \left( 1 + 2\sigma(\mathbf{r}_i(t_k))^2 \right)
$$

where $\sigma(\mathbf{r}_i(t_k))$ is the opacity of sample $k$ along training ray $i$ [S1].

**Beta Loss** is applied to 360° scenes to promote clear foreground-background decomposition by encouraging foreground transmittance to be either fully opaque or empty [S1]. The formula is:

$$
\mathcal{L}_\beta = \lambda_\beta \sum_{\mathbf{r}} (\log(T_{FG}(\mathbf{r})) + \log(1 - T_{FG}(\mathbf{r})))
$$

where $T_{FG}(\mathbf{r})$ is the accumulated foreground transmittance of ray $\mathbf{r}$ [S1].

**Comparison and Ablation Results:**
On 8 forward-facing NeRF scenes, applying TV on both SH and $\sigma$ yields PSNR 26.29, SSIM 0.839, LPIPS 0.210 [S1]. Removing TV drops PSNR to 25.25 (TV $\sigma$ only) or 25.80 (TV SH only) [S1]. Sparsity alone yields PSNR 26.31 [S1]. TV regularization on $\sigma$ has a quantitatively larger effect than on SH, but both are essential for avoiding visual artifacts on forward-facing scenes with few training views [S1]. Without TV regularization on either, 3 out of 8 scenes run out of memory [S1]. The sparsity regularizer is not strictly necessary for image quality but is retained to reduce memory footprint [S1].

## Background
The Plenoxels framework utilizes sparse voxel grids and spherical harmonics to represent radiance fields without neural networks, enabling fast rendering and optimization [S1]. The regularization strategies described herein were developed to address the instability and memory issues associated with optimizing high-dimensional SH coefficients in sparse voxel structures [S1]. The implementation relies on custom CUDA extensions for differentiable volume rendering due to limited support in standard autodiff libraries [S1].

## Sources
[S1] Plenoxels: Radiance Fields without Neural Networks, https://doi.org/10.1109/cvpr52688.2022.00542, fetched 2026-10-26
