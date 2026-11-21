# Densification Strategy

Also called: Adaptive Density Control, Densification and Pruning, Primitive Management

## Overview
Densification Strategy is the family of mechanisms used during the training of 3D Gaussian Splatting (3DGS) to dynamically adjust the number, spatial distribution, and properties of Gaussian primitives [9188-adaptive-density-control.md]. It solves the problem of under-reconstruction (missing geometry due to insufficient primitives) and over-reconstruction (coarse approximations or artifacts due to excessive or poorly placed primitives) by adding, cloning, splitting, or removing Gaussians based on optimization metrics [9188-adaptive-density-control.md]. It sets itself apart from static initialization methods by using view-space positional gradient magnitudes, opacity, or saliency scores to trigger densification and pruning operations at regular intervals, ensuring the representation remains compact while capturing fine details [9188-adaptive-density-control.md]. Unlike direct pruning methods that statically remove primitives (e.g., LightGaussian), Adaptive Density Control (ADC) balances addition and removal during optimization to avoid suboptimal simplification outcomes caused by inefficient spatial distribution [9188-adaptive-density-control.md].

## Shared principles
The core mechanism of densification strategies in 3DGS involves a periodic loop that evaluates the current state of the Gaussian cloud and modifies it to better fit the training data. The following principles are common to the standard densification loop and its direct variants [9314-pruning-and-opacity-management.md]:

1.  **Trigger Condition:** Densification is not performed at every optimization step. It is triggered periodically (e.g., every `refineEvery` steps) after a warm-up period (`warmupLength`) and before a stopping threshold (`stopSplitAt`, typically half the total steps) to allow the optimizer to stabilize [Parent][9188-adaptive-density-control.md]. If the current step modulo `resetInterval` is within `numCameras + refineEvery` of the start of the interval, densification is skipped [Parent][9188-adaptive-density-control.md].
2.  **Metric Computation:** A score or metric is computed for each Gaussian $\mathcal{G}_i$ to determine its contribution to the reconstruction error. Common metrics include:
    *   **Gradient Norm:** The magnitude of the gradient of the loss with respect to the Gaussian's 2D mean position ($\nabla_{\mu_{2D}}$) [Parent][9188-adaptive-density-control.md]. The 2D positional gradient magnitude $\tau_k$ is averaged over $N_{\text{views}}$ views between densification steps:
        $$ \tau_k = \frac{1}{N_{\text{views}}} \sum_{v=1}^{N_{\text{views}}} \left\| \frac{\partial L(I_v, I'_v)}{\partial \mu_v^v} \right\|_2 $$
        where $L$ is the image loss between ground truth $I_v$ and rendered $I'_v$ [9188-adaptive-density-control.md]. Note: In implementations using inversion-free splat evaluation, screen-space gradients are unavailable; instead, the gradient of splat means scaled by half the distance to the camera in view space is used [Parent][7].
    *   **Screen Size:** The projected area of the Gaussian in pixel space. Gaussians that are too small may not capture detail, while those that are too large may oversmooth [Parent].
    *   **Opacity:** The current opacity $\alpha_i$ of the Gaussian. Low opacity often indicates redundancy or "floaters" [Parent].
    *   **Saliency/Loss Contribution:** Aggregated loss or gradient magnitude over the pixels covered by the Gaussian [4705-score-based-densification.md].
3.  **Action (Split/Clone/Prune):**
    *   **Split:** A Gaussian is replaced by two new Gaussians. The new means are derived from the original mean plus/minus a perturbation along the principal axes of the covariance matrix. The new scales are typically reduced by a factor (e.g., 1.6) [Parent][4705-score-based-densification.md].
    *   **Clone:** A copy of the Gaussian is created with a small random perturbation to its mean to fill gaps without reducing scale [Parent][4705-score-based-densification.md].
    *   **Prune:** Gaussians failing specific criteria (e.g., low opacity, low significance) are removed from the active set [Parent].
4.  **Opacity Management:** To prevent over-densification and stabilize training, opacities of newly created Gaussians are initialized to a low value (e.g., logit -1.386) [Parent][4705-score-based-densification.md]. Some methods use opacity splitting formulas to maintain transmittance consistency during cloning [9188-adaptive-density-control.md].
5.  **Data Structure Management:** The set of Gaussians is stored in GPU buffers. Densification requires managing buffer capacity (often pre-allocated to be at least 3x the initial count) and performing compaction to remove pruned primitives [Parent][9188-adaptive-density-control.md].

## Subtopics

### [Score-based Densification](4705-score-based-densification.md)
[Score-based Densification](4705-score-based-densification.md) is a constructive densification strategy for 3D Gaussian Splatting that steers the addition of new primitives toward those that maximize reconstruction quality, ensuring the model size increases in a controlled manner toward an exact user-defined budget. Unlike standard adaptive density control which relies on gradient thresholds and subsequent pruning, this method uses a purely constructive schedule, avoiding the optimization instability and hardware constraint violations caused by substantial pruning or culling during training.

### [GroupFlow](6351-groupflow.md)
[GroupFlow](6351-groupflow.md) is a motion compression technique for dynamic 3D Gaussian Splatting that distills per-Gaussian neural deformation fields into shared SE(3) transformations for groupwise motion. It solves the computational bottleneck of evaluating individual deformation networks for every Gaussian at every timestep by clustering Gaussians into groups and assigning each group a single learned rigid-body trajectory. Unlike methods that retain per-Gaussian neural fields or use global rigid transforms, GroupFlow captures locally coherent motion with $J \ll N$ parameters, accelerating rendering by up to 13.71× while maintaining temporal coherence.

### [3DGS-MCMC](7852-3dgs-mcmc.md)
[3DGS-MCMC](7852-3dgs-mcmc.md) replaces the noisy positional gradient magnitude heuristics of standard 3D Gaussian Splatting with a probabilistic framework that treats the Gaussian cloud as samples from an underlying probability distribution. It uses Markov Chain Monte Carlo (MCMC) sampling to guide the life cycle (spawn/prune) of primitives, allowing the method to explore regions with unknown gradients and yielding more efficient, artifact-free clouds.

### [VS-Splat Voxel Selection](9132-vs-splat-voxel-selection.md)
[VS-Splat Voxel Selection](9132-vs-splat-voxel-selection.md) is a learnable, differentiable mechanism for filtering 3D Gaussian primitives based on their relevance to the target object, replacing static opacity thresholding. It solves the problem of background noise and redundant primitives in sparse-view reconstruction by predicting confidence scores via Gumbel-Sigmoid and using them to refine opacity values. It sets itself apart from siblings like [Score-based Densification](4705-score-based-densification.md) by operating on a voxel grid with learned confidence rather than gradient norms, and from [Adaptive Density Control](9188-adaptive-density-control.md) by using a stochastic binary approximation for selection rather than deterministic splitting rules.

### Adaptive Density Control (Split & Clone)
[Adaptive Density Control](9188-adaptive-density-control.md) is a dynamic optimization strategy used during the training of 3D Gaussian Splatting (3DGS) to adjust the number and spatial distribution of Gaussian primitives [S1, unofficial][S8, unofficial]. It solves the problem of under-reconstruction (missing geometry) and over-reconstruction (coarse approximations or artifacts) by dynamically adding, cloning, splitting, or removing Gaussians based on gradient magnitude, scale, and opacity [S1, unofficial][S8, unofficial]. It sets itself apart from static initialization methods by using view-space positional gradient magnitudes to trigger densification and pruning operations at regular intervals, ensuring the representation remains compact while capturing fine details [S1, unofficial][S2]. Unlike direct pruning methods that statically remove primitives (e.g., LightGaussian), ADC balances addition and removal during optimization to avoid suboptimal simplification outcomes caused by inefficient spatial distribution [S1, unofficial].

### [Pruning and Opacity Management](9314-pruning-and-opacity-management.md)
[Pruning and Opacity Management](9314-pruning-and-opacity-management.md) controls the density and redundancy of Gaussian primitives during training to prevent overfitting, reduce memory usage, and stabilize optimization. It solves the problem of unbounded Gaussian growth and the persistence of "floaters" (unstable or redundant primitives) by selectively removing low-contribution Gaussians and managing the opacity of newly created ones. It differs from its siblings by focusing specifically on the removal criteria (opacity, significance, temporal sensitivity) and the initialization/reset policies of opacities, whereas sibling articles like [Adaptive Density Control](9188-adaptive-density-control.md) focus on the densification (splitting/cloning) triggers and [Score-based Densification](4705-score-based-densification.md) focuses on the gradient-based metrics for adding primitives.

### [Locality-preserving Densification](9585-locality-preserving-densification.md)
[Locality-preserving Densification](9585-locality-preserving-densification.md) addresses the memory layout bottleneck in 3D Gaussian Splatting training, where the addition of new Gaussians to the end of parameter buffers causes spatially close primitives to be stored far apart in memory. This spatial fragmentation leads to uncoalesced memory accesses and increased warp divergence during rasterization. By reordering Gaussians based on their 3D spatial proximity using Z-ordering (Morton order), the method ensures that neighboring Gaussians in space are also adjacent in memory, thereby improving cache efficiency and reducing training time for scenes with high Gaussian counts. It is distinct from score-based or group-based densification strategies by focusing exclusively on the memory layout optimization rather than the geometric distribution or selection criteria of the primitives.

### [Opacity Split for Cloning](9694-opacity-split-for-cloning.md)
[Opacity Split for Cloning](9694-opacity-split-for-cloning.md) is a method to initialize the opacity of cloned Gaussians to maintain transmittance consistency, preventing the cloned child from having less impact than the parent alone. It solves the problem where standard cloning inherits the parent's full opacity, which artificially increases the total transmittance attenuation of the parent-child pair compared to the original single Gaussian, thereby biasing the alpha blending process. It sets itself apart from generic cloning by explicitly deriving new opacity values from the parent's opacity to preserve the cumulative transmittance $t_k$ before and after the split.

### [Progressive and Multi-stage Training](9966-progressive-and-multi-stage-training.md)
[Progressive and Multi-stage Training](9966-progressive-and-multi-stage-training.md) addresses the problem of converging to local minima with insufficient geometric detail by initializing a sparse representation and iteratively refining it through stages of increasing density or complexity. Unlike single-stage optimization which optimizes a static point cloud from the start, these methods dynamically grow the primitive set based on reconstruction error, yielding significantly higher rendering quality. They set themselves apart from static initialization methods by enabling dynamic growing and pruning during the optimization phase, which reduces memory consumption by removing redundant initial Gaussians while maintaining high fidelity.

### [Gaussian Insertion and Pruning](10074-gaussian-insertion-and-pruning.md)
[Gaussian Insertion and Pruning](10074-gaussian-insertion-and-pruning.md) is a densification strategy designed for monocular and RGB-D SLAM systems using 3D Gaussian Splatting, specifically addressing the initialization of new primitives in unseen regions and the removal of geometrically unstable outliers. Unlike standard offline densification which relies on gradient norms or screen-space error, this method uses visibility consistency across a sliding window of keyframes to validate new Gaussians. It is distinct from [Adaptive Density Control](9188-adaptive-density-control.md) by prioritizing SLAM stability over reconstruction fidelity, and from [Score-based Densification](4705-score-based-densification.md) by using explicit depth sampling and cross-view visibility checks rather than accumulated loss gradients.

### [TangoGS](10180-tangogs.md)
[TangoGS](10180-tangogs.md) is a density controller for 3D Gaussian Splatting that automatically scales the number of Gaussians across different capture sizes without per-capture retuning. It combines a capture-derived learning allowance (model sizing) with training-based adaptation (quality feedback) to determine the target Gaussian population. Unlike sibling methods that rely on fixed defaults or per-capture tuning, TangoGS ensures the model budget is invariant to redundant views by using track-adjusted pixels normalized against standard scenes.

### [Hyperparameter Settings](11076-hyperparameter-settings.md)
[Hyperparameter Settings](11076-hyperparameter-settings.md) define the numerical configuration for the optimizer, learning rates, and regularization terms used during the training of 3D Gaussian Splatting variants. These settings are critical for stabilizing the optimization process, particularly in physically-based inverse rendering where material-lighting ambiguities and geometric errors can lead to gradient overflow. This article consolidates specific hyperparameter values from and, focusing on the Adam optimizer configuration, per-parameter learning rates, and regularization strengths required to prevent instability caused by erroneous Gaussians.

### [Adaptive Gaussian Refinement](11448-adaptive-gaussian-refinement.md)
[Adaptive Gaussian Refinement](11448-adaptive-gaussian-refinement.md) is a technique that dynamically adjusts Gaussian parameters using neural network-predicted residuals to improve the modeling of complex non-rigid deformations and time-varying appearance. It addresses the limitation of static covariance matrices in representing dynamic behaviors by applying residual corrections computed by lightweight neural networks. Unlike standard densification strategies that modify the set of primitives via splitting or cloning, this method refines the existing parameters of each Gaussian to better fit temporal and angular variations.

### [Hybrid and Diffusion Representations](11473-hybrid-and-diffusion-representations.md)
[Hybrid and Diffusion Representations](11473-hybrid-and-diffusion-representations.md) extend 3D Gaussian Splatting (3DGS) from a discriminative reconstruction technique to a generative framework, enabling scene synthesis from single images or text prompts without iterative per-scene optimization. Unlike standard differentiable training loops that optimize Gaussian parameters via gradient descent to fit observed views, these methods employ feed-forward networks or diffusion models to directly predict Gaussian attributes. This approach solves the latency bottleneck of inverse rendering while addressing generation-specific artifacts such as the "Janus problem" (geometric inconsistency across viewpoints) and "floaters" (spurious low-opacity primitives).

### [Pruning and Sparsification](12354-pruning-and-sparsification.md)
[Pruning and Sparsification](12354-pruning-and-sparsification.md) are post-training or in-training strategies that reduce the number of 3D Gaussians in a scene to improve rendering efficiency, reduce memory footprint, or correct reconstruction artifacts. Unlike [Densification Strategy](6234-densification-strategy.md), which increases density to capture detail, this family removes redundant or insignificant primitives. It is distinguished from sibling methods by its focus on removing Gaussians based on specific criteria such as accumulated rendering significance, second-order uncertainty (Fisher Information), or spatial depth, rather than simple opacity thresholds or gradient norms alone.

### [Texel Size Adaptation and Primitive Management](12592-texel-size-adaptation-and-primitive-management.md)
[Texel Size Adaptation and Primitive Management](12592-texel-size-adaptation-and-primitive-management.md) is a content-aware optimization routine that dynamically adjusts the resolution of per-primitive texture maps and the number of 3D Gaussian primitives during training. It solves the problem of balancing model capacity against appearance quality by allocating texels and primitives only where necessary, preventing overfitting and memory exhaustion. Unlike standard densification strategies that rely primarily on gradient norms or screen-space area, this method uses a quantile-based error threshold to drive both primitive splitting and texture upscaling/downscaling, allowing the model to automatically adjust the ratio of primitives to texels based on scene complexity.

### GaussianSPA
- [S1, unofficial] Proposes an optimizing-sparsifying loop that interleaves standard training with a sparsification operator.
- [S1, unofficial] Removes redundant primitives while boosting the influence of the remaining collection.

### Blur Splitting
- [S2] A densification step added to the 3DGS baseline that improves rendering quality and LPIPS metrics.
- [S2] Adding Blur Split to 3DGS baseline increases Gaussians from 3.35M to 3.74M, SSIM from 0.815 to 0.819, and LPIPS from 0.216 to 0.195.

### Revised Densify and Prune Algorithm
- [S3] The complete densification and pruning algorithm integrating corrected scene extent, exponential ascending gradient threshold, and significance-aware pruning.
- [S3] Iterate over all Gaussians $g_k$ in the scene $G$.
- [S3] For densification: If the modified gradient magnitude $\tau'_k$ is greater than or equal to the threshold $T_i$, proceed to check splitting/cloning conditions.
- [S3] If the maximum scale $max(s_k)$ is greater than $P_{dense} \cdot e'_{scene}$, split the Gaussian $g_k$.
- [S3] Otherwise, clone the Gaussian $g_k$.
- [S3] For pruning: If the opacity $o_k$ is less than $o_{min}$ AND the accumulated significance $\sigma_k$ is in the bottom $N_{prune}$ of $\Sigma$, prune the Gaussian $g_k$.
- [S3] $N_{prune}$: The count of lowest-significance Gaussians to prune.
- [S3] Data: Scene of Gaussians G for gk ∈ G do //Densification if τ ′ k ≥ Ti then if max(sk) > Pdense · e ′ scene then splitGaussian(gk); else cloneGaussian(gk); end end //Significance Pruning if ok < omin then if σ*k* ∈ bottom(Nprune,Σ) then pruneGaussian(gk); end end end
- [S3] Implemented in the GitHub repository https://github.com/fraunhoferhhi/Improving-ADC-3DGS.

### Trick-GS
- [S4] A Gaussian Splatting method that combines progressive training with resolution, noise and Gaussian scales, learning to prune and mask primitives and SH bands by their significance, and an accelerated GS training framework to achieve faster run-time, smaller disk size, and faster convergence compared to vanilla GS.
- [S4] 3DGS reconstructs a scene using 3D Gaussian primitives rendered via differentiable volume splatting rasterizer extending EWA volume splatting. Each primitive has 59 parameters: position, covariance (parameterized by scaling matrix S and quaternion rotation q), opacity, diffused color, and spherical harmonic (SH) coefficients (typically 3 bands).
- [S4] On three datasets, Trick-GS achieves up to 2× faster training, 40× smaller disk size and 2× faster rendering speed compared to vanilla GS, while having comparable accuracy.

### Parallel Respawning and Densification with Edge Case Handling
- [S5, unofficial] A densification strategy where respawn and densify operations are performed in parallel, requiring specific edge case handling to prevent hierarchy degeneration.
- [S5, unofficial] If two sibling nodes both need to be respawned, only respawn the right node. This causes the left node to become a new dead leaf, which will be respawned in the next densification iteration.
- [S5, unofficial] If a node needs to be respawned whose sibling is not a leaf node, replace the entire parent node with the subtree of the node being respawned.
- [S5, unofficial] Apply a pruning strategy: zero the opacity of Gaussians that have not contributed for a number of iterations equal to twice the size of the training set. These will be respawned in the next densification iteration.

### Opacity Reset & Densification Trigger
- [S6, unofficial] If the current iteration is a multiple of opt.opacity_reset_interval, or if the dataset uses a white background and the current iteration equals opt.densify_from_iter, call gaussians.reset_opacity().
- [S6, unofficial] opt.opacity_reset_interval: Iteration interval at which Gaussian opacities are reset during training.
- [S6, unofficial] opt.densify_from_iter: Iteration threshold that triggers densification specifically for white-background datasets.

### Densify and Prune Optimization
- [S7, unofficial] Adjusting and removing the `densify_and_split` operation within the `densify_and_prune` process to improve speed without impacting rendering quality.

### Adaptive Density Control (Split & Clone)
- [S8, unofficial] A mechanism to adjust the density of 3D Gaussians during optimization to match local scene complexity by duplicating or subdividing Gaussians based on their coverage relative to the target region.
- [S8, unofficial] During optimization, evaluate each Gaussian's coverage area against the target reconstruction region. If the Gaussian is too small (Under-Reconstruction), duplicate it (Clone). If the Gaussian is too large (Over-Reconstruction), split it into smaller Gaussians.
- [S8, unofficial] On the Average-30K dataset, Full achieves PSNR 26.05. No-Split drops to 23.90. No-Clone drops to 25.91.
- [S8, unofficial] Split removal primarily degrades background quality, causing large blurry regions. Clone removal primarily breaks thin structures (e.g., bicycle spokes), causing them to appear fragmented.

### Termination entropy criterion
- [S9, unofficial] Uses termination entropy as the criterion for calculating information gain in active 3D reconstruction.
- [S9, unofficial] Performs consistently better than baselines across all scenes, allowing the agent to explore more diverse viewpoints and build better 3D models with a limited number of viewpoints.

### Outlier Removal
- [S10] A process to address noise and missing surface parts in the density space by using perceptual loss to identify problematic geometric aspects and removing outliers.
- [S10] Compute perceptual loss VGG(S_p, S_g) to measure high-level visual similarity between rendered patch S_p and real image patch S_g.
- [S10] Select patches where VGG(S_p, S_g) is lower than half the average value across the training set to prioritize patches with improved visual similarity.
- [S10] Augment the density space by adding a group of points along the pixel ray for selected patches, considering the depth range of neighboring pixels.
- [S10] Remove outliers by examining the number of points within a specified radius around each neural density point; if the count falls below a predefined threshold, the point is removed.

## Background
The concept of adaptive density control was introduced with the original 3D Gaussian Splatting paper by Kerbl et al. (2023) [S1, unofficial]. The method was designed to address the limitations of static point cloud representations by allowing the number of primitives to adapt to the complexity of the scene during optimization. Since then, numerous variants have been proposed to improve efficiency, stability, and quality, including score-based methods, MCMC-based approaches, and specialized strategies for dynamic scenes, SLAM, and compression.

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
[S3] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S4] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
[S5] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
[S6] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
[S7] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
[S8] GaussianSplatting論文のAblationについての解説・考察, https://qiita.com/kotauchisunsun/items/f1e89f28aa770b8d3b19, fetched 2026-10-29 (unofficial)
[S9] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
[S10] DGNR: Density-Guided Neural Point Rendering of Large Driving Scenes, https://doi.org/10.1109/tase.2024.3410891, fetched 2026-10-27
