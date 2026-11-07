# Geometric Priors and Depth Integration


## Overview
This family of methods enhances Neural Radiance Fields (NeRF) by integrating explicit geometric constraints—such as Structure-from-Motion (SfM) features, epipolar geometry, or monocular depth priors—into the optimization loop. This addresses the shape-radiance ambiguity and local minima issues inherent in standard NeRF, particularly under sparse-view conditions. It differs from sibling methods like [AdaNeRF](6195-adanerf.md) or [Latent-NeRF](9110-latent-nerf.md) by explicitly enforcing 3D geometric consistency (depth ordering, feature matching) rather than relying solely on appearance-based regularization or latent space compression.

## Shared principles
All methods in this family compute a radiance field function $f_\theta(\mathbf{x}, \mathbf{d})$ that outputs volume density $\sigma$ and color $\mathbf{c}$ for a query point $\mathbf{x}$ and direction $\mathbf{d}$. The final pixel color $\mathbf{C}(\mathbf{r})$ for a ray $\mathbf{r}$ is computed via the standard volume rendering integral:
$$
\mathbf{C}(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i \mathbf{c}_i
$$
where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$.

The shared principle distinguishing this family is the addition of auxiliary loss terms or architectural constraints derived from external geometric priors. These priors typically include:
1.  **Explicit 3D Constraints:** Using SfM matches or epipolar geometry to constrain the 3D coordinates $\mathbf{x}_s$ of points along rays [S1].
2.  **Depth Priors:** Using monocular depth estimators (e.g., DepthAnythingV2) to enforce relative depth ordering or structural consistency [S1].
3.  **Multi-View Consistency:** Enforcing photometric consistency between views using warping based on estimated depth and camera poses [S1].

The optimization minimizes a total loss $L_{total}$ composed of the standard rendering loss $L_{ren}$ and geometric regularization terms $L_{geo}$:
$$
L_{total} = L_{ren} + \sum_k \lambda_k L_{geo}^{(k)}
$$

## Subtopics

### [SfMNeRF](8664-sfmnerf.md)
- **What it is:** A method that integrates Structure-from-Motion (SfM) constraints, including matched feature positions, epipolar geometry, and multi-view photometric consistency, into NeRF training.
- **Distinction:** Explicitly constrains 3D-scene geometry to avoid local minima and reduce shape-radiance ambiguity. Unlike [DASNeRF](8668-dasnerf.md), it relies on explicit geometric priors (SIFT matches, epipolar lines) rather than differentiable surface reconstruction or depth-aware rendering losses alone.
- **When to choose:** Sparse input conditions where standard NeRF fails to reconstruct accurate geometry due to lack of texture or ambiguity.
- **Link:** [SfMNeRF](8664-sfmnerf.md)

### [DASNeRF](8668-dasnerf.md)
- **What it is:** A method for sparse-view 3D reconstruction that integrates accurate monocular depth priors (DepthAnythingV2) and a dynamic sampling mechanism.
- **Distinction:** Enforces relative depth ordering fidelity and structural consistency using depth priors. It employs a per-layer input fusion MLP and dynamic optimal sampling to mitigate lighting artifacts and sampling errors, differing from [SfMNeRF](8664-sfmnerf.md) by using dense depth maps instead of sparse SfM features.
- **When to choose:** Sparse-view scenarios requiring high detail preservation and handling of transparent/reflective objects where SfM features may be sparse.
- **Link:** [DASNeRF](8668-dasnerf.md)

### GeoNeRF
- **What it is:** A model consisting of a geometric morphology reasoner and a renderer [S1]. The reasoner builds a cascaded cost volume for source view sequences. The renderer infers geometric and appearance representations using a Transformer-based attention mechanism and cascaded cost volumes, then renders fine images using volume rendering [S1].
- **Distinction:** Uses a Transformer-based attention mechanism with cascaded cost volumes for geometric reasoning, distinct from the MLP-centric approaches of SfMNeRF and DASNeRF [S1].
- **When to choose:** Complex occlusion reasoning and scenarios requiring high-resolution rendering from source view sequences.
- **Link:** Not given in the sources.

## Background
The concept of integrating geometric priors into NeRF emerged to address the "shape-radiance ambiguity" and poor performance under sparse views. In 2022, Johari et al. proposed GeoNeRF, which uses a geometric morphology reasoner and Transformer-based renderer [S1]. In 2023, Mu et al. proposed a method using SDF and depth maps for implicit reconstruction [S1]. SfMNeRF and DASNeRF further refined these approaches by explicitly incorporating SfM constraints and monocular depth priors (DepthAnythingV2) respectively, to improve sparse-view reconstruction [S1].

## Sources
[S1] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
