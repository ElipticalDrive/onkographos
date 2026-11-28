# Depth-Regularizing Loss Functions

Also called: Depth Supervision, Geometric Regularization

## Overview
Depth-Regularizing Loss Functions constrain the geometric consistency of differentiable volume rendering by penalizing discrepancies between rendered depth maps and external geometric estimates, such as stereo-matching outputs or optical flow correspondences. Unlike photometric losses that only enforce color fidelity, these methods explicitly regularize the scene's implicit geometry, accelerating convergence and improving robustness in textureless or occluded regions. They are distinguished from sibling methods like [MIMO Objective](10931-mimo-objective.md) (which focuses on multi-scale image features) and [Ray-Based Volumetric Uncertainty Estimator](17852-ray-based-volumetric-uncertainty-estimator.md) (which models aleatoric uncertainty) by their direct reliance on explicit depth or flow supervision signals.

## Shared principles
All depth-regularizing losses operate on the principle that the rendered depth $\hat{\textbf{D}}$ must align with a geometric prior $\textbf{D}_{prior}$. The rendered depth is computed via standard volumetric integration along rays $\textbf{r}$:

$$
\hat{\textbf{D}}(\textbf{r}) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) z_i
$$

where $z_i$ is the sample position, $\sigma_i$ is the volume density, $\delta_i$ is the distance between samples, and $T_i$ is the accumulated transmittance [S1]. The loss function $\mathcal{L}_{depth}$ typically takes the form:

$$
\mathcal{L}_{depth} = \sum_{\textbf{r} \in \mathcal{R}} \mathcal{L}_{reg}(\hat{\textbf{D}}(\textbf{r}), \textbf{D}_{prior}(\textbf{r}))
$$

where $\mathcal{L}_{reg}$ is a robust loss metric (e.g., Huber or L1) and $\textbf{D}_{prior}$ is derived from either stereo matching or optical flow projection [S1].

## Subtopics

### [Depth-Supervised EndoNeRF Loss](15276-depth-supervised-endonerf-loss.md)
A training objective that combines photometric color loss with a depth-supervised loss using estimated stereo depth to accelerate convergence and remove the need for a depth refinement module. It sets itself apart from standard NeRF losses by explicitly constraining the rendered depth to match external stereo-matching estimates, thereby reducing training time compared to methods like STTR that require iterative depth refinement.

### [Optical Flow Loss](17262-optical-flow-loss.md)
Optical Flow Loss is a regularization term used in differentiable volume rendering and NeRF optimization to supervise the implicit scene representation and camera poses using temporal or spatial optical flow correspondences. It differs from photometric losses by enforcing geometric consistency between adjacent views (or frames) through depth-derived flow, making it particularly effective for weakly supervised scenarios or scenes with textureless regions where color-based matching fails.

### Depth Smoothness Loss
A regularization loss applied to depth patches rendered from sampled unseen camera poses to improve scene geometry based on the piecewise-smooth hypothesis [S1]. It is defined as:

$$
\mathcal{L}_{DS} = \frac{\lambda_{DS}}{|\mathcal{R}|} \sum_{\textbf{r}_c \in \mathcal{R}} \sum_{(x,y)} \|\nabla D(x, y)\|_2
$$

where $\mathcal{R}$ is the set of rays cast from sampled poses, $D$ is the depth patch centered at $\textbf{r}_c$, and $\nabla D$ is computed using finite differences [S1]. The weight $\lambda_{DS}$ is set to $5 \cdot 10^{-4}$ for coarse-stage training and $10^{-5}$ for fine-stage training in bounded inward-facing scenes, and $5 \cdot 10^{-4}$ for forward-facing scenes [S1]. Camera poses are sampled from a uniform distribution over a hemisphere (inward-facing) or 2D plane (forward-facing), or via interpolation if the distribution is unknown [S1].

## Background
The concept of using depth regularization in NeRF stems from the observation that photometric loss alone often leads to ambiguous geometry, particularly in textureless regions. RegNeRF [Niemeyer et al., 2022] introduced depth smoothness loss on sampled viewpoints to enforce piecewise-smooth geometry [S1]. Subsequent works like VGOS [S1] and others  extended this by incorporating external stereo-matching estimates (e.g., Unimatch) and optical flow constraints to provide stronger geometric priors, especially in dynamic or sparse-view settings. These methods are widely used in medical imaging (e.g., endoscopic surgery videos) and novel view synthesis from sparse inputs.

## Sources
[S1] VGOS: Voxel Grid Optimization for View Synthesis from Sparse Inputs, https://doi.org/10.24963/ijcai.2023/157, fetched 2026-10-26
