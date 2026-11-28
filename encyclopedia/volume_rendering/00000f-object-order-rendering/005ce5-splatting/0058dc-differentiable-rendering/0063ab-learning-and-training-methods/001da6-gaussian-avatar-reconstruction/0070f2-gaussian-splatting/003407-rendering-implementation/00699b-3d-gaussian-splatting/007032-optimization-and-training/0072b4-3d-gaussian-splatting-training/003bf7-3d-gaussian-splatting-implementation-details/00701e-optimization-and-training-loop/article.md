# Optimization and Training Loop

Also called: 3D Gaussian Splatting Training, 3DGS Optimization

## Overview
This family encompasses the iterative optimization procedures used to adjust the parameters of 3D Gaussian primitives (positions, covariances, opacities, and spherical harmonic coefficients) to minimize the photometric error between rendered images and ground-truth inputs. It differs from sibling methods like [TurboGS](15697-turbogs.md) or [Distributed 3DGS Training Loop](18650-distributed-3dgs-training-loop-with-adaptive-workload-divisi.md) by relying on standard stochastic gradient descent with fixed iteration budgets and explicit densification/pruning heuristics, rather than adaptive workload division or accelerated rendering pipelines [S2, unofficial][S1].

## Shared principles
The core mechanism common to all training loops in this family is the minimization of a composite loss function $\mathcal{L}$ between the rendered image $\hat{I}$ and the ground-truth image $I_{gt}$ [S1][S2, unofficial]. The standard photometric baseline is a weighted sum of L1 loss and differentiable Structural Similarity (D-SSIM) [S1][S2, unofficial]:
$$ \mathcal{L}_{img} = (1 - \lambda) \mathcal{L}_{L1} + \lambda \mathcal{L}_{D-SSIM} $$
where $\mathcal{L}_{L1}$ is the average absolute difference between corresponding pixel colors, and $\mathcal{L}_{D-SSIM}$ measures local structure, contrast, and texture [S2, unofficial]. The weight $\lambda$ is typically 0.2 [S2, unofficial][S1].

The optimization uses the Adam optimizer with $\beta_1 = 0.9$ and $\beta_2 = 0.99$ [S1][S2, unofficial]. Training typically runs for a fixed number of iterations, commonly around 30,000 for standard scenes, though specific schedules vary (e.g., [Gaussian Primitive Training Schedule](18977-gaussian-primitive-training-schedule.md) uses ~6,000) [S2, unofficial]. The process does not stop at a specific loss threshold because the loss curve continues to creep down with diminishing returns; a fixed iteration budget is used instead [S2, unofficial].

Gaussian parameters $\mathcal{G} = \{g_i\}_{i=1}^N$ include position $\mathbf{p}_i \in \mathbb{R}^3$, covariance $\Sigma_i$ (parameterized by rotation quaternion $\mathbf{q}_i \in \mathbb{R}^4$ and scale vector $\mathbf{s}_i \in \mathbb{R}^3$), opacity $\alpha_i \in [0, 1]$, and Spherical Harmonics (SH) coefficients $\mathbf{c}_{i, l}$ for view-dependent color [S2, unofficial][S1].

## Subtopics

### [3D Gaussian Splatting Training Loop](15139-3d-gaussian-splatting-training-loop.md)
The standard iterative process minimizing photometric error via Adam optimizer, random view sampling, and periodic densification/pruning heuristics.

### [TurboGS](15697-turbogs.md)
An error-guided framework that accelerates optimization by dynamically focusing computation on perceptually informative pixels using persistent error and age maps, replacing dense rasterization with tile-wise sparse sampling.

### [Error-Guided Sparse Pixel Sampling](15708-error-guided-sparse-pixel-sampling.md)
A strategy restricting loss evaluation to a sparse subset of pixels per tile, guided by error and age maps, to reallocate computation from well-reconstructed regions.

### [Frequency Guided Resolution Scheduler](15751-frequency-guided-resolution-scheduler.md)
A strategy that dynamically adjusts rendering resolution based on the frequency content (Fourier energy) of training data, allocating high-resolution iterations only after low-frequency structure is captured.

### [Multi-View Regulated Gaussian Splatting](15783-multi-view-regulated-gaussian-splatting.md)
Aggregates gradients from multiple views in each step to enforce multi-view coherence, summing gradients rather than averaging them to stabilize descent.

### [Mip-Splatting Training Loop & Evaluation Pipeline](17975-mip-splatting-training-loop-evaluation-pipeline.md)
Extends the standard loop with subpixel-aware rendering to mitigate aliasing, managing subpixel offsets and adaptive densification schedules.

### [Distributed 3DGS Training Loop with Adaptive Workload Division](18650-distributed-3dgs-training-loop-with-adaptive-workload-divisi.md)
Implements distributed training across multiple GPUs, batching camera selection and dividing rendering workloads based on historical performance.

### [Sparse SH Order Increment](18689-sparse-sh-order-increment.md)
A memory-efficient strategy that initializes all Gaussians with SH order 0 and selectively promotes only those with significant rendering errors to higher orders.

### [Gaussian Splatting Training with SDS Loss](18710-gaussian-splatting-training-with-sds-loss.md)
Optimizes Gaussians using Score Distillation Sampling (SDS) from a pretrained 2D diffusion model, enabling reconstruction from unstructured images or text prompts without ground-truth 3D geometry.

### [Regression-based FisherRF](18747-regression-based-fisherrf.md)
Estimates uncertainty by grouping Gaussian parameter variances into six categories and feeding them into a meta-regression model, rather than aggregating variances directly.

### [Gaussian Primitive Training Schedule](18977-gaussian-primitive-training-schedule.md)
Accelerates training by leveraging dense initial point clouds and interrupting the loop with pseudo-view depth optimization, reducing iterations from ~30,000 to ~6,000.

### [Loss Function Design](19337-loss-function-design.md)
Defines the composite loss function $\mathcal{L}$, establishing the photometric baseline and framework for extending with geometric or semantic constraints.

### [Immediate 3D Gaussian Splat Reconstruction](20247-immediate-3d-gaussian-splat-reconstruction.md)
A streaming pipeline for incremental reconstruction from unordered images, maintaining global consistency via loop closure and covisibility graphs.

### [Active View Selection and Residual Learning for 3DGS](20899-active-view-selection-and-residual-learning-for-3dgs.md)
Combines active view selection with residual learning to improve quality under sparse view conditions, using self-augmented pre-filtering to stabilize selection.

### [Uncertainty-Guided Residual Learning for 3DGS](20901-uncertainty-guided-residual-learning-for-3dgs.md)
Amplifies gradient flow to weakly contributing, high-uncertainty Gaussians by combining full-scene renders with uncertainty-intensified renders.

### [Submap-based Gaussian SLAM Mapping](21005-submap-based-gaussian-slam-mapping.md)
Manages incremental construction in SLAM by dividing the scene into submaps, resetting optimization state periodically to prevent drift.

### [3D Gaussian Splatting Optimization](22286-3d-gaussian-splatting-optimization.md)
Details the optimization schedule, including resolution warm-up (starting at 1/4 resolution) and staged introduction of Spherical Harmonics bands.

### [Gradient Modification](22461-gradient-modification.md)
Adjusts the backward pass to address vanishing gradients for distant primitives or gradient confusion, intercepting gradients to provide non-zero signals.

### [Pose and Camera Optimization](23423-pose-and-camera-optimization.md)
Estimates camera poses by minimizing photometric error, treating poses as learnable parameters rather than fixed inputs.

### DiffGS Framework
- **Step**: Applies an optimization process during Gaussian extraction to optimize Gaussians to exact locations [S1].
- **Parameter**: Truncation function $\lambda$ is introduced when modeling GauPF (Gaussian Probability Function). Projection function $\tau$ is implemented either as $\tau(x) = e^{-x}$ (Exponent) or as a linear projection [S1].
- **Number**: 50K Gaussians: PSNR 28.61, SSIM 0.9787, LPIPS 0.0251. 100K Gaussians: PSNR 30.35, SSIM 0.9838, LPIPS 0.0151. 350K Gaussians: PSNR 34.01, SSIM 0.9879, LPIPS 0.0149 [S1].

### PyGS Optimization
- **Definition**: The optimization process using a combined MSE and SSIM loss, along with density control strategies adapted from 3DGS, while preserving the multi-scale structure [S2, unofficial].
- **Step**: Apply splitting and cloning strategies from 3DGS for density control. Newly split/cloned Gaussians remain in their original level and cluster. Routinely reassess and reassign the levels of Gaussians based on their scales to preserve the multi-scale characteristics and balanced distribution across levels [S2, unofficial].
- **Formula**: $L = \lambda \cdot MSE(C, \hat{C}) + (1 - \lambda) \cdot SSIM(C, \hat{C})$ where $\lambda$ is a hyperparameter balancing the terms, $C$ is ground-truth color, and $\hat{C}$ is rendered color [S2, unofficial].

### Progressive Resolution Scheduling
- **Definition**: A training strategy that gradually increases rendering resolution during training to improve convergence speed and reduce computational overhead, starting from low resolution and increasing toward the target [S3].
- **Formula**: $R_t = R_{max} \cdot (\frac{t}{T})^\gamma$, where $R_t$ is the resolution at training step $t$, $R_{max}$ is the final resolution, $T$ is the total training steps, and $\gamma$ controls the growth rate [S3].
- **Parameter**: Initial resolution: 540x960. Final resolution: 1080x1920 [S3].

### Frequency-Spectrum Regularization
- **Definition**: A method that uses differences in frequency spectra of rendered and reference images to progressively regularize the model from coarse, low-frequency structures to fine, high-frequency details [S4].

### SuGaR Pipeline
- **Definition**: A pipeline to extract meshes from 3D Gaussian Splatting and build hybrid representations that enable easy composition and animation in Gaussian Splatting scenes by manipulating the mesh [S5, unofficial].
- **Step**: 1. Short vanilla 3DGS optimization: optimizing a vanilla 3D Gaussian Splatting model for 7k iterations, in order to let Gaussians position themselves in the scene. 2. SuGaR optimization: optimizing Gaussians alignment with the surface of the scene. 3. Mesh extraction: extracting a mesh from the optimized Gaussians. 4. SuGaR refinement: refining the Gaussians and the mesh together to build a hybrid Mesh+Gaussians representation. 5. Textured mesh extraction (Optional): extracting a traditional textured mesh from the refined SuGaR model as a tool for visualization, composition and animation in Blender using our Blender add-on [S5, unofficial].

### Geometric Initialization
- **Definition**: Initializing 3D Gaussian primitives directly from the retrieved CAD mesh surface rather than SfM point clouds, providing strong inductive bias [S6, unofficial].
- **Step**: Define target number of primitives $N_{splats}$. Draw area-weighted uniform random sample of points from mesh surface to obtain initial canonical positions $\mu_{can}^{(i)}$. Recover surface normals $n_i$ by casting ray from sample toward mesh interior and reading triangle normal of intersected face. Initialize rotation quaternion $q_i$ to shortest-arc rotation aligning local z-axis with $n_i$, creating oriented disk tangent to mesh [S6, unofficial].
- **Pitfall**: Standard 3DGS densification/pruning (cloning/splitting) is not employed to maintain strict adherence to the CAD surface geometry [S6, unofficial].

### C LUNO GS
- **Definition**: A compact framework for Language 3DGS that combines semantics-aware Gaussian sparsification and attribute vector quantization in a unified constrained objective, solved via an alternating primal-dual procedure [S7, unofficial].
- **Step**: The optimization alternates among four updates: 1) Primal Update (differentiable scene optimization), 2) Sparsification Update (cardinality projection on opacities), 3) Vector Quantization Update (quantization of attributes), and 4) Dual Update (Lagrange multiplier updates) [S7, unofficial].
- **Formula**: Rendering loss: $L_{RGB} = (1 - \lambda)L_1 + \lambda L_{D-SSIM}$, where $\lambda \in [0, 1]$ balances the terms [S7, unofficial].

### Gaussian Optimization with Photometric Gradients
- **Definition**: A refinement step that optimizes the spherical harmonic coefficients and Gaussian structures using visual-derived photometric gradients to enhance photorealism and density [S8].
- **Step**: Optimize the spherical harmonic coefficients of the Gaussians using photometric gradients derived from captured images. Refine the LiDAR Gaussian structures using images captured from various perspectives [S8].

### Opacity Sparsity Loss
- **Definition**: A modification to the training loss function that adds an L1 sparsity term on opacity to encourage lower opacity values, facilitating effective culling [S9].
- **Step**: Add an L1 sparsity term on opacity to the loss function during optimization [S9].

## Background
The standard 3D Gaussian Splatting optimization loop was introduced by Kerbl et al. in 2023 [S2, unofficial]. It relies on a differentiable rasterizer to render images from 3D Gaussians and minimizes a composite loss of L1 and D-SSIM using the Adam optimizer. Subsequent methods have extended this foundation to address specific challenges such as training speed [TurboGS](15697-turbogs.md), memory efficiency [Sparse SH Order Increment](18689-sparse-sh-order-increment.md), sparse view reconstruction [Active View Selection](20899-active-view-selection-and-residual-learning-for-3dgs.md), and integration with other modalities like diffusion models [SDS Loss](18710-gaussian-splatting-training-with-sds-loss.md) or CAD models [Geometric Initialization](22286-3d-gaussian-splatting-optimization.md).

## Sources
[S1] DiffGS: Functional Gaussian Splatting Diffusion, https://arxiv.org/abs/2410.19657v2, fetched 2026-10-26
[S2] PyGS: Large-scale Scene Representation with Pyramidal 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2405.16829, fetched 2026-10-26 (unofficial)
[S3] Optimization of 3D Gaussian Splatting for Accurate Image Reconstruction, https://doi.org/10.59232/dst-v4i2p101, fetched 2026-10-26
[S4] Practical Evaluation of Standard 3D Gaussian Splatting for Real-World Scene Reconstruction., https://doi.org/10.3390/s26185754, fetched 2026-10-26
[S5] Anttwo/SuGaR, https://github.com/Anttwo/SuGaR, fetched 2026-10-26 (unofficial)
[S6] CADSplat: Sparse-View 3D Gaussian Splatting Aided by CAD Models for Robust, Photorealistic Digital-Twin Reconstruction, https://arxiv.org/abs/2609.18473, fetched 2026-10-30 (unofficial)
[S7] ClunoGS: Compact Language 3D Gaussian Splatting via Unified Optimization, https://arxiv.org/abs/2610.30013, fetched 2026-10-26 (unofficial)
[S8] LIV-GaussMap: LiDAR-Inertial-Visual Fusion for Real-Time 3D Radiance Field Map Rendering, https://doi.org/10.1109/lra.2024.3400149, fetched 2026-10-26
[S9] Reducing the Memory Footprint of 3D Gaussian Splatting, https://doi.org/10.1145/3651282, fetched 2026-10-26
