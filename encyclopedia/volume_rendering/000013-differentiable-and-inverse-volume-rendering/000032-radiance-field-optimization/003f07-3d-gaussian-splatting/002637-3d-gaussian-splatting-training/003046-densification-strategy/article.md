# Densification Strategy

Also called: Adaptive Density Control, Gaussian Population Management, Densification and Pruning

## Overview
Densification strategies manage the dynamic population of 3D Gaussian primitives $\mathcal{G}$ during the training of 3D Gaussian Splatting (3DGS) to balance reconstruction fidelity against computational cost. They solve the problem of static initialization by adapting the number, size, and distribution of Gaussians to scene complexity, preventing under-reconstruction (missing geometry) and over-reconstruction (redundant floaters). This family sets itself apart from static initialization methods by using view-space gradients, error metrics, or probabilistic criteria to trigger cloning, splitting, and pruning operations at regular intervals [S1, unofficial][S2][S8, unofficial].

## Shared principles
The core mechanism involves periodically evaluating the contribution or error associated with each primitive $g_k \in \mathcal{G}$ and modifying the population $\mathcal{G}$ accordingly. Common principles include:

1.  **Gradient/Error Tracking:** Most methods track a scalar metric per Gaussian, such as the magnitude of the 2D positional gradient $\tau_k = \frac{1}{N_{\text{views}}} \sum_{v} \| \frac{\partial L}{\partial \mu_v^k} \|_2$ [S4, S8, unofficial] or rendering error $E_k$ [S1, unofficial].
2.  **Periodic Execution:** Densification and pruning are not performed at every optimization step to allow the optimizer to stabilize. They are executed every $N$ iterations (e.g., `refineEvery` = 100) [S5, S8, unofficial].
3.  **Splitting and Cloning:** When a Gaussian exceeds a threshold, it is replaced by children. Splitting typically involves sampling new means from a normal distribution scaled by the parent's covariance, while cloning duplicates the parent [S8, unofficial].
4.  **Pruning:** Gaussians with low opacity $\alpha_k < \alpha_{\min}$ or low significance are removed to reduce memory footprint and prevent floaters [S1, unofficial][S2].
5.  **Buffer Management:** Pruned or split Gaussians require buffer compaction or flag-based marking to maintain contiguous memory access for rasterization [Parent].

## Subtopics

### [Score-based Densification](4705-score-based-densification.md)
Score-based Densification is a constructive densification strategy that controls model size by adding new Gaussians only to those primitives that maximize a specific quality score, adhering to a pre-defined budget. Unlike standard densification methods that rely on gradient thresholds and subsequent pruning to manage redundancy, this method uses a purely constructive schedule, ensuring the model size increases in a controlled manner toward an exact user-defined target without requiring culling during training. This approach solves the problems of unpredictable storage demands and fluctuating training times inherent in standard 3DGS, making it suitable for downstream tasks requiring fixed-size inputs. See [Score-based Densification](4705-score-based-densification.md).

### [GroupFlow](6351-groupflow.md)
GroupFlow is a motion compression technique for dynamic 3D Gaussian Splatting that distills per-Gaussian neural deformation fields into shared SE(3) transformations for groupwise motion. It solves the computational bottleneck of evaluating individual neural networks for every Gaussian at every timestep by clustering Gaussians into groups and assigning each group a single learned rigid-body trajectory. This reduces the number of transformations per frame from $N$ (per Gaussian) to $J$ (per group), accelerating rendering by up to 13.71× and shortening training time by 2.53× compared to baseline deformable methods. See [GroupFlow](6351-groupflow.md).

### [3DGS-MCMC](7852-3dgs-mcmc.md)
3DGS-MCMC replaces the noisy positional gradient magnitude heuristics of standard Adaptive Density Control with a probabilistic framework that reinterprets the Gaussian cloud as samples from an underlying probability distribution. It uses Markov Chain Monte Carlo (MCMC) sampling to guide the spawning and pruning of primitives, allowing the method to explore regions with unknown gradients and yield more efficient, artifact-free clouds. See [3DGS-MCMC](7852-3dgs-mcmc.md).

### [VS-Splat Voxel Selection](9132-vs-splat-voxel-selection.md)
VS-Splat Voxel Selection is a learnable mechanism for selecting relevant 3D Gaussian primitives from a voxel grid by predicting confidence scores via Gumbel-Sigmoid and using them to refine opacity values. It solves the problem of redundant background primitives and unreliable thresholding in sparse-view reconstruction. It sets itself apart from its sibling [Adaptive Density Control](9188-adaptive-density-control.md) by replacing static opacity thresholding with a differentiable, learned selection process that significantly reduces voxel count while improving rendering quality. See [VS-Splat Voxel Selection](9132-vs-splat-voxel-selection.md).

### [Adaptive Density Control](9188-adaptive-density-control.md)
Adaptive Density Control (ADC) is a dynamic optimization strategy used during the training of 3D Gaussian Splatting (3DGS) to adjust the number and spatial distribution of Gaussian primitives. It solves the problem of static initialization by dynamically adding, cloning, or removing Gaussians based on gradient magnitude, scale, and opacity, thereby preventing under-reconstruction (missing geometry) and over-reconstruction (coarse approximations). It sets itself apart from static initialization methods by using view-space positional gradient magnitudes to trigger cloning, splitting, and pruning operations at regular intervals, ensuring the representation remains compact while capturing fine details. See [Adaptive Density Control](9188-adaptive-density-control.md).

### [Pruning and Opacity Management](9314-pruning-and-opacity-management.md)
Pruning and Opacity Management removes redundant or low-impact primitives to prevent unbounded growth, reduce floaters, and stabilize the optimization of radiance fields. It differs from densification strategies, which increase primitive count, by actively reducing the population based on opacity, sensitivity, or shape attributes. While densification addresses under-reconstruction, this family addresses over-reconstruction and redundancy. See [Pruning and Opacity Management](9314-pruning-and-opacity-management.md).

### [Locality-preserving Densification](9585-locality-preserving-densification.md)
Locality-preserving densification addresses the memory layout bottleneck in 3D Gaussian Splatting training, where the addition of new Gaussians to the end of parameter buffers causes spatially close primitives to be stored far apart in memory. By reordering the Gaussian population based on 3D spatial proximity using Z-ordering (Morton order), this method ensures that neighboring Gaussians are adjacent in memory, thereby improving memory access coalescing. It sets itself apart from sibling methods like [Score-based Densification](4705-score-based-densification.md) by focusing on hardware efficiency and training speed rather than altering the geometric distribution or selection criteria of the Gaussians themselves. See [Locality-preserving Densification](9585-locality-preserving-densification.md).

### [Progressive and Multi-stage Training](9966-progressive-and-multi-stage-training.md)
Progressive and Multi-stage Training refers to a family of optimization strategies that decouple the initialization, densification, and refinement of 3D Gaussian primitives into distinct phases or stages. It solves the problem of optimization instability and poor convergence that arises when attempting to simultaneously initialize a sparse representation, densify it to cover complex geometry, and optimize high-frequency attributes in a single monolithic process. Unlike single-stage training methods that optimize a static or dynamically densified set from scratch, this family sets itself apart by structuring the learning trajectory—either through coarse-to-fine geometric refinement, decoupled loss supervision, or staged network initialization—to achieve higher fidelity and stability. See [Progressive and Multi-stage Training](9966-progressive-and-multi-stage-training.md).

### [Gaussian Insertion and Pruning](10074-gaussian-insertion-and-pruning.md)
Gaussian Insertion and Pruning is a dynamic population management strategy used in monocular and RGB-D Gaussian Splatting SLAM systems to maintain a coherent 3D structure while tracking camera motion. Unlike static densification methods that rely solely on gradient magnitude, this method inserts new Gaussians at keyframes to capture newly visible scene elements and prunes geometrically unstable primitives based on multi-view visibility consistency. It addresses the problem of drift and floating artifacts in SLAM by ensuring that only Gaussians observed across multiple frames in the current keyframe window are retained. See [Gaussian Insertion and Pruning](10074-gaussian-insertion-and-pruning.md).

### [TangoGS](10180-tangogs.md)
TangoGS is a density controller for 3D Gaussian Splatting that automatically scales the Gaussian population size across different capture scales without per-capture hyperparameter tuning. It combines a capture-derived "learning allowance" (model sizing based on track-adjusted pixels) with training-based quality feedback (PSNR-driven adaptation) to determine the final number of Gaussians. Unlike sibling methods that rely on fixed defaults or per-capture tuning, TangoGS ensures the model budget is invariant to redundant views (e.g., duplicated images) and adapts dynamically to reconstruction quality. See [TangoGS](10180-tangogs.md).

### [Hyperparameter Settings](11076-hyperparameter-settings.md)
Hyperparameter settings define the numerical configuration for the optimizer, learning rates, and regularization terms used during the training of 3D Gaussian Splatting variants. This article focuses on the specific values required to stabilize optimization, particularly in physically-based inverse rendering contexts where material-lighting ambiguities can cause instability. It sets apart from general training guides by providing exact learning rates for PBR parameters (base color, metallic, roughness) and regularization strengths that prevent gradient overflow caused by erroneous Gaussians. See [Hyperparameter Settings](11076-hyperparameter-settings.md).

### [Adaptive Gaussian Refinement](11448-adaptive-gaussian-refinement.md)
Adaptive Gaussian Refinement is a technique that dynamically adjusts Gaussian parameters using neural network-predicted residuals to improve the modeling of complex non-rigid deformations and time-varying appearance. It addresses the limitation of static covariance matrices in standard 3D Gaussian Splatting, which cannot adequately represent dynamic shape changes or motion-induced deformations. Unlike sibling methods that rely on gradient-based densification triggers [Parent], this method operates continuously during rendering by applying learned corrections to the base Gaussian parameters before conditional slicing. See [Adaptive Gaussian Refinement](11448-adaptive-gaussian-refinement.md).

### [Hybrid and Diffusion Representations](11473-hybrid-and-diffusion-representations.md)
This family addresses the problem of generating a complete set of 3D Gaussians $\mathcal{G}$ from a single input (image or text) without per-scene optimization, which is required by methods like [Adaptive Density Control](9188-adaptive-density-control.md). It sets itself apart by using feed-forward neural networks or diffusion models to predict Gaussian parameters directly, rather than optimizing them via gradient descent on a fixed or dynamically densified set. See [Hybrid and Diffusion Representations](11473-hybrid-and-diffusion-representations.md).

### [Pruning and Sparsification](12354-pruning-and-sparsification.md)
Pruning and Sparsification are mechanisms within 3D Gaussian Splatting training that remove redundant or low-contribution Gaussians to reduce memory footprint and improve rendering efficiency. Unlike densification strategies which increase Gaussian count to resolve reconstruction errors, pruning reduces the population $\mathcal{G}$ by eliminating primitives that contribute minimally to the final image or exhibit high uncertainty. This family sets itself apart from simple opacity culling by incorporating significance-aware metrics, uncertainty-based Fisher Information scores, or depth-dependent thresholds to preserve spatially significant or distant structures that standard heuristics might incorrectly discard. See [Pruning and Sparsification](12354-pruning-and-sparsification.md).

### [Texel Size Adaptation and Primitive Management](12592-texel-size-adaptation-and-primitive-management.md)
This method manages the capacity of 3D Gaussian splats by dynamically adjusting the resolution of their associated texture maps (texel size) and the number of active primitives based on rendering error. It solves the problem of balancing appearance quality against memory and computational cost by ensuring that high-error regions receive more resources (more texels or more primitives) while low-error regions are simplified. It differs from sibling methods like [Adaptive Density Control](9188-adaptive-density-control.md) by explicitly linking texture resolution to primitive splitting and by using a quantile-based error threshold to drive these decisions, rather than relying solely on gradient magnitudes or opacity thresholds. See [Texel Size Adaptation and Primitive Management](12592-texel-size-adaptation-and-primitive-management.md).

### GaussianSPA
- [S1, unofficial] Proposes an optimizing-sparsifying loop that interleaves standard training with a sparsification operator.
- [S1, unofficial] Removes redundant primitives while boosting the influence of the remaining collection.

### Blur Splitting
- [S2] A densification step added to the 3DGS baseline that improves rendering quality and LPIPS metrics.
- [S2] Adding Blur Split to 3DGS baseline increases Gaussians from 3.35M to 3.74M, SSIM from 0.815 to 0.819, and LPIPS from 0.216 to 0.195.

### Trick-GS
- [S3] A Gaussian Splatting method that combines progressive training with resolution, noise and Gaussian scales, learning to prune and mask primitives and SH bands by their significance, and an accelerated GS training framework to achieve faster run-time, smaller disk size, and faster convergence compared to vanilla GS.
- [S3] 3DGS reconstructs a scene using 3D Gaussian primitives rendered via differentiable volume splatting rasterizer extending EWA volume splatting. Each primitive has 59 parameters: position, covariance (parameterized by scaling matrix S and quaternion rotation q), opacity, diffused color, and spherical harmonic (SH) coefficients (typically 3 bands).
- [S3] On three datasets, Trick-GS achieves up to 2× faster training, 40× smaller disk size and 2× faster rendering speed compared to vanilla GS, while having comparable accuracy.

### Parallel Respawning and Densification with Edge Case Handling
- [S4, unofficial] A densification strategy where respawn and densify operations are performed in parallel, requiring specific edge case handling to prevent hierarchy degeneration.
- [S4, unofficial] If two sibling nodes both need to be respawned, only respawn the right node. This causes the left node to become a new dead leaf, which will be respawned in the next densification iteration.
- [S4, unofficial] If a node needs to be respawned whose sibling is not a leaf node, replace the entire parent node with the subtree of the node being respawned.
- [S4, unofficial] Apply a pruning strategy: zero the opacity of Gaussians that have not contributed for a number of iterations equal to twice the size of the training set. These will be respawned in the next densification iteration.

### Densify and Prune Optimization
- [S5, unofficial] Adjusting and removing the `densify_and_split` operation within the `densify_and_prune` process to improve speed without impacting rendering quality.

### Termination entropy criterion
- [S6, unofficial] Uses termination entropy as the criterion for calculating information gain in active 3D reconstruction.
- [S6, unofficial] Performs consistently better than baselines across all scenes, allowing the agent to explore more diverse viewpoints and build better 3D models with a limited number of viewpoints.

### Outlier Removal
- [S7] A process to address noise and missing surface parts in the density space by using perceptual loss to identify problematic geometric aspects and removing outliers.
- [S7] Compute perceptual loss VGG(S_p, S_g) to measure high-level visual similarity between rendered patch S_p and real image patch S_g.
- [S7] Select patches where VGG(S_p, S_g) is lower than half the average value across the training set to prioritize patches with improved visual similarity.
- [S7] Augment the density space by adding a group of points along the pixel ray for selected patches, considering the depth range of neighboring pixels.
- [S7] Remove outliers by examining the number of points within a specified radius around each neural density point; if the count falls below a predefined threshold, the point is removed.

### Joint Optimization
- [S8, unofficial] Joint optimization of newly added view-dependent Gaussians with pre-existing kernels, refining $T$, $\beta$, and opacity orientation using multi-view photometric constraints.
- [S8, unofficial] During image-space optimization, original Gaussians' rendered outputs are treated as background. After projection, original kernels' opacity is made optimizable while freezing other attributes to mitigate over-occlusion.
- [S8, unofficial] Apply full parameter optimization to newly added Gaussians. Set number of iterations proportional to training set size to ensure each additional kernel is sampled approximately the same number of times.
- [S8, unofficial] Optimize parameters with specific learning rates: position (0.000016), diffuse color (0.001), higher-order spherical harmonic coefficients (0.001/20), opacity (0.02), scaling (0.002), rotation (0.0005), opacity lobe orientation `d` (0.001), opacity lobe width `T` (0.0002), and sharpness `β` (0.002).
- [S8, unofficial] Employ SparseAdam to optimize the opacity of original Gaussian primitives visible in the current viewpoint.

## Background
3D Gaussian Splatting (3DGS) was introduced by Kerbl et al. in 2023 [S3]. The original method used a static initialization from Structure-from-Motion (SfM) points and a fixed densification schedule based on positional gradients. Subsequent research has expanded the family of densification strategies to address limitations in memory efficiency, dynamic scene handling, and sparse-view reconstruction. Methods like [Score-based Densification](4705-score-based-densification.md) and [TangoGS](10180-tangogs.md) focus on controlling model size, while [3DGS-MCMC](7852-3dgs-mcmc.md) and [GroupFlow](6351-groupflow.md) introduce probabilistic and motion-compression techniques. The family continues to evolve with integration into SLAM [Gaussian Insertion and Pruning](10074-gaussian-insertion-and-pruning.md) and physically-based rendering [Hyperparameter Settings](11076-hyperparameter-settings.md).

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
[S3] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
[S4] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
[S5] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
[S6] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
[S7] DGNR: Density-Guided Neural Point Rendering of Large Driving Scenes, https://doi.org/10.1109/tase.2024.3410891, fetched 2026-10-27
[S8] Augmented Radiance Field: A General Framework for Enhanced Gaussian Splatting, https://doi.org/10.48550/arXiv.2602.19916, fetched 2026-10-30 (unofficial)
