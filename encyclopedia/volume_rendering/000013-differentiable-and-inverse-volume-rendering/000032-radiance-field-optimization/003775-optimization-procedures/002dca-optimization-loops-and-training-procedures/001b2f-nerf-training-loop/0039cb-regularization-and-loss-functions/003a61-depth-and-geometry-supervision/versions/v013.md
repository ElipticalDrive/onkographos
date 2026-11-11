# Depth and Geometry Supervision

Also called: Geometric Regularization, Depth Priors

## Overview
Depth and Geometry Supervision refers to a family of training objectives that incorporate explicit geometric constraints—such as depth maps, 3D points, or structural priors—into the optimization of Neural Radiance Fields (NeRF) [S1, S5]. While standard NeRF relies solely on photometric consistency, which often leads to shape-radiance ambiguity and artifacts like "floaters" in sparse-view settings, these methods enforce physical plausibility by aligning the rendered density distribution or depth with external or internal geometric signals [S1, S5]. This family is distinguished from sibling methods like [Diffusion-regularized NeRF](7381-diffusion-regularized-nerf.md) or [Semantic Consistency Loss](11955-semantic-consistency-loss-clip-based.md) by its direct reliance on metric or structural geometric data rather than generative priors or semantic embeddings [S1, S5].

## Shared principles
All methods in this family operate within an augmented loss structure where the total loss $\mathcal{L}_{\text{total}}$ is a weighted sum of the standard photometric reconstruction loss and one or more regularization terms $\mathcal{L}_k$ [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]:

$$ \mathcal{L}_{\text{total}} = \mathcal{L}_{\text{photo}} + \sum_k \lambda_k \mathcal{L}_k $$

where $\lambda_k$ are hyperparameters controlling the strength of each regularizer [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]. The photometric loss is typically the Mean Squared Error (MSE) between the rendered color $\mathbf{C}$ and the ground truth image $I$:

$$ \mathcal{L}_{\text{photo}} = || \mathbf{C} - I ||_2^2 $$

Regularization terms $\mathcal{L}_k$ operate on the density $\sigma$ and color $\mathbf{c}$ fields, the rendered weights $w_i$, or the final rendered images to constrain the solution space [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]. The gradients of these terms are backpropagated through the differentiable volume rendering process to update the NeRF parameters [S1, 8582-regnerf.md]. Common strategies include enforcing spatial smoothness, ensuring geometric consistency across views, aligning the output distribution with learned priors, or matching semantic embeddings.

## Subtopics

### [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md)
Geometric Baseline Regularization is a composite loss function designed to prevent degenerate solutions in Neural Radiance Field (NeRF) optimization, particularly in few-view settings where the network might collapse to copying training images. It combines photometric reconstruction with three geometric priors—foreground completeness, frustum consistency, and distribution compactness—to enforce physically plausible scene geometry. Unlike methods requiring external depth maps [8679-single-view-nerf-training-with-depth-map.md] or 3D points [12693-3d-point-supervision-loss.md], it relies solely on internal consistency constraints derived from the rendering process itself.

### [Guided Optimization of NeRF](7496-guided-optimization-of-nerf.md)
Guided Optimization of NeRF mitigates the shape-radiance ambiguity inherent in Neural Radiance Fields (NeRF), particularly in poorly textured indoor environments where standard sampling fails to recover correct geometry despite accurate RGB synthesis. It sets itself apart from sibling methods by explicitly using adapted monocular depth priors to constrain the spatial range of stratified sampling along camera rays, rather than relying solely on geometric regularization losses or indirect distribution alignment. This approach builds on [Depth and Geometry Supervision](8727-depth-and-geometry-supervision.md) by integrating depth guidance directly into the forward sampling process.

### [Single-view NeRF Training with Depth Map](8679-single-view-nerf-training-with-depth-map.md)
This method addresses the under-constrained optimization problem of reconstructing a 360-degree scene from a single panoramic input by initializing the geometry of visible regions using a 360-degree depth estimation prior. It sets itself apart from sibling methods like [Depth-Supervised NeRF](8687-depth-supervised-nerf.md) by explicitly handling the scale ambiguity and misalignment inherent in projecting a single panorama into multiple perspective views for depth estimation, rather than relying on multi-view consistency or direct monocular depth supervision on the final rendered rays.

### [Depth-Supervised NeRF](8687-depth-supervised-nerf.md)
Depth-Supervised NeRF (DS-NeRF) is a training procedure that accelerates Neural Radiance Field optimization and improves quality with sparse input views by incorporating depth supervision from sparse 3D keypoints. It solves the problem of NeRF overfitting to incorrect geometries when insufficient views are available by enforcing that the rendered ray termination distribution matches surface priors derived from Structure-from-Motion (SfM). Unlike methods relying on data-driven priors from other scenes or semantic consistency, DS-NeRF requires no additional training and generalizes to any scene where SfM succeeds, making it a flexible, "free" supervisory signal.

### [Depth Structural Consistency Regularization](9371-depth-structural-consistency-regularization.md)
Depth Structural Consistency Regularization (RDSC) is a geometric regularization term used in Neural Radiance Field (NeRF) training to enforce local depth continuity by aligning rendered depth values with structural priors from external depth maps, such as those generated by DepthAnythingV2. It distinguishes itself from sibling methods like [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md) by explicitly modeling the continuity of neighboring depth pixels using a k-nearest neighbors (KNN) approach and a boundary threshold, thereby preserving sharp structural edges while smoothing noise.

### [Statistical Depth Refinement](10185-statistical-depth-refinement.md)
Statistical Depth Refinement is a regularization technique used during NeRF training to prevent the model from overfitting to corrupt stereo depth maps caused by fuzzy pixels and specular highlights. It addresses the problem where direct supervision on noisy depth leads to abrupt geometric artifacts by replacing statistically identified corrupt depth values with smoother depth estimates from an underfitting model. It differs from standard depth supervision [8679-single-view-nerf-training-with-depth-map.md] by actively masking out unreliable depth pixels rather than minimizing error on all pixels, and differs from distribution-based supervision [8687-depth-supervised-nerf.md] by using a hard patching mechanism based on residual quantiles rather than soft distribution alignment.

### [Novel View Regularization](10806-novel-view-regularization.md)
Novel View Regularization mitigates the shape-radiance ambiguity in Neural Radiance Fields (NeRF) when training data is sparse by injecting synthetic rays into the optimization loop. Unlike methods that rely on external depth priors or semantic labels, this approach enforces internal consistency by ensuring the model generalizes to unseen viewpoints during training. It sets itself apart from siblings like [Depth-Supervised NeRF](8687-depth-supervised-nerf.md) by requiring no ground-truth depth maps, instead using the network's own predictions on novel rays as a self-supervisory signal.

### [3D Point Supervision Loss](12693-3d-point-supervision-loss.md)
A training loss that minimizes the Euclidean distance between 3D points backprojected from rendered depth and known ground-truth 3D points, enabling robust camera pose registration in sparse-view regimes. Unlike direct depth supervision [8679-single-view-nerf-training-with-depth-map.md], which often fails to refine poses and leads to disjoint local solutions, this method enforces a global geometric connection between training images and scene geometry.

### Coarse-Fine Consistency Loss
- **Definition**: A loss function that enforces consistency between depth estimates of the coarse and fine NeRF models to prevent ineffective hierarchical sampling and blur [S1].
- **Formula**: $L_{cfc} = \mathbf{1}_{\{m_f=1\}} \odot ||z_c - \nabla(z_f)||^2 + \mathbf{1}_{\{m_c=1\}} \odot ||\nabla(z_c) - z_f||^2$, where $z_c$ and $z_f$ are coarse and fine depths, and $m_c/m_f$ are reliability masks determined by reprojection [S1].

### Zero-level set projection
- **Definition**: Refines inaccurate median depth from a pre-trained NeRF, particularly at grazing view angles, by iteratively projecting points onto the zero level-set of the surface using TSDF values [S2].
- **Step**: For each 3D point $p$ on the viewing ray, perform exactly 4 iterative gradient descent steps along the ray direction using the TSDF predicted value to update the point position [S2].
- **Formula**: $p_{new} = p_{old} + \alpha \times \hat{t}$, where $[\hat{t}, \hat{n}, \hat{f}] = \phi(p)$; $p$ is the current 3D point, $\alpha$ is a small constant step size, $[\hat{t}, \hat{n}, \hat{f}]$ are the tangent, normal, and feature outputs from network $\phi$ evaluated at $p$ [S2].
- **Parameter**: $\alpha$: Small constant controlling the step size of the gradient descent update. Value not specified in this section.
- **Pitfall**: Median depth from a pre-trained NeRF becomes inaccurate when the angle between the view direction and the surface is small (grazing angles) [S2].
- **Usage**: Applied in NeRFMeshing as a preprocessing step to initialize the mesh geometry before training the SSAN model [S2].

### Robust Depth Loss
- **Definition**: A loss function designed to address geometry error degradation in extrapolation performance by being less sensitive to outliers than L2 loss, encouraging NeRF to terminate rays around pseudo depth while retaining freedom for inaccurate geometry [S3].
- **Formula**: $\mathcal{L}_{\text{robust}}(\mathbf{r}) = \begin{cases} 0.5\Delta D(\mathbf{r})^{2}, &\text{if } \Delta D(\mathbf{r}) < \beta \\ \beta^{2} (0.5 + \log (\frac{\Delta D(\mathbf{r})}{\beta})), &\text{otherwise } \end{cases}$ where $\Delta D(\mathbf{r}) = \hat{D}(\mathbf{r}) - D(\mathbf{r})$ is the absolute depth difference, $\beta$ is a constant set to 0.1, and $\hat{D}(\mathbf{r})$ is the depth expectation along the camera axis of ray $r(t)$ [S3].
- **Parameter**: $\beta$: A constant threshold for the robust depth loss, set to 0.1 throughout the experiments [S3].

### Depth Supervision with Planar Architectural Depth Priors
- **Definition**: Incorporates precomputed depth estimates for floor, ceiling, and walls into NeRF training to enforce geometric constraints on textureless planar surfaces [S4].
- **Step**: 1. Compute depth estimates for floor, ceiling, and walls using semantic segmentation. 2. Incorporate these priors into training with MSE and BoundL losses alongside photometric loss. 3. Train until convergence [S4].
- **Parameter**: $\delta$ (BoundL tolerance): 1 mm. $\lambda_{color}$: 10 (bedroom), 1 (living room). $\lambda_{depth}$: 10 (bedroom), 1 (living room) [S4].
- **Number**: RMSE of priors vs ground truth: 2.786 mm (bedroom), 3.201 mm (living room). Converged in 120,000 iterations [S4].
- **Comparison**: Faster convergence than photometric-only (120k vs 200k) and patch-regularization (120k vs 280k) methods due to depth supervision enabling quick identification of empty spaces, concentrating sampling on occupied regions, and providing a stronger optimization signal [S4].

### Indirect Depth Supervision (KL Loss)
- **Definition**: A regularization technique that uses depth priors to constrain the NeRF weights by aligning the probability distribution of rendering weights with the depth prior, specifically using KL divergence [S5].
- **Formula**: $L_{KL} = \sum_{i}^{N_r} \sum_{t=t_n}^{t_f} D(r_i) - \epsilon \log w_k \exp(-\frac{(t - D(r_i))^2}{2\hat{\sigma}^2}) \Delta t_k$, where $w_k$ are rendering weights, $t_k$ are sampled points, $\Delta t_k$ are distances, $D(r_i)$ is ground truth depth, $\epsilon$ is a kernel parameter, and $\hat{\sigma}$ is a variance parameter [S5].
- **Parameter**: $\epsilon(x)$: A kernel that integrates to one (a distribution) with a bounded domain parameterized by $\epsilon$ [S5].
- **Pitfall**: The URF loss (Urban-NeRF) has no open-source implementation, so KL loss is selected as the representative for indirect supervision [S5].

### Geometry and Opacity Regularization
- **Formula**: Sparsity loss: $L_S = \frac{1}{N} \sum_{i=1}^{N} O(i)$, where $N$ is number of rays intersecting the 3D box, encouraging sparse opacity to suppress floaters. Opacity loss: $L_O = -\frac{1}{N} \sum_{i=1}^{N} [O(i) \cdot \log(O(i)) + (1 - O(i)) \cdot \log(1 - O(i))]$, encouraging opacity values to be 0 or 1 to avoid translucent effects [S6].

### Depth Filtering with RGB Confidence
- **Definition**: Filters predicted depth maps by computing a per-pixel confidence metric based on the reconstruction error between predicted and ground truth RGB images [S7, unofficial].
- **Step**: Load predicted RGBs and depths from NeRF results directory. Load ground truth RGBs from the data directory. Compute confidence map by taking 1 minus the mean absolute error between normalized predicted and ground truth RGBs across channels. Pass ground truth RGB, predicted depth, confidence map, and save path to a parallel processing pool for filtering [S7, unofficial].
- **Parameter**: worker_num: Number of parallel workers for the multiprocessing pool, default 8 [S7, unofficial].

### Direct Depth Supervision
- **Definition**: A training objective that directly compares the depth rendered by the NeRF with a ground truth depth prior using a pixel-wise loss function [S5].
- **Step**: First sample a batch of $N_r$ rays, then compare the rendered depth and ground truth depth with different loss functions (MSE or L1) [S5].

### GeoAug
- **Definition**: An approach that uses data augmentation by rendering new training images with new noisy camera poses using DSNeRF as a baseline and using depth as regularization [S8].

## Background
The integration of geometric supervision into NeRF training emerged as a solution to the "shape-radiance ambiguity," where multiple 3D configurations can produce identical 2D images, particularly in sparse-view or textureless scenarios [S1, S5]. Early work, such as DS-NeRF, demonstrated that sparse Structure-from-Motion (SfM) keypoints could serve as effective, "free" supervision to guide the density field [S1]. Subsequent research expanded this to include monocular depth priors [8679-single-view-nerf-training-with-depth-map.md], stereo depth maps [10185-statistical-depth-refinement.md], and architectural priors [S4]. The field has evolved from simple L1/L2 depth losses to more sophisticated mechanisms like KL-divergence alignment [S5], robust losses handling outliers [S3], and consistency checks between coarse/fine networks [S1]. These methods are widely used in applications requiring high-fidelity geometry, such as 3D mesh extraction [S2], indoor reconstruction [S4], and free-view synthesis from sparse inputs [S1].

## Sources
[S1] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
[S2] NeRFMeshing: Distilling Neural Radiance Fields into Geometrically-Accurate 3D Meshes, https://doi.org/10.1109/3dv62453.2024.00093, fetched 2026-10-29
[S3] NeRFVS: Neural Radiance Fields for Free View Synthesis via Geometry Scaffolds, https://doi.org/10.1109/cvpr52729.2023.01588, fetched 2026-10-26
[S4] Improving Geometric Consistency for 360-Degree Neural Radiance Fields in Indoor Scenarios, https://arxiv.org/abs/2503.13710v1, fetched 2026-10-27
[S5] Digging into Depth Priors for Outdoor Neural Radiance Fields, https://doi.org/10.1145/3581783.3612306, fetched 2026-10-30
[S6] GO-NeRF: Generating Objects in Neural Radiance Fields for Virtual Reality Content Creation, https://doi.org/10.1109/tvcg.2025.3549558, fetched 2026-10-29
[S7] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
[S8] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
