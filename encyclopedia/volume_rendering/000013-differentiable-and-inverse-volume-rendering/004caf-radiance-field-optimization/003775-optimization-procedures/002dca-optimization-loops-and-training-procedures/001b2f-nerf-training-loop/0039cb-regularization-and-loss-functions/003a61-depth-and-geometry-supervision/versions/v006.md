# Depth and Geometry Supervision


## Overview
Depth and Geometry Supervision methods accelerate Neural Radiance Field (NeRF) optimization and improve geometric accuracy in sparse-view or single-view settings by incorporating explicit depth constraints into the training loss. Unlike sibling methods that rely on semantic priors, diffusion models, or occlusion elimination, this family explicitly minimizes the discrepancy between rendered ray termination distributions (or depths) and external depth estimates, such as those from Structure-from-Motion (SfM) keypoints or monocular estimators.

## Shared principles
All methods in this family share the fundamental structure of an augmented loss function. The total loss $\mathcal{L}_{\text{total}}$ is a weighted sum of the standard photometric reconstruction loss and one or more regularization terms $\mathcal{L}_k$:

$$ \mathcal{L}_{\text{total}} = \mathcal{L}_{\text{photo}} + \sum_k \lambda_k \mathcal{L}_k $$

where $\lambda_k$ are hyperparameters controlling the strength of each regularizer [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]. The photometric loss is typically the Mean Squared Error (MSE) between the rendered color $\mathbf{C}$ and the ground truth image $I$:

$$ \mathcal{L}_{\text{photo}} = || \mathbf{C} - I ||_2^2 $$

Regularization terms $\mathcal{L}_k$ operate on the density $\sigma$ and color $\mathbf{c}$ fields or the rendered weights $w_i$ to constrain the solution space [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]. Common strategies include enforcing spatial smoothness, ensuring geometric consistency across views, or aligning the output distribution with learned priors.

## Subtopics

### [Guided Optimization of NeRF](7496-guided-optimization-of-nerf.md)
Guided Optimization of NeRF mitigates the shape-radiance ambiguity inherent in Neural Radiance Fields (NeRF) for indoor scenes by constraining the sampling of points along camera rays using adapted monocular depth priors [7496-guided-optimization-of-nerf.md]. Unlike standard NeRF, which samples uniformly between sparse reconstruction bounds, this method calculates adaptive near and far bounds based on multi-view geometric consistency errors, ensuring sampled points are concentrated around plausible geometry [7496-guided-optimization-of-nerf.md]. It further refines the output by computing per-pixel confidence scores from RGB reconstruction errors to weight a plane bilateral filter on the rendered depth map [7496-guided-optimization-of-nerf.md].

### [Single-view NeRF Training with Depth Map](8679-single-view-nerf-training-with-depth-map.md)
This method addresses the under-constrained optimization problem of reconstructing a 360-degree scene from a single input panorama by initializing the geometry of visible regions using monocular depth estimation [8679-single-view-nerf-training-with-depth-map.md]. It differs from [Depth-Supervised NeRF](8687-depth-supervised-nerf.md) by specifically handling the scale ambiguity and field-of-view limitations of monocular estimators through a deformable alignment step to a canonical panorama depth map before NeRF training, and from [Guided Optimization of NeRF](7496-guided-optimization-of-nerf.md) by focusing on single-view panoramic reconstruction rather than multi-view consistency or object removal [8679-single-view-nerf-training-with-depth-map.md].

### [Depth-Supervised NeRF](8687-depth-supervised-nerf.md)
Depth-Supervised NeRF (DS-NeRF) is a training procedure that accelerates Neural Radiance Field optimization and improves quality with sparse input views by incorporating depth supervision from sparse 3D keypoints generated during Structure-from-Motion (SfM) preprocessing [8687-depth-supervised-nerf.md]. Unlike methods relying on data-driven priors from other scenes or semantic consistency, DS-NeRF enforces geometric plausibility by minimizing the KL divergence between the rendered ray termination distribution and the noisy depth distribution of the keypoints, reducing overfitting and speeding up training by 2-3x [8687-depth-supervised-nerf.md].

### Zero-level set projection
#### definition
- [S1] Refines inaccurate median depth from a pre-trained NeRF, particularly at grazing view angles, by iteratively projecting points onto the zero level-set of the surface using TSDF values.
#### step
- [S1] For each 3D point p on the viewing ray, perform exactly 4 iterative gradient descent steps along the ray direction using the TSDF predicted value to update the point position.
#### formula
- [S1] $p_{new} = p_{old} + \alpha \times \hat{t}$, where $[\hat{t}, \hat{n}, \hat{f}] = \phi(p)$; $p$ is the current 3D point, $\alpha$ is a small constant step size, $[\hat{t}, \hat{n}, \hat{f}]$ are the tangent, normal, and feature outputs from network $\phi$ evaluated at $p$.
#### parameter
- [S1] $\alpha$: Small constant controlling the step size of the gradient descent update. Value not specified in this section.
#### pitfall
- [S1] Median depth from a pre-trained NeRF becomes inaccurate when the angle between the view direction and the surface is small (grazing angles).
#### usage
- [S1] Applied in NeRFMeshing as a preprocessing step to initialize the mesh geometry before training the SSAN model.

## Background
The concept of using depth supervision to regularize NeRF training was introduced in [Depth-Supervised NeRF](8687-depth-supervised-nerf.md), which demonstrated that aligning the ray termination distribution with sparse SfM keypoints significantly accelerates convergence and improves geometry [8687-depth-supervised-nerf.md]. Subsequent work expanded this to single-view scenarios using deformable alignment of monocular depth estimates [8679-single-view-nerf-training-with-depth-map.md] and to indoor scenes using adaptive sampling bounds derived from multi-view consistency [7496-guided-optimization-of-nerf.md]. The zero-level set projection technique was later introduced in [NeRFMeshing](S1) to refine mesh extraction from NeRF outputs by correcting inaccuracies at grazing angles [S1].

## Sources
[S1] NeRFMeshing: Distilling Neural Radiance Fields into Geometrically-Accurate 3D Meshes, https://doi.org/10.1109/3dv62453.2024.00093, fetched 2026-10-29
