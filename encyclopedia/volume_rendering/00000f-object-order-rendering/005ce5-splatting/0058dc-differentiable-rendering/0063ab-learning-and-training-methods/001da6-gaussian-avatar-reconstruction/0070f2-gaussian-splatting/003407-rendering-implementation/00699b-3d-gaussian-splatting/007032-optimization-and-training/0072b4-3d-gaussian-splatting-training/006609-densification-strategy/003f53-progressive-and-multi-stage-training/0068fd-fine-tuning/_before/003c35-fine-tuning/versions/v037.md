# Fine-tuning
Also called: 3DGS Fine-tuning, Gaussian Splatting Fine-tuning

## Overview
Fine-tuning in 3D Gaussian Splatting (3DGS) refers to the optimization phase that refines a pre-initialized set of 3D Gaussian primitives to achieve high-fidelity reconstruction or specific appearance properties [S1, S3]. It solves the problem of initializing the radiance field by leveraging pretrained priors or coarse estimates to accelerate convergence and reduce memory consumption compared to training from scratch [S1]. It sets itself apart from standard training by often decoupling geometry and appearance optimization, enabling dynamic densification (growing/pruning) during the refinement phase, and employing specialized loss functions for tasks like inverse rendering or stylization [S1, S2, S3].

## How it works
The fine-tuning process generally begins with a set of initialized 3D Gaussian primitives, which may be generated via a feed-forward network or extracted from Structure-from-Motion (SfM) [S1]. The optimization then proceeds by minimizing a task-specific loss function while updating the geometric and color attributes of the Gaussians [S1, S2].

### Inputs and Outputs
*   **Inputs:** Initial set of 3D Gaussians $\mathcal{G} = \{g_i\}$, where each $g_i$ has position $\mathbf{p}_i$, scale $\mathbf{s}_i$, rotation $\mathbf{q}_i$, opacity $\alpha_i$, and spherical harmonic (SH) coefficients $\mathbf{c}_i$ [S1, S2]. Input images $\{I_k\}$ and corresponding camera poses [S1, S2].
*   **Outputs:** Optimized set of 3D Gaussians $\mathcal{G}'$ with refined attributes [S1, S2].

### Data Structures
The core data structure is the list of 3D Gaussians. Each Gaussian $g_i$ is defined by:
*   $\mathbf{p}_i \in \mathbb{R}^3$: Mean position.
*   $\mathbf{s}_i \in \mathbb{R}^3$: Scale (often represented as log-scale for optimization stability) [S2].
*   $\mathbf{q}_i \in \mathbb{R}^4$: Rotation (unit quaternion) [S2].
*   $\alpha_i \in [0, 1]$: Opacity [S1].
*   $\mathbf{c}_i$: SH coefficients for view-dependent color [S1].
*   In inverse rendering variants, additional attributes for material $\mathbf{m}_i$ and lighting $\mathbf{e}_i$ are included [S2].

### Steps
1.  **Initialization:** Generate initial 3D Gaussian primitives. This can be done via a direct feed-forward process where geometry and color are predicted simultaneously, or by using standard SfM points [S1].
2.  **Loss Computation:** Render the scene using differentiable rasterization to obtain predicted images $\hat{I}$. Compute the loss $\mathcal{L}$ between $\hat{I}$ and ground truth $I$ [S1, S2].
    *   For standard fine-tuning, $\mathcal{L}$ typically includes L1 color loss and SSIM loss [S1].
    *   For inverse rendering, $\mathcal{L}$ includes color reconstruction, normal smoothing, and material/lighting decomposition losses [S2].
    *   For stylization, $\mathcal{L}$ includes nearest neighbor feature match loss and depth preservation loss [S3].
3.  **Gradient Update:** Update all trainable parameters (geometry, color, and optionally material/lighting) using an optimizer (typically Adam) [S1, S2].
    *   Learning rate: $1 \times 10^{-3}$ is common [S1].
4.  **Densification/Pruning (Optional):** During fine-tuning, the method may enable growing (splitting/cloning) and pruning of primitives to adapt to scene complexity and remove redundant initial estimates [S1]. This is distinct from the initial densification phase in vanilla 3DGS, as it occurs after a coarse initialization [S1].
5.  **Stage Decoupling (Specific Variants):** In multi-stage fine-tuning (e.g., GS-IR), the optimization is split:
    *   **Stage 1:** Optimize geometry $\mathbf{T} = \{\mathbf{p}, \mathbf{s}, \mathbf{q}, \mathbf{n}\}$ and appearance $\mathbf{A}$ by minimizing $\mathcal{L}_{init} = \mathcal{L}_c + \mathcal{L}_n$ for 30,000 iterations [S2].
    *   **Stage 2:** Fix $\mathbf{T}$ and $\mathbf{A}$, then optimize material $\mathbf{M}$ by minimizing $\mathcal{L}_d$ for 10,000 iterations [S2].

### Formulas
*   **Standard Loss:** $\mathcal{L} = \lambda_{L1}\mathcal{L}_{L1} + \lambda_{SSIM}\mathcal{L}_{SSIM} + \lambda_{entropy}\mathcal{L}_{entropy}$ [S1].
*   **Inverse Rendering Initial Loss:** $\mathcal{L}_{init} = \mathcal{L}_c + \mathcal{L}_n$ [S2].
    *   $\mathcal{L}_c$: L1 color reconstruction loss.
    *   $\mathcal{L}_n = \mathcal{L}_{n-p} + \lambda_{n-TV} TV_{normal}$, where $TV_{normal}$ is the total variation of the predicted normal map [S2].
*   **Inverse Rendering Decomposition Loss:** $\mathcal{L}_d = \mathcal{L}_{shade} + \lambda_M TV_{mat} + \lambda_E TV_{light}$ [S2].
    *   $\mathcal{L}_{shade}$: Difference between rendered shaded image and ground truth.
    *   $TV_{mat}$: Total variation of the material map.
    *   $TV_{light}$: Total variation of the lighting map [S2].
*   **Stylization Loss:** Combines nearest neighbor feature match loss and depth preservation loss [S3].

### Parameters
*   **Optimizer:** Adam [S1, S2].
*   **Learning Rate:** $1 \times 10^{-3}$ [S1].
*   **Iterations:** Varies by variant. Standard fine-tuning may run for tens of thousands of iterations [S1]. GS-IR uses 30,000 iterations for Stage 1 and 10,000 for Stage 2 [S2].
*   **Smoothing Weights (GS-IR):** $\lambda_{n-TV} = 5.0$, $\lambda_M = 1.0$, $\lambda_E = 0.01$ [S2].

### Edge Cases and Pitfalls
*   **Redundancy:** Initial feed-forward Gaussians may be redundant, requiring pruning to maintain high fidelity while reducing memory consumption [S1].
*   **Overfitting:** Without proper regularization (e.g., entropy loss or TV smoothing), fine-tuning may overfit to input views, especially in sparse novel view synthesis scenarios [S1, S3].
*   **Blurry Inputs:** In defocus-blurry environments, SfM extraction is poor, leading to sparse point clouds. Supplementary point addition or deblurring techniques may be required before fine-tuning [S1, S3].

## Subtopics

### [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md)
A densification control mechanism that uses an exponentially ascending gradient threshold to regulate splitting and cloning during fine-tuning. It allows aggressive densification when few Gaussians exist and restricts it as the scene becomes dense, balancing convergence speed and overfitting.

### [SplatFormer Training Objective & Refinement](11745-splatformer-training-objective-refinement.md)
A feed-forward neural module that refines initial 3DGS representations to enable robust novel view synthesis from out-of-distribution (OOD) camera angles. It predicts attribute residuals to correct bias from standard optimization, ensuring geometric consistency unlike 2D image restoration methods.

### [MCMC-based 3DGS / SGLD for 3DGS](11917-mcmc-based-3dgs-sgld-for-3dgs.md)
Reformulates 3DGS as Markov Chain Monte Carlo (MCMC) sampling, converting updates into Stochastic Gradient Langevin Dynamics (SGLD). It introduces noise to promote exploration, removing the need for manual densification heuristics like cloning and splitting during fine-tuning.

### [Deblurring 3D Gaussian Splatting](12318-deblurring-3d-gaussian-splatting.md)
A real-time framework that reconstructs sharp scenes from blurry inputs by manipulating Gaussian covariance and means using an MLP during fine-tuning. It handles defocus and motion blur, achieving >800 FPS inference, and supplements sparse point clouds via KNN interpolation.

### [Supplementary Point Addition for Sparse Point Clouds](12323-supplementary-point-addition-for-sparse-point-clouds.md)
A densification technique that adds synthetic points to compensate for sparsity in far-end regions of defocus-blurry scenes where SfM fails. It ensures adequate coverage in large-depth-of-field scenarios during the fine-tuning phase.

### [Augmented Radiance Field](13218-augmented-radiance-field.md)
A post-enhancement framework that adds Gaussian kernels with view-dependent opacity to model specular components explicitly. It decouples diffuse and specular radiance, avoiding the memory and stability issues of high-order SH coefficients in standard 3DGS.

### [Frequency Regularization for Densification](13484-frequency-regularization-for-densification.md)
(FreGS) Addresses over-reconstruction by minimizing the discrepancy between the frequency spectrum of the rendered image and the ground truth in Fourier space. It guides coarse-to-fine densification via progressive frequency annealing, increasing pixel gradients in over-reconstructed regions.

### [Error-Guided Redistribution](14037-error-guided-redistribution.md)
Refines Gaussian allocation based on reconstruction quality metrics (positional gradients or SSIM) rather than averaged gradient magnitudes. It actively moves primitives to high-error regions or identifies defective Gaussians for targeted densification.

### CoherentGS
A regularized optimization approach for sparse novel view synthesis that constrains Gaussian movement in 2D image space using monocular depth estimates [S1]. It prevents "needle-like" artifacts and overfitting on extremely sparse inputs (3–4 images) by enforcing single-view and multi-view constraints [S1].

### [Pruning and Sparsity Control](14431-pruning-and-sparsity-control.md)
Methods that reduce the number of active Gaussians to lower memory consumption while maintaining quality. They actively eliminate redundant primitives throughout optimization using geometric or volume-based criteria, distinguishing them from methods relying solely on gradient magnitude.

### [Rendering-Free Pruning via Hessian Importance](14447-rendering-free-pruning-via-hessian-importance.md)
(REFINE) Identifies and removes unimportant primitives without forward rendering passes by using an analytically approximated Hessian field. It estimates image-space quality degradation based on parameter-space gradients, achieving significant speedups in pruning.

### [GoF-based Dynamic 3DGS Video Construction](14770-gof-based-dynamic-3dgs-video-construction.md)
Constructs dynamic 3DGS video by segmenting input sequences into Groups of Frames (GoFs) and training independent deformation fields within each segment. It prevents motion drift and geometric inconsistencies from long-range temporal dependencies by decoupling static and dynamic components.

### [TIP-Editor](14859-tip-editor.md)
A method for editing 3DGS scenes using text and image prompts, combining stepwise 2D personalization with score distillation sampling (SDS). It uses LoRA for reference image alignment and a dedicated localization loss for scene adaptation, enabling high-fidelity local modifications.

### Two-stage Optimization
A specific fine-tuning strategy (e.g., in GS-IR) that decouples geometry/appearance optimization from material decomposition [S2]. Stage 1 optimizes geometry and appearance for 30,000 iterations using color and normal losses; Stage 2 fixes these and optimizes material for 10,000 iterations using decomposition loss [S2].

### StylizedGS
A controllable 3D stylization method that performs color matching with a style image and uses a 3DGS filter to remove artifacts [S3]. It employs nearest neighbor feature match loss and depth preservation loss for optimization, enabling artistic appearance editing [S3].

## Operating characteristics
*   **Time Complexity:** Fine-tuning is generally faster than training from scratch due to good initialization [S1]. Multi-stage methods (e.g., GS-IR) add overhead but improve convergence for specific tasks [S2].
*   **Memory Complexity:** Depends on the number of Gaussians. Pruning and feed-forward initialization can significantly reduce memory consumption compared to dense SfM-based initialization [S1].
*   **Performance:** Achieves photorealism on par with or surpassing other methods when pretrained priors are used [S1]. Real-time deblurring variants achieve >800 FPS [S1].
*   **Strengths:** Accelerates convergence, reduces memory usage, handles specific tasks (inverse rendering, stylization, deblurring) via specialized losses [S1, S2, S3].
*   **Limitations:** Requires careful tuning of hyperparameters (learning rate, smoothing weights) [S1, S2]. May overfit to input views without regularization [S1, S3]. Performance degrades with extremely sparse inputs without additional constraints (e.g., CoherentGS) [S1].
*   **When to Use:** When high-fidelity reconstruction is needed quickly, when specific appearance properties (material, style, sharpness) must be refined, or when dealing with sparse/blurry inputs [S1, S2, S3].
*   **When Not to Use:** When computational resources are extremely limited and coarse results are acceptable, or when the scene has no distinct features for SfM/initialization [S1].

## Background
Fine-tuning in 3D Gaussian Splatting emerged as a refinement of the original 3DGS method to address limitations in initialization, convergence speed, and specific rendering tasks. The original 3DGS method relied on dense SfM points and extensive densification [S1]. Subsequent works like EVolSplat demonstrated that feed-forward initialization followed by fine-tuning could reduce memory usage and accelerate convergence [S1]. Inverse rendering approaches like GS-IR introduced multi-stage fine-tuning to decouple geometry, appearance, and material optimization [S2]. Stylization and editing methods like StylizedGS and TIP-Editor adapted the fine-tuning process to incorporate style losses and prompt-based guidance [S3]. The method is widely used in urban view synthesis, inverse rendering, and 3D editing applications [S1, S2, S3].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S2] GS-IR: 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.1109/cvpr52733.2024.02045, fetched 2026-10-26
[S3] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
