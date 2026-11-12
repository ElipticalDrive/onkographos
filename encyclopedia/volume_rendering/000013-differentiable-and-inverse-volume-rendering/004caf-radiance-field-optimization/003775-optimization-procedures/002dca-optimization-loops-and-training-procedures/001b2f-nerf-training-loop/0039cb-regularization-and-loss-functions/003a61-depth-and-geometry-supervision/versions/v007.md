# Depth and Geometry Supervision

Also called: Geometric Regularization, Depth-Supervised NeRF

## Overview
Depth and Geometry Supervision methods accelerate Neural Radiance Field (NeRF) optimization and improve geometric accuracy by incorporating external depth estimates or geometric priors into the training loss. Unlike standard photometric reconstruction, which suffers from shape-radiance ambiguity, these methods constrain the density field to align with surface geometry. They differ from sibling methods like [Diffusion-Regularized NeRF Training](7381-diffusion-regularized-nerf-training.md) by using explicit geometric constraints (depth maps or level sets) rather than learned generative priors, and from [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md) by focusing on supervising the ray termination distribution or sampling bounds rather than just density smoothness.

## Shared principles
All methods in this family share the fundamental structure of an augmented loss function. The total loss $\mathcal{L}_{\text{total}}$ is a weighted sum of the standard photometric reconstruction loss and one or more regularization terms $\mathcal{L}_k$:

$$ \mathcal{L}_{\text{total}} = \mathcal{L}_{\text{photo}} + \sum_k \lambda_k \mathcal{L}_k $$

where $\lambda_k$ are hyperparameters controlling the strength of each regularizer [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]. The photometric loss is typically the Mean Squared Error (MSE) between the rendered color $\mathbf{C}$ and the ground truth image $I$:

$$ \mathcal{L}_{\text{photo}} = || \mathbf{C} - I ||_2^2 $$

Regularization terms $\mathcal{L}_k$ operate on the density $\sigma$ and color $\mathbf{c}$ fields or the rendered weights $w_i$ to constrain the solution space [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]. Common strategies include enforcing spatial smoothness, ensuring geometric consistency across views, or aligning the output distribution with learned priors.

## Subtopics

### [Guided Optimization of NeRF](7496-guided-optimization-of-nerf.md)
Guided Optimization of NeRF mitigates the shape-radiance ambiguity inherent in Neural Radiance Fields (NeRF) for indoor scenes by restricting the sampling of points along camera rays to ranges derived from adapted monocular depth priors. Unlike standard NeRF, which samples uniformly between sparse reconstruction bounds, this method uses multi-view consistency checks to define adaptive near and far bounds, ensuring sampled points concentrate around plausible geometry. It further refines the output depth map using confidence scores derived from RGB reconstruction errors and plane bilateral filtering.

### [Single-view NeRF Training with Depth Map](8679-single-view-nerf-training-with-depth-map.md)
This method addresses the under-constrained optimization problem of reconstructing a 3D scene from a single panoramic input by initializing the NeRF geometry using estimated depth maps. It sets itself apart from sibling methods like [Depth-Supervised NeRF](8687-depth-supervised-nerf.md) by explicitly performing a deformable alignment of multiple perspective depth estimates into a canonical panorama before training, rather than applying depth loss directly to raw estimates or using multi-view consistency. It builds on [Depth and Geometry Supervision](8727-depth-and-geometry-supervision.md) by providing a robust initialization strategy for visible regions.

### [Depth-Supervised NeRF](8687-depth-supervised-nerf.md)
Depth-Supervised NeRF (DS-NeRF) is a training procedure that accelerates Neural Radiance Field optimization and improves quality with sparse input views by incorporating depth supervision from sparse 3D keypoints generated during Structure-from-Motion (SfM) preprocessing. Unlike methods that rely on data-driven priors from category-specific training data (e.g., pixelNeRF, metaNeRF) or semantic consistency, DS-NeRF only requires the existence of 3D keypoints, making it flexible and not dependent on training data distribution assumptions. It enforces that the ray termination distribution matches surface priors, reducing overfitting and speeding up training by 2-3x.

### Zero-level set projection
- **Definition**: Refines inaccurate median depth from a pre-trained NeRF, particularly at grazing view angles, by iteratively projecting points onto the zero level-set of the surface using TSDF values [S1].
- **Step**: For each 3D point $p$ on the viewing ray, perform exactly 4 iterative gradient descent steps along the ray direction using the TSDF predicted value to update the point position [S1].
- **Formula**: $p_{new} = p_{old} + \alpha \times \hat{t}$, where $[\hat{t}, \hat{n}, \hat{f}] = \phi(p)$; $p$ is the current 3D point, $\alpha$ is a small constant step size, $[\hat{t}, \hat{n}, \hat{f}]$ are the tangent, normal, and feature outputs from network $\phi$ evaluated at $p$ [S1].
- **Parameter**: $\alpha$: Small constant controlling the step size of the gradient descent update. Value not specified in this section [S1].
- **Pitfall**: Median depth from a pre-trained NeRF becomes inaccurate when the angle between the view direction and the surface is small (grazing angles) [S1].
- **Usage**: Applied in NeRFMeshing as a preprocessing step to initialize the mesh geometry before training the SSAN model [S1].

## Background
The concept of using depth supervision to regularize NeRF training was introduced in Depth-Supervised NeRF (DS-NeRF) [S1]. This approach leverages sparse 3D keypoints from Structure-from-Motion (SfM) to guide the optimization. Later, methods like NeRFMeshing [S1] extended geometric supervision by integrating zero-level set projections for mesh extraction. These techniques address the fundamental limitation of NeRF's shape-radiance ambiguity, where photometric loss alone cannot uniquely determine geometry.

## Sources
[S1] NeRFMeshing: Distilling Neural Radiance Fields into Geometrically-Accurate 3D Meshes, https://doi.org/10.1109/3dv62453.2024.00093, fetched 2026-10-29
