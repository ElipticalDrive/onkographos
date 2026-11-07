# Depth and Geometry Supervision

Also called: Depth-Supervised NeRF, Guided Optimization, Single-View Depth NeRF

## Overview
Depth and Geometry Supervision accelerates Neural Radiance Field (NeRF) optimization and improves geometric accuracy by incorporating explicit depth constraints into the training loss, addressing the shape-radiance ambiguity inherent in volume rendering [S1, 8687-depth-supervised-nerf.md]. Unlike standard NeRF, which relies solely on photometric consistency, this family of methods enforces that the ray termination distribution or sampling bounds align with external depth priors, such as Structure-from-Motion (SfM) keypoints, monocular estimates, or multi-view consistent depth maps [S1, 8687-depth-supervised-nerf.md, 7496-guided-optimization-of-nerf.md, 8679-single-view-nerf-training-with-depth-map.md].

## Shared principles
The core mechanism involves augmenting the standard photometric reconstruction loss ($\mathcal{L}_{\text{photo}}$) with additional terms that enforce geometric plausibility, spatial smoothness, or statistical priors. The total loss $\mathcal{L}_{\text{total}}$ is generally a weighted sum:
$$ \mathcal{L}_{\text{total}} = \mathcal{L}_{\text{photo}} + \sum_k \lambda_k \mathcal{L}_k $$
where $\lambda_k$ are hyperparameters controlling the strength of each regularizer [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]. The photometric loss is typically the Mean Squared Error (MSE) between the rendered color $\mathbf{C}$ and the ground truth image $I$:
$$ \mathcal{L}_{\text{photo}} = || \mathbf{C} - I ||_2^2 $$
Regularization terms $\mathcal{L}_k$ operate on the density $\sigma$ and color $\mathbf{c}$ fields or the rendered weights $w_i$ to constrain the solution space [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md].

## Subtopics

### [Guided Optimization of NeRF](7496-guided-optimization-of-nerf.md)
Mitigates shape-radiance ambiguity in indoor scenes by restricting sampling domain along camera rays using adapted monocular depth priors. It computes adaptive near and far bounds based on multi-view geometric consistency errors, ensuring sampled points concentrate around plausible geometry.

### [Single-view NeRF Training with Depth Map](8679-single-view-nerf-training-with-depth-map.md)
Addresses the under-constrained optimization of reconstructing a 360-degree scene from a single panoramic input by initializing geometry using a 360-degree depth estimation method. It explicitly handles scale ambiguity and misalignment inherent in projecting monocular depth estimates from multiple perspectives onto a canonical panorama.

### [Depth-Supervised NeRF](8687-depth-supervised-nerf.md)
Accelerates NeRF optimization with sparse input views by incorporating depth supervision from sparse 3D keypoints generated during Structure-from-Motion (SfM) preprocessing. It enforces that the ray termination distribution matches surface priors derived from SfM points, reducing overfitting and speeding up training.

### Zero-level set projection
- **Definition**: Refines inaccurate median depth from a pre-trained NeRF, particularly at grazing view angles, by iteratively projecting points onto the zero level-set of the surface using TSDF values [S1].
- **Step**: For each 3D point $p$ on the viewing ray, perform exactly 4 iterative gradient descent steps along the ray direction using the TSDF predicted value to update the point position [S1].
- **Formula**: $p_{new} = p_{old} + \alpha \times \hat{t}$, where $[\hat{t}, \hat{n}, \hat{f}] = \phi(p)$; $p$ is the current 3D point, $\alpha$ is a small constant step size, $[\hat{t}, \hat{n}, \hat{f}]$ are the tangent, normal, and feature outputs from network $\phi$ evaluated at $p$ [S1].
- **Parameter**: $\alpha$: Small constant controlling the step size of the gradient descent update. Value not specified in this section [S1].
- **Pitfall**: Median depth from a pre-trained NeRF becomes inaccurate when the angle between the view direction and the surface is small (grazing angles) [S1].
- **Usage**: Applied in NeRFMeshing as a preprocessing step to initialize the mesh geometry before training the SSAN model [S1].

## Background
The concept of using depth supervision to regularize NeRF training was introduced by [S1, 8687-depth-supervised-nerf.md], who demonstrated that incorporating sparse 3D keypoints from SfM pipelines significantly accelerates convergence and improves geometry for sparse-view scenarios. Subsequent work extended this to guided optimization using monocular priors [S1, 7496-guided-optimization-of-nerf.md] and single-view panorama reconstruction [S1, 8679-single-view-nerf-training-with-depth-map.md]. The zero-level set projection technique was developed as part of NeRFMeshing to refine mesh initialization from NeRF outputs [S1].

## Sources
[S1] NeRFMeshing: Distilling Neural Radiance Fields into Geometrically-Accurate 3D Meshes, https://doi.org/10.1109/3dv62453.2024.00093, fetched 2026-10-29
