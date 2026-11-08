# Loss Function Configuration

Also called: Loss Weighting Scheme, Training Loss Weights

## Overview
Loss Function Configuration defines the scalar weights applied to individual loss components within the total objective function during radiance-field optimization. It balances the influence of photometric reconstruction, perceptual metrics, geometric consistency, and regularization terms to stabilize training and improve convergence. It differs from [Sparse Depth Consistency Loss](7294-sparse-depth-consistency-loss.md) and [Photometric and Image Reconstruction Losses](9789-photometric-and-image-reconstruction-losses.md) by focusing exclusively on the relative scaling factors rather than the mathematical formulation of the losses themselves. It complements [Loss Weight Configuration](7883-loss-weight-configuration.md) by providing specific empirical values derived from experimental validation for dynamic and static scenes.

## How it works
The total loss $\mathcal{L}$ is a weighted sum of multiple components. The configuration process involves selecting appropriate weights $w_i$ for each term $\mathcal{L}_i$ based on the training stage (coarse vs. fine) and scene characteristics (static vs. dynamic).

### Inputs and Outputs
- **Inputs**: Training stage identifier (coarse/fine), scene motion magnitude (static/large), and the set of active loss components.
- **Outputs**: A dictionary or configuration object mapping loss keys to their respective scalar weights.

### Data Structures
The configuration is typically stored as a map of string keys to floating-point numbers.
- **Key**: Loss component identifier (e.g., `ptc`, `bg`, `d_norm`, `d_tv`, `LPIPS`).
- **Value**: Non-negative float representing the weight.

### Steps
1. **Initialize Base Weights**: Load default weights for the current training stage.
2. **Adjust for Scene Dynamics**: If the scene exhibits large motion, reduce the weight of deformation-related regularization terms to prevent over-smoothing or instability.
3. **Apply Stage-Specific Scaling**: Use lower weights for auxiliary losses in the fine stage to prioritize precise photometric alignment over global consistency.
4. **Validate Configuration**: Ensure weights sum to a reasonable magnitude or are normalized if the loss components have vastly different scales.

### Formulas
The general form for the total loss with configured weights is:
$$ \mathcal{L} = \mathcal{L}_{photo} + w_{ptc} \cdot \mathcal{L}_{ptc} + w_{bg} \cdot \mathcal{L}_{bg} + w_{d\_norm} \cdot \mathcal{L}_{d\_norm} + w_{d\_tv} \cdot \mathcal{L}_{d\_tv} $$
where:
- $\mathcal{L}_{photo}$: Photometric reconstruction loss.
- $\mathcal{L}_{ptc}$: Point consistency or perceptual term.
- $\mathcal{L}_{bg}$: Background regularization.
- $\mathcal{L}_{d\_norm}$: Deformation norm regularization.
- $\mathcal{L}_{d\_tv}$: Total variation regularization on deformation.
- $w_{ptc}, w_{bg}, w_{d\_norm}, w_{d\_tv}$: Configurable scalar weights.

For SplatVoxel, the loss is configured as:
$$ \mathcal{L} = \mathcal{L}_{\text{MSE}}(\hat{I}, I) + \lambda \mathcal{L}_{\text{LPIPS}}(\hat{I}, I) $$
where $\lambda$ is the LPIPS weighting factor.

### Parameters and Typical Values
**Neural Deformable Voxel Grid (Dynamic Scenes) [S1]:**
- **Coarse Stage**:
  - $w_{ptc} = 0.1$
  - $w_{bg} = 0.01$
  - $w_{d\_norm} = 0.1$
  - $w_{d\_tv} = 1$
- **Fine Stage**:
  - $w_{ptc} = 0.01$
  - $w_{bg} = 0.001$
  - $w_{d\_norm} = 0.01$
  - $w_{d\_tv} = 1$
- **Large Motion Adjustment** (e.g., 'Hell Warrior', 'Jumping Jacks', 'T-Rex'):
  - $w_{d\_norm}$ is reduced to $0.01$ (coarse) and $0.001$ (fine).

**SplatVoxel (Static/Feed-Forward) [S3]:**
- **Multi-view Transformer**: $\lambda = 0.5$
- **Voxel Transformer**: $\lambda = 4.0$
- **Batch Size**: 128
- **Total Iterations**: 300K (200K Stage 1, 100K Stage 2)

### Edge Cases and Pitfalls
- **Large Motion Scenes**: Using standard deformation weights on scenes with large motion can lead to poor reconstruction. The weight $w_{d\_norm}$ must be explicitly reduced [S1].
- **Precision**: Weights are typically small floats ($<1$) for auxiliary terms to prevent them from dominating the photometric loss.
- **Stage Transitions**: Weights must change between coarse and fine stages; failing to reduce auxiliary weights in the fine stage can hinder convergence to high-frequency details [S1].

## Subtopics

### Hybrid Gradients
A gradient computation strategy that combines RGBXY derivatives and standard derivatives to leverage global matching in early stages and precise local alignment in late stages [S2]. It uses RGBXY derivatives (Eq. 15) for the first 75% of optimization iterations and switches to standard derivatives (Eq. 5) for the last 25% [S2]. This approach mitigates the inaccuracy of approximated optimal transport when initial and target shapes are already well-aligned [S2].

### Two-Stage Photometric and Perceptual Training Strategy
A training protocol for SplatVoxel feed-forward networks that uses photometric loss with perceptual weighting [S3]. Stage 1 trains the multi-view Transformer backbone for 200K iterations. Stage 2 jointly fine-tunes both multi-view and voxel Transformers for an additional 100K iterations [S3]. The LPIPS weight $\lambda$ is set to 0.5 for the multi-view transformer and 4.0 for the voxel transformer [S3].

## Operating characteristics
- **Time Complexity**: Configuration itself is $O(1)$, but the choice of weights affects convergence speed. Lower weights on auxiliary losses in the fine stage may require more iterations to converge but yield higher quality [S1].
- **Memory Complexity**: Negligible; only stores scalar weights.
- **Performance**: Proper weighting significantly impacts PSNR and training stability. For dynamic scenes, reducing $w_{d\_norm}$ prevents artifacts in high-motion regions [S1]. For feed-forward models, higher LPIPS weights on voxel transformers improve perceptual quality [S3].
- **Strengths**: Simple to implement; highly effective at balancing competing objectives.
- **Limitations**: Requires empirical tuning per dataset and scene type; not automatically adaptive.

## Background
The concept of weighted loss functions is fundamental in differentiable rendering. Specific configurations for dynamic view synthesis were detailed in the Neural Deformable Voxel Grid paper [S1], which introduced stage-specific weighting for deformation regularization. The hybrid gradient strategy was proposed in the context of RGBXY derivatives for optimal transport-based rendering [S2]. The two-stage perceptual training strategy was introduced for SplatVoxel to improve feed-forward 3D Gaussian Splatting [S3].

## Sources
[S1] Neural Deformable Voxel Grid for Fast Optimization of Dynamic View Synthesis, https://doi.org/10.1007/978-3-031-26319-4_27, fetched 2026-10-26
[S2] Differentiable Rendering Using RGBXY Derivatives and Optimal Transport, https://doi.org/10.1145/3550454.3555479, fetched 2026-10-27
[S3] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
