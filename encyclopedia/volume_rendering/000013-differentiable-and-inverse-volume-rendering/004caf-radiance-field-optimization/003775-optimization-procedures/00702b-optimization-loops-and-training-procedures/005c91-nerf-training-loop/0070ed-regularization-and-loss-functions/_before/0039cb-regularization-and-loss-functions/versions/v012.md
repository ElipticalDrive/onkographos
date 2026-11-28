# Regularization and Loss Functions


## Overview
Regularization and Loss Functions constrain the optimization of Neural Radiance Fields (NeRFs) to prevent degenerate solutions, such as "floaters" (semi-transparent artifacts) and image copying, particularly in sparse-view or few-shot settings. This family distinguishes itself from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the objective function's mathematical structure rather than the sampling strategy, and from [Training Configuration](2332-training-configuration.md) by defining specific penalty terms rather than hyperparameter schedules. It addresses the shape-radiance ambiguity inherent in volume rendering by introducing geometric, structural, or generative priors into the loss landscape.

## Shared principles
The total training loss $\mathcal{L}_{\text{total}}$ is a weighted sum of a standard photometric reconstruction loss and one or more regularization terms $\mathcal{L}_k$:

$$ \mathcal{L}_{\text{total}} = \mathcal{L}_{\text{photo}} + \sum_k \lambda_k \mathcal{L}_k $$

where $\lambda_k$ are hyperparameters controlling the strength of each regularizer [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]. The photometric loss is typically the Mean Squared Error (MSE) between the rendered color $\mathbf{C}$ and the ground truth image $I$:

$$ \mathcal{L}_{\text{photo}} = || \mathbf{C} - I ||_2^2 $$

Regularization terms $\mathcal{L}_k$ operate on the density $\sigma$ and color $\mathbf{c}$ fields or the rendered weights $w_i$ to constrain the solution space [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]. Common strategies include enforcing spatial smoothness, ensuring geometric consistency across views, or aligning the output distribution with learned priors.

## Subtopics

### [Diffusion-Regularized NeRF Training](7381-diffusion-regularized-nerf-training.md)
Mitigates severe under-constraint in limited-view settings by using a Denoising Diffusion Model (DDM) trained on RGBD patches to regularize geometry and color. It sets itself apart from [RegNeRF](8582-regnerf.md) by using a DDM rather than normalizing flows, and from [Depth and Geometry Supervision](8727-depth-and-geometry-supervision.md) by learning a probabilistic prior rather than using explicit depth sensors.

### [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md)
Prevents degenerate solutions in few-view settings by combining photometric error with three geometric priors: foreground completeness, frustum consistency, and depth distribution compactness. It distinguishes itself from [RegNeRF](8582-regnerf.md) and [Depth and Geometry Supervision](8727-depth-and-geometry-supervision.md) by relying solely on internal ray weights and view frustum overlaps without external depth inputs.

### [RegNeRF](8582-regnerf.md)
Improves view synthesis from sparse views by constraining unobserved regions via a smoothness prior and a normalizing flow prior. It distinguishes itself from [DiffNeRF](8572-diffnerf.md) by approximating spatial gradients via finite differences on patch-based dataloaders rather than automatic differentiation on ray-based representations.

### [Occlusion-Eliminating NeRF Variant](8653-occlusion-eliminating-nerf-variant.md)
Addresses novel view synthesis degradation caused by foreground occlusions by modeling the scene and occluding background as separate components using a dual-MLP architecture. It distinguishes itself from generic regularization methods by using cost volume and depth inconsistency losses to disentangle foreground objects from background artifacts.

### [Depth and Geometry Supervision](8727-depth-and-geometry-supervision.md)
Accelerates optimization and improves geometric accuracy by incorporating external depth estimates or structural constraints. It is distinguished from [Diffusion-Regularized NeRF Training](7381-diffusion-regularized-nerf-training.md) by relying on explicit depth values rather than learned generative priors, and from [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md) by focusing on alignment with specific depth targets.

### [Specific Loss Formulations](10636-specific-loss-formulations.md)
Encompasses precise mathematical formulations for regularization and reconstruction losses, such as patch-based MSE or adaptive uncertainty weighting, to address spatial misalignment and few-shot generalization.

### [Diffusion-based Regularization for NeRF](10661-diffusion-based-regularization-for-nerf.md)
Incorporates a denoising diffusion model as a prior for density and color fields to mitigate interference factors in limited-view scenarios. It builds on [Regularization and Loss Functions](8586-regularization-and-loss-functions.md) by adding a gradient term derived from the score of the diffusion prior, preventing the disappearance of fine structures.

### [Multiplex Training via S3IM](10894-multiplex-training-via-s3im.md)
Augments standard photometric losses with a Stochastic Structural SIMilarity (S3IM) term to improve reconstruction quality for complex details. It distinguishes itself from standard SSIM by capturing non-local structural information from distant pixels across different images via stochastic patching.

### [S3IM (Stochastic Structural SIMilarity)](10895-s3im-stochastic-structural-similarity.md)
Captures nonlocal structural similarity by processing multiple data points as a collective set. Unlike conventional pixel-wise losses, S3IM cannot be expressed as the sum of any loss computed over individual pixels, enforcing global structural consistency at nearly no additional computational cost.

### Cache Consistency Loss
- [S1] A loss supervising the physically-based model to constrain the appearance of the NeRF cache by matching diffuse and specular colors.
- [S1] Supervise cache diffuse color $c_{\text{cache}}(\mathbf{x})$ and specular color $c_{\text{cache}}(\mathbf{x}, \omega_i)$ to match physically-based model colors $c_{\text{phys}}(\mathbf{x})$ and $c_{\text{phys}}(\mathbf{x}, \omega_i)$.
- [S1] Predict an additional output from the cache $c_{\text{irradiance}}(\mathbf{x}, \omega_o)$ and supervise it to match irradiance from the physically-based model (computed by setting BRDF to perfectly Lambertian with $a(\mathbf{x}) = 1$).

### Emptiness Loss on Weights
- [S2] A ray regularization technique that applies emptiness loss to the NeRF weight $w_i$.
- [S2] Fails because $w_i$ in NeRF is a biased geometry proxy.
- [S2] Achieves 19.33 PSNR on DTU 3-view, performing worse than Model-C (20.01) and AR-NeRF (20.07).

### NeRF Pre-training (Density and Color)
- [S3] Training the density net $\Theta_\sigma$ and color branch $\Theta_c$ using posed RGB images and an appearance loss, orthogonal to the Instance Field component.
- [S3] Appearance loss: $L_p = \frac{1}{2|R|} \sum_{r \in R} \| \hat{C}(r) - C(r) \|^2$, where $R$ are sampled rays in a training batch, $C(r)$ is ground truth RGB, and $\hat{C}(r)$ is predicted RGB.
- [S3] $\Theta_\sigma$: Density network parameters.
- [S3] $\Theta_c$: Color branch network parameters.

## Background
The need for regularization in NeRF optimization arises from the ill-posed nature of inverse volume rendering, where multiple density-color configurations can produce the same 2D projections. Early works like [RegNeRF](8582-regnerf.md) introduced smoothness and normalizing flow priors to handle sparse views [S1]. Subsequent research explored geometric baselines [7475-geometric-baseline-regularization.md], diffusion models [7381-diffusion-regularized-nerf-training.md, 10661-diffusion-based-regularization-for-nerf.md], and structural similarity metrics [10894-multiplex-training-via-s3im.md, 10895-s3im-stochastic-structural-similarity.md] to improve generalization and geometric fidelity.

## Sources
[S1] Flash Cache: Reducing Bias in Radiance Cache Based Inverse Rendering, https://doi.org/10.1007/978-3-031-73390-1_2, fetched 2026-10-27
[S2] Few-shot NeRF by Adaptive Rendering Loss Regularization, https://core.ac.uk/download/708671746.pdf, fetched 2026-10-30
[S3] Instance Neural Radiance Field, https://doi.org/10.1109/ICCV51070.2023.00079, fetched 2026-10-29
