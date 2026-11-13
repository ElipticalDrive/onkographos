# Depth and Geometry Supervision

Also called: Geometric Regularization, Depth-Aware Training

## Overview
Depth and Geometry Supervision refers to a family of training procedures that incorporate explicit geometric constraints—such as depth maps, 3D points, or structural priors—into the optimization of Neural Radiance Fields (NeRFs) to resolve the shape-radiance ambiguity inherent in sparse-view or textureless scenarios [S1, 8687-depth-supervised-nerf.md]. Unlike sibling methods that rely on external generative priors (e.g., [Diffusion-regularized NeRF](7381-diffusion-regularized-nerf.md)) or semantic consistency ([Semantic Consistency Loss (CLIP-based)](11955-semantic-consistency-loss-clip-based.md)), this family explicitly aligns the rendered density distribution or ray termination with known geometric signals, ranging from sparse SfM keypoints to dense monocular depth estimates [S1, 8687-depth-supervised-nerf.md, 9371-depth-structural-consistency-regularization.md].

## Shared principles
All methods in this family operate within an augmented loss structure where the total loss $\mathcal{L}_{\text{total}}$ is a weighted sum of the standard photometric reconstruction loss and one or more regularization terms $\mathcal{L}_k$ [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]:

$$ \mathcal{L}_{\text{total}} = \mathcal{L}_{\text{photo}} + \sum_k \lambda_k \mathcal{L}_k $$

where $\lambda_k$ are hyperparameters controlling the strength of each regularizer [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]. The photometric loss is typically the Mean Squared Error (MSE) between the rendered color $\mathbf{C}$ and the ground truth image $I$:

$$ \mathcal{L}_{\text{photo}} = || \mathbf{C} - I ||_2^2 $$

Regularization terms $\mathcal{L}_k$ operate on the density $\sigma$ and color $\mathbf{c}$ fields, the rendered weights $w_i$, or the final rendered images to constrain the solution space [S1, 7475-geometric-baseline-regularization.md, 8582-regnerf.md]. Common strategies include enforcing spatial smoothness, ensuring geometric consistency across views, aligning the output distribution with learned priors, or matching semantic embeddings. The gradients of these terms are backpropagated through the differentiable volume rendering process to update the NeRF parameters [S1, 8582-regnerf.md].

## Subtopics

### [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md)
A composite loss function designed to prevent degenerate solutions in NeRF optimization, particularly in few-view settings where models may collapse to copying training images. It combines photometric reconstruction with three geometric priors: foreground completeness, frustum consistency, and depth distribution compactness. It serves as a strong baseline for differentiable volume rendering optimization, distinct from sibling methods that rely on external depth supervision or point clouds [8687-depth-supervised-nerf.md, 12693-3d-point-supervision-loss.md].

### [Guided Optimization of NeRF](7496-guided-optimization-of-nerf.md)
Mitigates shape-radiance ambiguity in poorly textured indoor scenes by constraining the sampling range of NeRFs using adapted monocular depth priors. Unlike standard NeRF, which samples uniformly between near and far bounds derived from sparse 3D reconstruction, this method calculates adaptive sampling intervals based on multi-view geometric consistency errors, ensuring sampled points concentrate around reliable depth estimates. It sets itself apart from sibling methods like [Depth-Supervised NeRF](8687-depth-supervised-nerf.md) by using depth priors primarily to guide the *sampling strategy* during optimization rather than solely as a direct regression target, thereby avoiding the degradation caused by unreliable flow correspondences in texture-less regions.

### [Single-view NeRF Training with Depth Map](8679-single-view-nerf-training-with-depth-map.md)
Addresses the under-constrained optimization problem of single-view NeRF reconstruction by initializing the geometry of visible regions using a 360-degree depth estimation. It sets itself apart from sibling methods like [Depth-Supervised NeRF](8687-depth-supervised-nerf.md) by specifically handling the scale ambiguity and field-of-view limitations inherent in projecting a single panorama into perspective views for depth estimation, using a deformable alignment step to create a canonical depth map before NeRF training.

### [Depth-Supervised NeRF](8687-depth-supervised-nerf.md)
DS-NeRF accelerates NeRF optimization and improves quality under sparse-view conditions by incorporating depth supervision from sparse 3D keypoints generated during Structure-from-Motion (SfM) preprocessing. Unlike methods that rely on data-driven priors from other scenes or semantic consistency, DS-NeRF uses the "free" depth estimates inherent to standard SfM pipelines (e.g., COLMAP) to enforce that the rendered ray termination distribution matches surface priors, thereby reducing overfitting to empty space and speeding up convergence. It is distinct from sibling methods like [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md) by explicitly modeling depth uncertainty as a Gaussian distribution and minimizing the Kullback-Leibler (KL) divergence between the rendered distribution and this noisy depth prior.

### [Depth Structural Consistency Regularization](9371-depth-structural-consistency-regularization.md)
RDSC is a geometric regularization term applied during NeRF training to enforce continuity constraints on rendered depth maps using external depth priors, such as those from DepthAnythingV2. It distinguishes itself from sibling methods like [Geometric Baseline Regularization](7475-geometric-baseline-regularization.md) by explicitly modeling local depth variations between neighboring pixels through a k-nearest neighbor (KNN) search, rather than relying solely on global smoothness or pixel-wise alignment. This approach ensures that the structural consistency of the input depth map is preserved in the NeRF output, even in the presence of minor depth variations.

### [Statistical Depth Refinement](10185-statistical-depth-refinement.md)
A post-processing step applied during the training of radiance fields to mitigate artifacts caused by corrupt stereo depth inputs, such as those arising from fuzzy pixels or specular highlights in surgical scenes. It prevents the model from overfitting to these noisy depth values by replacing them with smoother depth estimates derived from an underfitting model state. Unlike direct depth supervision methods that treat all depth pixels equally, this method statistically identifies outliers based on residual distributions and patches them, thereby preserving geometric consistency while reducing abrupt reconstruction artifacts.

### [Novel View Regularization](10806-novel-view-regularization.md)
Mitigates the shape-radiance ambiguity in NeRFs when training data is sparse by injecting synthetic rays into the optimization loop to provide additional geometric supervision. Unlike methods that rely solely on observed input views, this approach explicitly guides the density field toward geometrically plausible solutions, preventing overfitting to the limited training set and enhancing geometry accuracy in unseen regions.

### [3D Point Supervision Loss](12693-3d-point-supervision-loss.md)
A geometric regularization term that minimizes the distance between 3D points backprojected from rendered depth and ground-truth 3D points. It solves the problem of pose registration and geometry optimization in sparse-view regimes where standard depth losses fail to refine camera poses. Unlike sibling methods that compare rendered depth directly to ground-truth depth maps, this method enforces a global geometric connection by operating in the world frame, preventing disjoint local solutions.

### [Generalized Ray Supervision](14127-generalized-ray-supervision.md)
A framework for optimizing neural scene representations by treating arbitrary rays in a scene as "ray sensors" that record radiance, extending the standard NeRF approach which is limited to camera rays. It differs from sibling methods like [Depth and Geometry Supervision](8727-depth-and-geometry-supervision.md) by not requiring explicit geometric labels (e.g., depth maps or point clouds); instead, it leverages the physical light transport properties of specific ray types (such as shadow rays) to provide direct radiance supervision for the neural field.

### Coarse-Fine Consistency Loss
- **Definition:** A loss function that enforces consistency between depth estimates of the coarse and fine NeRF models to prevent ineffective hierarchical sampling and blur [S1].
- **Formula:** $\mathcal{L}_{cfc} = \mathbb{1}_{\{m_f=1\}} \odot ||z_c - \hat{\nabla}(z_f)||^2 + \mathbb{1}_{\{m_c=1\}} \odot ||\hat{\nabla}(z_c) - z_f||^2$, where $z_c$ and $z_f$ are coarse and fine depths, and $m_c/m_f$ are reliability masks determined by reprojection [S1].

### Zero-level set projection
- **Definition:** Refines inaccurate median depth from a pre-trained NeRF, particularly at grazing view angles, by iteratively projecting points onto the zero level-set of the surface using TSDF values [S2].
- **Step:** For each 3D point $p$ on the viewing ray, perform exactly 4 iterative gradient descent steps along the ray direction using the TSDF predicted value to update the point position [S2].
- **Formula:** $p_{new} = p_{old} + \alpha \times \hat{t}$, where $[\hat{t}, \hat{n}, \hat{f}] = \phi(p)$; $p$ is the current 3D point, $\alpha$ is a small constant step size, $[\hat{t}, \hat{n}, \hat{f}]$ are the tangent, normal, and feature outputs from network $\phi$ evaluated at $p$ [S2].
- **Parameter:** $\alpha$: Small constant controlling the step size of the gradient descent update. Value not specified in this section [S2].
- **Pitfall:** Median depth from a pre-trained NeRF becomes inaccurate when the angle between the view direction and the surface is small (grazing angles) [S2].
- **Usage:** Applied in NeRFMeshing as a preprocessing step to initialize the mesh geometry before training the SSAN model [S2].

### Robust Depth Loss
- **Definition:** A loss function designed to address geometry error degradation in extrapolation performance by being less sensitive to outliers than L2 loss, encouraging NeRF to terminate rays around pseudo depth while retaining freedom for inaccurate geometry [S3].
- **Formula:** $\mathcal{L}_{\text{robust}}(\mathbf{r}) = \begin{cases} 0.5\Delta D(\mathbf{r})^{2}, &\text{if } \Delta D(\mathbf{r}) < \beta \\ \beta^{2} (0.5 + \log (\frac{\Delta D(\mathbf{r})}{\beta})), &\text{otherwise } \end{cases}$ where $\Delta D(\mathbf{r}) = \hat{D}(\mathbf{r}) - D(\mathbf{r})$ is the absolute depth difference, $\beta$ is a constant set to 0.1, and $\hat{D}(\mathbf{r})$ is the depth expectation along the camera axis of ray $r(t)$ [S3].
- **Parameter:** $\beta$: A constant threshold for the robust depth loss, set to 0.1 throughout the experiments [S3].

### Depth Supervision with Planar Architectural Depth Priors
- **Definition:** Incorporates precomputed depth estimates for floor, ceiling, and walls into NeRF training to enforce geometric constraints on textureless planar surfaces [S4].
- **Step:** 1. Compute depth estimates for floor, ceiling, and walls using semantic segmentation. 2. Incorporate these priors into training with MSE and BoundL losses alongside photometric loss. 3. Train until convergence [S4].
- **Parameter:** $\delta$ (BoundL tolerance): 1 mm. $\lambda_{color}$: 10 (bedroom), 1 (living room). $\lambda_{depth}$: 10 (bedroom), 1 (living room) [S4].
- **Number:** RMSE of priors vs ground truth: 2.786 mm (bedroom), 3.201 mm (living room). Converged in 120,000 iterations [S4].
- **Comparison:** Faster convergence than photometric-only (120k vs 200k) and patch-regularization (120k vs 280k) methods due to depth supervision enabling quick identification of empty spaces, concentrating sampling on occupied regions, and providing a stronger optimization signal [S4].

### Indirect Depth Supervision (KL Loss)
- **Definition:** A regularization technique that uses depth priors to constrain the NeRF weights by aligning the probability distribution of rendering weights with the depth prior, specifically using KL divergence [S5].
- **Formula:** $L_{KL} = \sum_{i}^{N_r} \sum_{t=t_n}^{t_f} D(r_i) - \epsilon \log w_k \exp(-\frac{(t - D(r_i))^2}{2\hat{\sigma}^2}) \Delta t_k$, where $w_k$ are rendering weights, $t_k$ are sampled points, $\Delta t_k$ are distances, $D(r_i)$ is ground truth depth, $\epsilon$ is a kernel parameter, and $\hat{\sigma}$ is a variance parameter [S5].
- **Parameter:** $\epsilon(x)$: A kernel that integrates to one (a distribution) with a bounded domain parameterized by $\epsilon$ [S5].
- **Pitfall:** The URF loss (Urban-NeRF) has no open-source implementation, so KL loss is selected as the representative for indirect supervision [S5].

### Geometry and Opacity Regularization
- **Formula:** Sparsity loss: $L_S = \frac{1}{N} \sum_{i=1}^{N} O(i)$, where $N$ is number of rays intersecting the 3D box, encouraging sparse opacity to suppress floaters. Opacity loss: $L_O = -\frac{1}{N} \sum_{i=1}^{N} [O(i) \cdot \log(O(i)) + (1 - O(i)) \cdot \log(1 - O(i))]$, encouraging opacity values to be 0 or 1 to avoid translucent effects [S6].

### Depth Filtering with RGB Confidence
- **Definition:** Filters predicted depth maps by computing a per-pixel confidence metric based on the reconstruction error between predicted and ground truth RGB images [S7, unofficial].
- **Step:** Load predicted RGBs and depths from NeRF results directory. Load ground truth RGBs from the data directory. Compute confidence map by taking 1 minus the mean absolute error between normalized predicted and ground truth RGBs across channels. Pass ground truth RGB, predicted depth, confidence map, and save path to a parallel processing pool for filtering [S7, unofficial].
- **Parameter:** worker_num: Number of parallel workers for the multiprocessing pool, default 8 [S7, unofficial].

### Direct Depth Supervision
- **Definition:** A training objective that directly compares the depth rendered by the NeRF with a ground truth depth prior using a pixel-wise loss function [S5].
- **Step:** First sample a batch of $N_r$ rays, then compare the rendered depth and ground truth depth with different loss functions (MSE or L1) [S5].

### GeoAug
- **Definition:** An approach that uses data augmentation by rendering new training images with new noisy camera poses using DSNeRF as a baseline and using depth as regularization [S8].

## Background
The integration of geometric supervision into NeRF optimization emerged as a response to the "shape-radiance ambiguity," where sparse or textureless inputs lead to degenerate solutions such as floating artifacts or incorrect geometry [S1, 8687-depth-supervised-nerf.md]. Early work, such as DS-NeRF, leveraged the sparse 3D keypoints naturally produced by Structure-from-Motion (SfM) pipelines to regularize the density field via KL divergence between rendered and prior depth distributions [S1, 8687-depth-supervised-nerf.md]. Subsequent methods expanded this to include dense monocular depth priors, stereo depth refinement, and structural consistency constraints to handle challenging indoor and outdoor scenes [S1, 9371-depth-structural-consistency-regularization.md, 10185-statistical-depth-refinement.md]. The family also includes techniques that use depth to guide sampling strategies [7496-guided-optimization-of-nerf.md] or to enforce consistency between hierarchical coarse and fine models [S1].

## Sources
[S1] Simple-RF: Regularizing Sparse Input Radiance Fields with Simpler Solutions, https://arxiv.org/abs/2404.19015v5, fetched 2026-10-27
[S2] NeRFMeshing: Distilling Neural Radiance Fields into Geometrically-Accurate 3D Meshes, https://doi.org/10.1109/3dv62453.2024.00093, fetched 2026-10-29
[S3] NeRFVS: Neural Radiance Fields for Free View Synthesis via Geometry Scaffolds, https://doi.org/10.1109/cvpr52729.2023.01588, fetched 2026-10-26
[S4] Improving Geometric Consistency for 360-Degree Neural Radiance Fields in Indoor Scenarios, https://arxiv.org/abs/2503.13710v1, fetched 2026-10-27
[S5] Digging into Depth Priors for Outdoor Neural Radiance Fields, https://doi.org/10.1145/3581783.3612306, fetched 2026-10-30
[S6] GO-NeRF: Generating Objects in Neural Radiance Fields for Virtual Reality Content Creation, https://doi.org/10.1109/tvcg.2025.3549558, fetched 2026-10-29
[S7] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
[S8] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
