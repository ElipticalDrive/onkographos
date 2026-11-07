# Adaptive Ray Sampling

Also called: Adaptive Sampling, Hierarchical Sampling, Importance Sampling in NeRF

## Overview
Adaptive Ray Sampling refers to a family of strategies in neural radiance field (NeRF) training that dynamically adjust the distribution of sample points along viewing rays to improve reconstruction quality and training efficiency. Unlike uniform sampling, which treats all space along a ray equally, these methods concentrate computational resources in regions of high geometric or photometric complexity, such as object surfaces or high-frequency details [S1, S3, unofficial]. This family is distinct from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing specifically on *where* to sample rather than *how* to integrate, and from [Mip-NeRF 360](3306-mip-nerf-360.md) by typically operating in Euclidean space with point-wise samples rather than conical frustums.

## Shared principles
All methods in this family share the goal of reducing the number of unnecessary network evaluations in empty space or converged regions while increasing density in ambiguous or high-detail areas. They generally operate by:
1.  **Estimating Importance:** Using a proxy metric (e.g., coarse network weights, rendering error, color/depth variance, or occupancy) to identify "important" regions along a ray or in image space.
2.  **Resampling/Selection:** Generating new sample points or selecting rays based on this importance distribution, often using inverse transform sampling or stochastic selection.
3.  **Integration:** Rendering the final color using the adaptively sampled points.

Common mathematical components include the volume rendering equation:
$$ \mathbf{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i $$
where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the transmittance, $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ is the opacity, and $\delta_i$ is the distance between samples [S4, unofficial]. The adaptation lies in the selection of $N$ points and their positions $t_i$.

## Subtopics

### [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md)
A hierarchical strategy using two MLPs (coarse and fine). The coarse network uniformly samples the ray to identify opaque regions via accumulated weights. The fine network then densely samples these regions using inverse transform sampling on the coarse weights. It is chosen when high-frequency detail reconstruction is critical and uniform sampling is too sparse.

### [Probability-based Ray Sampling with Quadtree Subdivision](7481-probability-based-ray-sampling-with-quadtree-subdivision.md)
Uses a quadtree structure on image patches. It calculates rendering error per patch; if error is low, the patch is "marked" and sampled sparsely. If error is high, it subdivides into four children for denser sampling. It is chosen for accelerating training in large scenes by reducing ray counts in converged regions.

### [TermiNeRF](8828-terminerf.md)
Trains a lightweight sampling network to predict a probability distribution $P(t)$ directly from ray parameters, bypassing the need for a coarse pass or spatial data structures. It uses centered-logarithmic sampling bins and matches predicted weights to ground-truth rendering weights derived from a pre-trained color network. It is chosen for robustness in geometrically ambiguous regions and reducing color network evaluations.

### [Pixel Region and Depth Guided Ray Sampling](8866-pixel-region-and-depth-guided-ray-sampling.md)
Biases ray selection in 2D image space based on color and depth variance. Rays are cast more frequently from pixels with high gradient magnitude in color or estimated depth. It is chosen to accelerate convergence and improve geometry perception by focusing on complex pixel regions rather than refining along single rays.

### [Two-Stage Sampling with Occupancy Grid and Nonlinear Function](9146-two-stage-sampling-with-occupancy-grid-and-nonlinear-functio.md)
Combines an occupancy grid to skip empty space (coarse) with a nonlinear piecewise exponential function to bias samples toward surfaces within occupied cells (fine). It is chosen for implicit surface reconstruction (NeuS) where SDF gradients are used, offering deterministic biasing compared to stochastic inverse transform sampling.

## Background
The concept of adaptive sampling in volume rendering predates NeRF, originally developed to handle sparse data or complex geometry efficiently [S1, S3, unofficial]. In the context of NeRF, the "Coarse-to-Fine" approach was introduced by Mildenhall et al. in the original NeRF paper (2020) to address the inefficiency of uniform sampling [S3, unofficial]. Subsequent works like TermiNeRF (2022) and various quadtree-based methods (2021-2022) expanded this to include learned sampling distributions and spatial hierarchies to further reduce computational cost [S1, S2].

## Sources
[S1] Probability-based Ray Sampling with Quadtree Subdivision notes
[S2] TermiNeRF notes
[S3] Coarse-to-Fine NeRF notes
[S4] Coarse-to-Fine NeRF Pseudocode and Formulas
