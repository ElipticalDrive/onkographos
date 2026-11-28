# Densification Strategy

Also called: Adaptive Density Control, Density Control, Densification and Pruning

## Overview
Densification Strategy is the mechanism in 3D Gaussian Splatting (3DGS) that dynamically adjusts the number and spatial distribution of Gaussian primitives during training to balance reconstruction fidelity with computational efficiency [S1, unofficial][S8, unofficial]. It solves the problem of under-reconstruction (missing geometry due to insufficient primitives) and over-reconstruction (coarse approximations or floating artifacts due to excessive primitives) by adding, cloning, or removing Gaussians based on gradient magnitude, scale, and opacity [S1, unofficial][S2]. It sets itself apart from static initialization methods by using view-space positional gradient magnitudes to trigger cloning, splitting, and pruning operations at regular intervals, ensuring the representation remains compact while capturing fine details [S1, unofficial][S2]. Unlike direct pruning methods that statically remove primitives (e.g., LightGaussian), Adaptive Density Control (ADC) dynamically balances addition and removal to avoid suboptimal spatial distribution [S1, unofficial].

## Shared principles
All densification strategies in this family operate within the differentiable training loop of 3D Gaussian Splatting. They share the following core mechanisms:

1.  **Trigger Condition:** Densification is not performed at every optimization step. It is triggered periodically (e.g., every `refineEvery` steps) after a warm-up period and before a stopping threshold (e.g., `stopSplitAt`) to allow the optimizer to stabilize [Parent][S5, unofficial].
2.  **Metric Computation:** A score is computed for each Gaussian $\mathcal{G}_i$ to determine its contribution to the reconstruction error. Common metrics include:
    *   **Gradient Norm:** The magnitude of the gradient of the loss with respect to the Gaussian's 2D mean position ($\nabla_{\mu_{2D}}$) [Parent][S5, unofficial].
    *   **Screen Size:** The projected area of the Gaussian in pixel space. Gaussians that are too small may not capture detail, while those that are too large may oversmooth [Parent].
    *   **Opacity:** The current opacity $\alpha_i$ of the Gaussian. Low opacity often indicates redundancy or "floaters" [Parent].
    *   **Saliency/Loss Contribution:** Aggregated loss or gradient magnitude over the pixels covered by the Gaussian [S1, unofficial].
3.  **Action (Split/Clone/Prune):**
    *   **Split:** A Gaussian is replaced by two new Gaussians. The new means are derived from the original mean plus/minus a perturbation along the principal axes of the covariance matrix. The new scales are typically reduced by a factor (e.g., 1.6) [Parent][S1, unofficial].
    *   **Clone:** A copy of the Gaussian is created with a small random perturbation to its mean to fill gaps without reducing scale [Parent][S1, unofficial].
    *   **Prune:** Gaussians failing specific criteria (e.g., low opacity, low significance) are removed from the active set [Parent].
4.  **Opacity Management:** To prevent over-densification and stabilize training, opacities of newly created Gaussians are initialized to a low value (e.g., logit -1.386) [Parent][S1, unofficial]. Some methods use opacity splitting formulas to maintain transmittance consistency during cloning [S5, unofficial].
5.  **Data Structure Management:** The set of Gaussians is stored in GPU buffers. Densification requires managing buffer capacity (often pre-allocated to be at least 3x the initial count) and performing compaction to remove pruned primitives [Parent][S4].

## Subtopics

### [Score-based Densification](4705-score-based-densification.md)
Score-based Densification is a method for controlling the final model size of 3D Gaussian Splatting (3DGS) by steering densification toward Gaussians that maximize reconstruction quality, ensuring the primitive count increases in a controlled manner toward an exact user-defined budget. Unlike standard 3DGS which relies on fixed thresholds and subsequent pruning, this approach uses a purely constructive schedule, avoiding the optimization instability and hardware spikes associated with aggressive culling. It builds on [Densification Strategy](6234-densification-strategy.md) by replacing heuristic splitting with a scored, budget-aware growth curve.

### [GroupFlow](6351-groupflow.md)
GroupFlow is a motion compression technique for dynamic 3D Gaussian Splatting that replaces per-Gaussian neural deformation fields with shared SE(3) transformations for groups of Gaussians. It solves the computational bottleneck of evaluating individual deformation networks for every Gaussian at every timestep, accelerating rendering by up to 13.71× and reducing training time by 2.53× while maintaining high image quality. Unlike methods that retain per-Gaussian parameters, GroupFlow distills dense motion into a smaller set of groupwise rigid-body trajectories, regularizing noisy motion estimates.

### [3DGS-MCMC](7852-3dgs-mcmc.md)
3DGS-MCMC replaces the noisy positional gradient magnitude heuristics of standard 3D Gaussian Splatting with a probabilistic framework that interprets the Gaussian cloud as samples from an underlying probability distribution. It uses Markov Chain Monte Carlo (MCMC) sampling to guide the life cycle (spawn/prune) of primitives, specifically addressing the instability and artifacts caused by gradient-based densification. Unlike sibling methods that rely on fixed thresholds or local error maps, 3DGS-MCMC explores regions with unknown gradients through Metropolis–Hastings acceptance criteria, yielding more efficient and artifact-free point clouds.

### [VS-Splat Voxel Selection](9132-vs-splat-voxel-selection.md)
VS-Splat Voxel Selection is a learnable voxel selection mechanism used within the VS-Splat framework to identify and retain only object-centric voxels for 3D Gaussian Splatting, thereby reducing redundancy and improving rendering quality. It distinguishes itself from sibling methods like Score-based Densification by using a Gumbel-Sigmoid distribution to predict confidence scores that refine opacity values, rather than relying on fixed thresholds or hard gradient-based selection. This approach outperforms opacity-based thresholding (OBT) and other variants while using significantly fewer voxels.

### [Adaptive Density Control](9188-adaptive-density-control.md)
Adaptive Density Control (ADC) is the dynamic optimization strategy used in 3D Gaussian Splatting (3DGS) to adjust the number and spatial distribution of Gaussian primitives during training. It solves the problem of under-reconstruction (missing geometry) and over-reconstruction (coarse approximations) by adding, cloning, or removing Gaussians based on gradient magnitude, scale, and opacity. It sets itself apart from static initialization methods by using view-space positional gradient magnitudes to trigger cloning, splitting, and pruning operations at regular intervals, ensuring the representation remains compact while capturing fine details. Unlike direct pruning methods that statically remove primitives (e.g., LightGaussian), ADC dynamically balances addition and removal to avoid suboptimal spatial distribution.

### [Pruning and Opacity Management](9314-pruning-and-opacity-management.md)
Pruning and Opacity Management maintains the efficiency and quality of 3D Gaussian Splatting by removing redundant or erroneous primitives ("floaters") and regulating the opacity of new Gaussians to prevent unbounded growth. It differs from sibling methods by focusing specifically on the *removal* and *initialization* phases of the density control loop, whereas siblings like [Score-based Densification](4705-score-based-densification.md) focus on the *creation* (splitting/cloning) logic.

### [Locality-preserving Densification](9585-locality-preserving-densification.md)
Locality-preserving densification addresses the memory access throttling caused by the standard densification process in 3D Gaussian Splatting, where new primitives are appended to the end of parameter buffers, causing spatially close Gaussians to be stored far apart in memory. By reordering Gaussians in parameter buffers based on their 3D spatial proximity using Z-ordering (Morton order), this method ensures that neighboring Gaussians in 3D space are also close in memory, thereby improving memory access coalescing. It sets itself apart from sibling methods like Score-based Densification by focusing on the data layout and memory efficiency rather than the criteria for splitting or cloning primitives.

### [Significance-Aware Pruning](9676-significance-aware-pruning.md)
Significance-Aware Pruning is a densification strategy for 3D Gaussian Splatting that removes redundant Gaussians by evaluating their contribution to the final image via accumulated alpha blending coefficients, rather than relying solely on opacity or spatial size. It solves the problem of standard opacity pruning removing large, structurally important Gaussians that happen to have low opacity, and standard size pruning causing an explosion in Gaussian count by replacing large primitives with many small ones. Unlike [Score-based Densification](4705-score-based-densification.md) which often uses gradient norms, this method specifically tracks the radiance contribution ($w_k$) of each Gaussian across all views to distinguish between "invisible" noise and "transparent but critical" structure.

### [Principled Uncertainty Pruning](9680-principled-uncertainty-pruning.md)
Principled Uncertainty Pruning (PUP) is a post-hoc method for compressing pretrained 3D Gaussian Splatting (3D-GS) models by removing Gaussians with high uncertainty regarding their contribution to the reconstruction error. Unlike sibling methods that rely on heuristic visibility scores or opacity thresholds, PUP computes a sensitivity score derived from the Fisher Information matrix of the L2 reconstruction loss, specifically approximating the Hessian with respect to spatial parameters. It employs a multi-round prune-refine pipeline to achieve high compression ratios (up to 90%) while preserving visual fidelity and foreground details better than single-round pruning or heuristic-based alternatives like LightGaussian.

### [Opacity Split for Cloning](9694-opacity-split-for-cloning.md)
Opacity Split for Cloning is a densification strategy that clones a single Gaussian into $N$ copies while minimizing the difference between the rasterization outcomes before and after cloning, ensuring the rendering result remains unchanged. It specifically addresses the opacity initialization of cloned Gaussians to maintain transmittance consistency, preventing the cloned child from having less impact than the parent alone due to the cumulative nature of alpha blending. It sets apart from standard cloning by explicitly deriving opacity and covariance adjustments to preserve the integral of the Gaussian's contribution along arbitrary slices, rather than simply inheriting the parent's opacity.

### [Progressive and Multi-stage Training](9966-progressive-and-multi-stage-training.md)
Progressive and Multi-stage Training methods address the instability and poor convergence of single-stage 3D Gaussian Splatting optimization by dividing the training process into distinct phases or stages. Unlike standard approaches that apply densification uniformly from the start, these methods progressively refine the Gaussian representation, balancing global structure capture with local detail refinement to achieve higher rendering quality [7173-hybrid-coarse-to-fine-strategy].

### [Gaussian Insertion and Pruning](10074-gaussian-insertion-and-pruning.md)
Gaussian Insertion and Pruning is a densification strategy designed for monocular and RGB-D SLAM systems using 3D Gaussian Splatting, specifically addressing the initialization of new Gaussians from depth estimates and the removal of geometrically unstable primitives. Unlike standard densification which splits existing Gaussians based on gradient norms, this method inserts new Gaussians at keyframes to capture newly visible scene elements and prunes those that fail multiview consistency checks. It is distinct from sibling methods like [Score-based Densification](4705-score-based-densification.md) by relying on visibility windows and depth-based initialization rather than per-step gradient magnitude thresholds.

### [TangoGS](10180-tangogs.md)
TangoGS is a density controller for 3D Gaussian Splatting that automatically scales the number of Gaussians across different capture sizes without per-capture retuning. It combines a capture-derived "learning allowance" (model sizing based on track-adjusted pixels) with training-based adaptation (quality feedback) to determine the final Gaussian population. Unlike sibling methods that require manual hyperparameter tuning for each scene, TangoGS uses a shared configuration for all captures, matching the mean PSNR of the best-performing baselines with significantly fewer Gaussians.

### [Hyperparameter Settings](11076-hyperparameter-settings.md)
Hyperparameter Settings define the numerical configuration for the differentiable optimization loop in 3D Gaussian Splatting variants, specifically governing the Adam optimizer, learning rates for geometric and photometric attributes, and regularization strengths. This method ensures stable convergence by preventing gradient overflow and material-lighting ambiguities that arise from erroneous Gaussian behavior, such as extreme opacity or scale updates. It is distinct from sibling methods like [Score-based Densification](4705-score-based-densification.md) by focusing on the *values* of the optimization constants rather than the *logic* of primitive management.

### [Adaptive Gaussian Refinement](11448-adaptive-gaussian-refinement.md)
Adaptive Gaussian Refinement is a technique that dynamically adjusts Gaussian parameters using neural network-predicted residuals to improve the modeling of complex non-rigid deformations and time-varying appearance. It addresses the limitation of static covariance matrices in standard 3D Gaussian Splatting, which cannot adequately represent dynamic behaviors such as motion-induced shape changes. Unlike sibling methods that rely on geometric heuristics or gradient norms for densification, this method uses lightweight MLPs to predict continuous corrections to position, time, direction, and covariance based on a contextual feature vector.

### [Hybrid and Diffusion Representations](11473-hybrid-and-diffusion-representations.md)
This family addresses the problem of generating 3D Gaussian Splatting (3DGS) representations from limited input data, such as a single image or text prompt, where traditional inverse rendering is ill-posed or computationally prohibitive. It sets itself apart from sibling methods like [Score-based Densification](4705-score-based-densification.md) by replacing iterative gradient-based densification with learned generative priors (diffusion or feed-forward transformers) to synthesize the initial Gaussian set directly.

#### GaussianSPA
- [S1, unofficial] Proposes an optimizing-sparsifying loop that interleaves standard training with a sparsification operator.
- [S1, unofficial] Removes redundant primitives while boosting the influence of the remaining collection.

#### Blur Splitting
- [S2] A densification step added to the 3DGS baseline that improves rendering quality and LPIPS metrics.
- [S2] Adding Blur Split to 3DGS baseline increases Gaussians from 3.35M to 3.74M, SSIM from 0.815 to 0.819, and LPIPS from 0.216 to 0.195.

#### Revised Densify and Prune Algorithm
- [S3] The complete densification and pruning algorithm integrating corrected scene extent, exponential ascending gradient threshold, and significance-aware pruning.
- [S3] Iterate over all Gaussians $g_k$ in the scene $G$.
- [S3] For densification: If the modified gradient magnitude $\tau'_k$ is greater than or equal to the threshold $T_i$, proceed to check splitting/cloning conditions.
- [S3] If the maximum scale $max(s_k)$ is greater than $P_{dense} \cdot e'_{scene}$, split the Gaussian $g_k$.
- [S3] Otherwise, clone the Gaussian $g_k$.
- [S3] For pruning: If the opacity $o_k$ is less than $o_{min}$ AND the accumulated significance $\sigma_k$ is in the bottom $N_{prune}$ of $\Sigma$, prune the Gaussian $g_k$.
- [S3] $N_{prune}$: The count of lowest-significance Gaussians to prune.
- [S3] Data: Scene of Gaussians G for gk ∈ G do //Densification if τ ′ k ≥ Ti then if max(sk) > Pdense · e ′ scene then splitGaussian(gk); else cloneGaussian(gk); end end //Significance Pruning if ok < omin then if σ*k* ∈ bottom(Nprune,Σ) then pruneGaussian(gk); end end end
- [S3] Implemented in the GitHub repository https://github.com/fraunhoferhhi/Improving-ADC-3DGS.

#### Trick-GS
- [S4] A Gaussian Splatting method that combines progressive training with resolution, noise and Gaussian scales, learning to prune and mask primitives and SH bands by their significance, and an accelerated GS training framework to achieve faster run-time, smaller disk size, and faster convergence compared to vanilla GS.
- [S4] 3DGS reconstructs a scene using 3D Gaussian primitives rendered via differentiable volume splatting rasterizer extending EWA volume splatting. Each primitive has 59 parameters: position, covariance (parameterized by scaling matrix S and quaternion rotation q), opacity, diffused color, and spherical harmonic (SH) coefficients (typically 3 bands).
- [S4] On three datasets, Trick-GS achieves up to 2× faster training, 40× smaller disk size and 2× faster rendering speed compared to vanilla GS, while having comparable accuracy.

#### Parallel Respawning and Densification with Edge Case Handling
- [S5, unofficial] A densification strategy where respawn and densify operations are performed in parallel, requiring specific edge case handling to prevent hierarchy degeneration.
- [S5, unofficial] If two sibling nodes both need to be respawned, only respawn the right node. This causes the left node to become a new dead leaf, which will be respawned in the next densification iteration.
- [S5, unofficial] If a node needs to be respawned whose sibling is not a leaf node, replace the entire parent node with the subtree of the node being respawned.
- [S5, unofficial] Apply a pruning strategy: zero the opacity of Gaussians that have not contributed for a number of iterations equal to twice the size of the training set. These will be respawned in the next densification iteration.

#### Opacity Reset & Densification Trigger
- [S6, unofficial] If the current iteration is a multiple of opt.opacity_reset_interval, or if the dataset uses a white background and the current iteration equals opt.densify_from_iter, call gaussians.reset_opacity().
- [S6, unofficial] opt.opacity_reset_interval: Iteration interval at which Gaussian opacities are reset during training.
- [S6, unofficial] opt.densify_from_iter: Iteration threshold that triggers densification specifically for white-background datasets.

#### Densify and Prune Optimization
- [S7, unofficial] Adjusting and removing the `densify_and_split` operation within the `densify_and_prune` process to improve speed without impacting rendering quality.

#### Adaptive Density Control (Split & Clone)
- [S8, unofficial] A mechanism to adjust the density of 3D Gaussians during optimization to match local scene complexity by duplicating or subdividing Gaussians based on their coverage relative to the target region.
- [S8, unofficial] During optimization, evaluate each Gaussian's coverage area against the target reconstruction region. If the Gaussian is too small (Under-Reconstruction), duplicate it (Clone). If the Gaussian is too large (Over-Reconstruction), split it into smaller Gaussians.
- [S8, unofficial] On the Average-30K dataset, Full achieves PSNR 26.05. No-Split drops to 23.90. No-Clone drops to 25.91.
- [S8, unofficial] Split removal primarily degrades background quality, causing large blurry regions. Clone removal primarily breaks thin structures (e.g., bicycle spokes), causing them to appear fragmented.

## Background
The concept of adaptive density control was introduced with the original 3D Gaussian Splatting paper by Kerbl et al. (2023), which established the baseline algorithm for splitting and cloning Gaussians based on 2D positional gradients [S1, unofficial][S8, unofficial]. Since then, numerous variations have been proposed to address specific limitations such as memory efficiency [S1, unofficial], dynamic scene modeling [S1, unofficial], and compression [S1, unofficial]. The field has evolved from simple gradient-threshold heuristics to more sophisticated methods involving probabilistic sampling [S1, unofficial], Fisher Information-based uncertainty [S1, unofficial], and learnable voxel selection [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
[S3] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S4] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
[S5] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
[S6] Yang-SyZng/AVM-GS, https://github.com/Yang-SyZng/AVM-GS, fetched 2026-10-27 (unofficial)
[S7] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
[S8] GaussianSplatting論文のAblationについての解説・考察, https://qiita.com/kotauchisunsun/items/f1e89f28aa770b8d3b19, fetched 2026-10-29 (unofficial)
