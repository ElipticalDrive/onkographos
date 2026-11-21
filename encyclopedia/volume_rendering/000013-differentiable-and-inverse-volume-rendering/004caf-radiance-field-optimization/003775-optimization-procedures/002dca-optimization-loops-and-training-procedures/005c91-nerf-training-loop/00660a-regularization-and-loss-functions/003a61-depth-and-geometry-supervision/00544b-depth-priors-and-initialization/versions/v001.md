# Depth Priors and Initialization

Also called: Depth-guided initialization, Monocular depth priors, Planar architectural depth priors

## Overview
This family of methods addresses the ill-posed nature of single-view or sparse-view NeRF optimization by initializing or regularizing the geometry using external depth estimates [8679-single-view-nerf-training-with-depth-map.md, 14220-depth-supervision.md]. It sets itself apart from sibling methods like [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md) by explicitly fusing perspective depth maps into a canonical panorama depth map via deformable alignment before training, rather than relying solely on RGB consistency or post-hoc inpainting [8679-single-view-nerf-training-with-depth-map.md]. It distinguishes itself from [Depth-Supervised NeRF](8687-depth-supervised-nerf.md) by focusing on refining existing radiance-field optimizations through external geometric priors rather than relying solely on multi-view consistency or specific single-view architectures [14220-depth-supervision.md].

## Shared principles
All methods in this family operate within the differentiable volume rendering framework where gradients flow from the loss $\mathcal{L}$ back through the rendered color $\hat{\mathbf{C}}(\mathbf{r})$ and depth $\hat{\mathbf{D}}(\mathbf{r})$ to the network parameters $\theta$ (density $\sigma$ and color $\mathbf{c}$) [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md].

The rendering process computes expected color and depth along rays $r(t) = o + td$ using sample points $t_i$ with weights $w_i = T(t_i)\rho(t_i)$, where $\rho(t_i) = 1 - \exp(-\sigma(\mathbf{r}(t_i))(t_{i+1} - t_i))$ and $T(t_i) = \prod_{j=1}^{i-1} (1 - \rho(t_j))$ [S1]. The rendering weights $w_i$ serve as the probability distribution of absorption along the ray, often denoted as $h(t) = T(t)\sigma(t)$, which is used in distributional geometric losses [S1].

The rendered depth is typically the expected distance along the ray:
$$ \hat{D}(\mathbf{r}) = \sum_{i=1}^N w_i t_i $$
where $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$ are the rendering weights, $t_i$ are sample depths, and $T_i$ is the accumulated transmittance [S1].

Gradients are backpropagated through the differentiable volume rendering process to update the density and color field parameters [S1, 8582-regnerf.md].

## Subtopics

### [Single-view NeRF Training with Depth Map](8679-single-view-nerf-training-with-depth-map.md)
This method initializes the geometry of visible regions in a single-view NeRF using a 360-degree depth estimation pipeline, addressing the under-constrained optimization problem inherent to single-view reconstruction [8679-single-view-nerf-training-with-depth-map.md]. It sets itself apart from sibling methods by explicitly fusing perspective depth maps into a canonical panorama depth map via deformable alignment before NeRF training, rather than relying solely on RGB consistency or post-hoc inpainting [8679-single-view-nerf-training-with-depth-map.md].

### [Depth Supervision](14220-depth-supervision.md)
Depth supervision is a training technique that improves reconstructed geometry and reduces floating artifacts by providing depth priors derived from off-the-shelf monocular depth estimation models [14220-depth-supervision.md]. It distinguishes itself from sibling methods like [Single-view NeRF Training with Depth Map](8679-single-view-nerf-training-with-depth-map.md) by focusing on refining existing radiance-field optimizations through external geometric priors rather than relying solely on multi-view consistency or specific single-view architectures [14220-depth-supervision.md].

### Depth Supervision with Planar Architectural Depth Priors
**Definition:** Incorporates precomputed depth estimates for floor, ceiling, and walls into NeRF training to enforce geometric constraints on textureless planar surfaces [S1].
**Steps:** 1. Compute depth estimates for floor, ceiling, and walls using semantic segmentation. 2. Incorporate these priors into training with MSE and BoundL losses alongside photometric loss. 3. Train until convergence [S1].
**Parameters:** $\delta$ (BoundL tolerance): 1 mm. $\lambda_{color}$: 10 (bedroom), 1 (living room). $\lambda_{depth}$: 10 (bedroom), 1 (living room) [S1].
**Metrics:** RMSE of priors vs ground truth: 2.786 mm (bedroom), 3.201 mm (living room). Converged in 120,000 iterations [S1].
**Comparison:** Faster convergence than photometric-only (120k vs 200k) and patch-regularization (120k vs 280k) methods due to depth supervision enabling quick identification of empty spaces, concentrating sampling on occupied regions, and providing a stronger optimization signal [S1].

## Background
The method originates from the challenge of single-view 360-degree reconstruction, where standard monocular depth estimators fail due to limited field-of-view training data [S1]. The approach was developed to handle scale ambiguity in single-view depth estimation by projecting panoramas into perspective views and performing deformable alignment [S1]. It is used in indoor scenarios to improve geometric consistency for textureless surfaces like walls, floors, and ceilings, where photometric loss alone leads to "floaters" and cloudy artifacts [S1].

## Sources
[S1] Improving Geometric Consistency for 360-Degree Neural Radiance Fields in Indoor Scenarios, https://arxiv.org/abs/2503.13710v1, fetched 2026-10-27
