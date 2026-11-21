# Densification Strategy

Also called: Adaptive Density Control, Gaussian Densification, Primitive Management

## Overview
Densification strategies are dynamic optimization mechanisms used during the training of 3D Gaussian Splatting (3DGS) to adjust the number and spatial distribution of Gaussian primitives [S1, unofficial][S8, unofficial]. They solve the problem of under-reconstruction (missing geometry in sparse regions) and over-reconstruction (coarse approximations or artifacts in simple regions) by periodically cloning, splitting, and pruning Gaussians based on gradient magnitudes, screen size, and opacity [S1, unofficial][S8, unofficial]. Unlike static initialization methods, these strategies use view-space positional gradient magnitudes to trigger densification operations at regular intervals, ensuring the representation remains compact while capturing fine details [S1, unofficial][S2].

## Shared principles
The family shares the goal of managing the primitive count to match local scene complexity. Common mechanisms include:
1.  **Gradient-Based Triggering:** Densification is typically triggered by the magnitude of the 2D positional gradient $\frac{\partial L_v}{\partial \mu_v^k}$ for each Gaussian $g_k$ across training views [S4][S8, unofficial]. A Gaussian becomes a candidate if its average gradient magnitude $\tau_k$ exceeds a threshold $T_{grad}$ [S8, unofficial].
2.  **Split vs. Clone Decision:** For candidates, the action depends on the Gaussian's size relative to the scene extent $e_{scene}$ and a density threshold $P_{dense}$ [S8, unofficial]. If the maximum scaling factor $\max(s_k) > P_{dense} \cdot e_{scene}$, the Gaussian is split into smaller children to resolve over-reconstruction [S8, unofficial]. If $\max(s_k) \le P_{dense} \cdot e_{scene}$, it is cloned to resolve under-reconstruction [S8, unofficial].
3.  **Opacity Management:** Pruning and opacity resets are used to remove redundant "floaters" and stabilize optimization. Gaussians with opacity $\alpha$ below a threshold $\alpha_{min}$ are removed or clamped [S1, unofficial][S2]. Opacity may be periodically reset to allow re-optimization [S6, unofficial].
4.  **Periodic Execution:** Densification and pruning are performed periodically (e.g., every `refineEvery` steps) rather than at every step to allow the optimizer to stabilize [S1, unofficial].
5.  **Buffer Management:** Pruned primitives are removed from the active set, often requiring buffer compaction or marking for garbage collection [Parent].

## Subtopics

### [Score-based Densification](4705-score-based-densification.md)
Score-based Densification is a constructive densification strategy that controls model size by steering the addition of new Gaussians toward primitives that maximize reconstruction quality, adhering to a strict user-defined budget. Unlike standard Adaptive Density Control which relies on gradient norms and opacity thresholds to trigger splits or clones, this method uses a composite score combining gradient, saliency, and primitive properties to select candidates, ensuring a predictable growth curve without requiring intermediate pruning or culling.

### [GroupFlow](6351-groupflow.md)
GroupFlow is a motion compression technique for dynamic 3D Gaussian Splatting that distills per-Gaussian neural deformation fields into shared SE(3) transformations for groupwise motion. It solves the computational bottleneck of evaluating individual neural networks for every Gaussian at every timestep by clustering Gaussians into groups and assigning each group a single time-varying rigid transformation. Unlike sibling methods that may retain per-Gaussian neural fields or use global rigid assumptions, GroupFlow captures locally coherent motion through learned control trajectories, balancing fidelity and efficiency.

### [3DGS-MCMC](7852-3dgs-mcmc.md)
3DGS-MCMC replaces the noisy positional gradient magnitude heuristics of standard 3D Gaussian Splatting with a Markov Chain Monte Carlo (MCMC) framework to guide the spawning and pruning of Gaussian primitives. It reinterprets the Gaussian cloud as samples from an underlying probability distribution, using Metropolis–Hastings sampling to explore regions with unknown gradients, thereby yielding more efficient and artifact-free reconstructions.

### [VS-Splat Voxel Selection](9132-vs-splat-voxel-selection.md)
VS-Splat voxel selection is a learnable, differentiable method for filtering 3D Gaussian primitives by predicting confidence scores via Gumbel-Sigmoid and using them to refine opacity values. It solves the problem of redundant background primitives and manual threshold tuning inherent in opacity-based thresholding (OBT). It sets itself apart from siblings like [Score-based Densification](4705-score-based-densification.md) by operating on a pre-generated voxel grid to select object-centric voxels rather than adapting primitive density via splitting/cloning, and from [Adaptive Density Control](9188-adaptive-density-control.md) by using a stochastic Gumbel-Sigmoid relaxation instead of hard gradient-norm thresholds.

### [Adaptive Density Control](9188-adaptive-density-control.md)
Adaptive Density Control (ADC) is a dynamic optimization strategy used during the training of 3D Gaussian Splatting (3DGS) to adjust the number and spatial distribution of Gaussian primitives. It solves the problem of under-reconstruction (missing geometry in sparse regions) and over-reconstruction (coarse approximations or artifacts in simple regions) by periodically cloning, splitting, and pruning Gaussians based on gradient magnitudes, screen size, and opacity. It sets itself apart from static initialization methods by using view-space positional gradient magnitudes to trigger densification operations at regular intervals, ensuring the representation remains compact while capturing fine details.

### [Pruning and Opacity Management](9314-pruning-and-opacity-management.md)
Pruning and Opacity Management are mechanisms to control the number of primitives (Gaussians, splats, or triangles) during training, preventing unbounded growth, removing redundant "floaters," and stabilizing the optimization process. Unlike densification strategies that add primitives, these methods remove them based on opacity, sensitivity, or significance metrics. They are distinct from sibling methods like [Score-based Densification](4705-score-based-densification.md) which focuses on gradient magnitude for splitting, and [Adaptive Density Control](9188-adaptive-density-control.md) which uses opacity resets and size thresholds in standard 3DGS.

### [Locality-preserving Densification](9585-locality-preserving-densification.md)
Locality-preserving densification addresses the memory access throttling caused by standard Gaussian densification, where new primitives are appended to the end of parameter buffers, causing spatially close Gaussians to be stored far apart in memory. By reordering the Gaussian set based on 3D spatial proximity using Z-ordering (Morton order), it ensures that neighboring Gaussians in space are also close in memory, thereby improving memory access coalescing. It sets itself apart from sibling methods like [Score-based Densification](4705-score-based-densification.md) or [Adaptive Density Control](9188-adaptive-density-control.md) by focusing on data layout optimization rather than geometric splitting criteria, though it is typically applied in conjunction with them.

### [Opacity Split for Cloning](9694-opacity-split-for-cloning.md)
Opacity Split for Cloning is a method used during the densification phase of 3D Gaussian Splatting to clone existing Gaussians while preserving the visual fidelity of the rendered image. It solves the problem of "transmittance bias" introduced when a Gaussian is duplicated: in standard alpha blending, a cloned child inheriting the parent's full opacity causes the combined transmittance to drop more than intended, artificially darkening the background or subsequent primitives. The method ensures that the rasterization outcome remains unchanged by adjusting the opacity of the parent and clones according to a specific splitting formula.

### [Progressive and Multi-stage Training](9966-progressive-and-multi-stage-training.md)
Progressive and Multi-stage Training refers to optimization strategies that decompose the densification and refinement of 3D Gaussian primitives into distinct temporal phases or hierarchical levels. It solves the instability inherent in optimizing sparse, randomly initialized point clouds by first establishing a stable global structure before refining high-frequency details. Unlike single-stage training, which often suffers from slow convergence or poor global coherence, these methods yield significantly higher PSNR and SSIM [7173-hybrid-coarse-to-fine-strategy.md].

### [Gaussian Insertion and Pruning](10074-gaussian-insertion-and-pruning.md)
Gaussian Insertion and Pruning is a dynamic primitive management strategy used in monocular and RGB-D SLAM pipelines based on 3D Gaussian Splatting. It addresses the problem of maintaining a coherent 3D structure and capturing newly visible scene elements during camera motion, which static densification strategies cannot handle. Unlike sibling methods that focus on optimizing reconstruction error for static scenes, this method relies on temporal visibility consistency across a sliding window of keyframes to insert new Gaussians and prune geometrically unstable ones.

### [TangoGS](10180-tangogs.md)
TangoGS is a density control strategy for 3D Gaussian Splatting that automatically scales the number of Gaussians to the complexity of the capture without per-scene hyperparameter tuning. It combines a capture-derived "learning allowance" (model sizing based on track-adjusted pixels) with training-based quality feedback (PSNR-driven pruning) to determine the final Gaussian population. It sets itself apart from siblings like [Adaptive Density Control](9188-adaptive-density-control.md) by decoupling the global model budget from the optimization loop, ensuring the model size is invariant to redundant views and scales predictably with capture extent.

### [Hyperparameter Settings](11076-hyperparameter-settings.md)
Hyperparameter settings define the numerical configuration for the optimizer, learning rates, and regularization terms used during the training of 3D Gaussian Splatting (3DGS) and its variants. These settings are critical for stabilizing the optimization process, preventing gradient overflow, and ensuring convergence to a physically plausible or high-fidelity reconstruction. They differ from sibling methods like [Score-based Densification](4705-score-based-densification.md) by focusing on the *magnitude* and *decay* of updates rather than the *structural* modification of the Gaussian set, though they operate concurrently within the training loop.

### [Adaptive Gaussian Refinement](11448-adaptive-gaussian-refinement.md)
Adaptive Gaussian Refinement is a technique that dynamically adjusts Gaussian parameters using neural network-predicted residuals to improve the modeling of complex non-rigid deformations and time-varying appearance. It addresses the limitation of static covariance matrices in representing dynamic behaviors by allowing spatial position, temporal coordinate, directional mean, and covariance to vary as functions of observed time. It is distinct from standard densification strategies (like [Adaptive Density Control](9188-adaptive-density-control.md)) because it modifies existing primitives rather than splitting or cloning them, and it is specific to high-dimensional (7D) Gaussian representations that include temporal and angular dimensions.

### [Hybrid and Diffusion Representations](11473-hybrid-and-diffusion-representations.md)
Hybrid and Diffusion Representations address the problem of generating 3D Gaussian Splatting (3DGS) scenes from single images or text prompts without iterative per-scene optimization. Unlike sibling methods that rely on gradient-based densification [Densification Strategy](6234-densification-strategy.md) or score-based refinement [Score-based Densification](4705-score-based-densification.md), these approaches use learned generative priors to predict a complete set of Gaussians in a single forward pass. They are distinguished by their ability to synthesize geometry and attributes directly from conditions, bypassing the need for initial point cloud initialization and subsequent gradient descent.

### [Pruning and Sparsification](12354-pruning-and-sparsification.md)
Pruning and Sparsification are post-training or in-training strategies that reduce the number of 3D Gaussians in a scene to improve rendering efficiency, reduce memory footprint, or correct reconstruction artifacts. Unlike [Densification Strategy](6234-densification-strategy.md), which increases density to capture detail, this family removes redundant or insignificant primitives. It is distinguished from sibling methods by its focus on removing Gaussians based on specific criteria such as accumulated rendering significance, second-order uncertainty (Fisher Information), or spatial depth, rather than simple opacity thresholds or gradient norms alone.

### [Texel Size Adaptation and Primitive Management](12592-texel-size-adaptation-and-primitive-management.md)
This method manages the capacity of 3D Gaussian primitives and their associated texture maps by adapting texel resolution and splitting primitives based on quantile error thresholds. It solves the problem of balancing model complexity against appearance quality by ensuring that high-error regions receive more geometric detail (via splitting) and higher texture resolution (via upscaling), while low-error regions are simplified. It differs from sibling methods like [Score-based Densification](4705-score-based-densification.md) by explicitly linking the number of texels per primitive to the geometric density, rather than treating texture resolution as a static or globally uniform parameter.

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
The concept of adaptive density control was introduced in the original 3D Gaussian Splatting paper by Kerbl et al. (2023) [S8, unofficial]. The method was designed to dynamically adjust the number of Gaussians to match the complexity of the scene, addressing the limitations of static point cloud initialization. Since then, numerous variants have been proposed to improve efficiency, quality, and robustness, including score-based methods [S1, unofficial], MCMC-based approaches [S1, unofficial], and dynamic scene adaptations [S1, unofficial]. The family has expanded to include specialized strategies for SLAM [S1, unofficial], compression [S2], and neural network-based refinement [S1, unofficial].

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
