# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric error between rendered images and ground-truth observations by updating the parameters of a neural radiance field [S5, S6, S14, unofficial]. It solves the inverse problem of reconstructing a continuous 3D scene representation from a discrete set of 2D images [S5, unofficial]. What sets it apart from its siblings, such as [Stage-based Differentiable Volume/Radiance Field Training](4241-stage-based-differentiable-volume-radiance-field-training.md) which decouples geometry and appearance, or [Photometric Optimization](5790-photometric-optimization.md) which focuses on specific loss formulations, is its foundational reliance on stochastic ray sampling and standard MSE minimization via Adam optimization to jointly learn density and color fields [S5, S14, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of $N$ training images with known camera poses (extrinsics/intrinsics) [S5, S11, unofficial]. Ground-truth RGB values $\mathbf{C}_{gt}$ [S5, unofficial].
*   **Outputs:** Optimized neural network weights $\Theta$ representing the radiance field $F_\Theta$ [S5, unofficial].

**Data Structures:**
*   **Rays Buffer (`rays_rgb`):** A pre-computed array of shape $[(N-1) \times H \times W, 3, 3]$ containing all possible training rays [S5, S14, unofficial].
    *   Axis 0: Flattened pixel index.
    *   Axis 1: Channel type (0: Ray Origin $\mathbf{o}$, 1: Ray Direction $\mathbf{d}$, 2: Target RGB $\mathbf{C}_{gt}$).
    *   Axis 2: XYZ/RGB components.
    *   Data Type: `float32` [S5, S14, unofficial].
*   **Checkpoint:** A dictionary containing:
    *   `'model'` or `'coarse'`/`'fine'`: State dictionaries of the neural networks [S10, S11, unofficial].
    *   `'optimizer'`: State dictionary of the optimizer [S10, S11, unofficial].
    *   `'step'` or `'epoch'`: Current training iteration count [S10, S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the NeRF model(s) (e.g., coarse and fine MLPs) [S11, unofficial].
    *   Initialize the optimizer (typically Adam) with learning rate $\eta$ [S5, S11, unofficial].
    *   Initialize the learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   Precompute the `rays_rgb` buffer by generating rays for all pixels in all training images [S5, S11, unofficial].

2.  **Iteration Loop:** For iteration $i$ from 1 to $N_{iters}$:
    *   **Ray Sampling:**
        *   If batching is enabled, select a batch of $N_{rand}$ rays from `rays_rgb` [S5, S14, unofficial].
        *   Extract ray origins $\mathbf{o}$, directions $\mathbf{d}$, and target colors $\mathbf{C}_{gt}$ for the batch [S5, S14, unofficial].
        *   If `precrop_iters` > $i$, restrict sampling to a central crop of the image [S2, S14, unofficial].
    *   **Forward Pass (Rendering):**
        *   Sample points $\mathbf{x}$ along the rays $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ within near/far bounds [S5, S13, unofficial].
        *   Query the NeRF model $F_\Theta(\mathbf{x}, \mathbf{d})$ to obtain volume density $\sigma$ and color $\mathbf{c}$ [S5, S6, unofficial].
        *   Compute the rendered color $\hat{\mathbf{C}}(\mathbf{r})$ using the differentiable volume rendering equation (see Parent article) [S5, S6, unofficial].
        *   If using a coarse-fine scheme, render coarse predictions $\hat{\mathbf{C}}_{coarse}$ and fine predictions $\hat{\mathbf{C}}_{fine}$ [S11, unofficial].
    *   **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) loss between rendered and ground-truth colors [S5, S14, unofficial]:
            $$
            \mathcal{L}_{rgb} = \frac{1}{B} \sum_{j=1}^{B} \| \hat{\mathbf{C}}_j - \mathbf{C}_{gt,j} \|^2
            $$
            where $B$ is the batch size [S14, S17, unofficial].
        *   If coarse output $\hat{\mathbf{C}}_{coarse}$ is available, add its MSE loss to the total loss [S6, S11, unofficial]:
            $$
            \mathcal{L}_{total} = \mathcal{L}_{coarse} + \mathcal{L}_{fine}
            $$
        *   [G] Add any additional regularization terms (e.g., Eikonal, sparsity) if defined in the specific configuration [S8][S16, unofficial].
    *   **Backpropagation and Update:**
        *   Zero gradients: `optimizer.zero_grad()` [S18, unofficial].
        *   Compute gradients: `loss.backward()` [S18, unofficial].
        *   Update weights: `optimizer.step()` [S18, unofficial].
        *   Update learning rate according to the scheduler [S5, S11, unofficial].
    *   **Logging and Checkpointing:**
        *   Log loss and PSNR every $I_{print}$ steps [S2, S14, unofficial].
        *   Render validation images every $I_{img}$ steps [S2, S11, unofficial].
        *   Save checkpoint every $I_{weights}$ steps [S2, S11, unofficial].

**Parameters with Typical Values:**
*   $N_{rand}$ (Batch Size): 1024 or 2048 rays per iteration [S5, S11, S14, unofficial].
*   Learning Rate ($\eta$): $5 \times 10^{-4}$ for Adam [S11, unofficial].
*   $N_{iters}$ (Total Iterations): 100,000 to 1,000,000 [S11, S14, unofficial].
*   Decay Rate: 0.1 [S6, S18, unofficial].
*   Decay Steps: $args.lrate\_decay \times 1000$ [S5, S18, unofficial].
*   Chunk Size: Limits rays processed per forward pass to manage VRAM (e.g., 8192) [S5, S14, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering all rays at once exceeds GPU memory. Mitigated by chunking rays during the forward pass [S5, S14, unofficial].
*   **Gradient Graph Freeing:** Higher-order derivatives (e.g., for logging gradients) must be computed *before* `loss.backward()` to avoid freeing the autograd graph [S3, unofficial].
*   **Empty Checkpoints:** Loading a checkpoint with step 0 is an error condition [S15, unofficial].
*   **Stability in Growth:** When growing sub-networks, initialize alpha linear weights to match the parent network to prevent density collapse [S1].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images from a neural radiance field during optimization. It uses stochastic perturbations and adaptive sampling, unlike evaluation which is deterministic.

### [Training Configuration](2332-training-configuration.md)
Defines hyperparameters, optimization routines, and architectural constraints (e.g., learning rate schedules, loss weights) critical for convergence in methods like mip-NeRF and Instant-NGP.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs the scene graph, initializing the density grid and spherical harmonics coefficients, binding neural network parameters to the integrator components.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
Jointly optimizes NeRF weights and camera poses/intrinsics from unposed images, treating poses as learnable variables updated via gradient descent.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Adds mathematical terms (e.g., Eikonal, sparsity) to the loss to constrain the solution space, mitigate floaters, and improve geometric accuracy.

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
A single-view NeRF strategy that computes a mask of conflicted regions and eliminates them from training to avoid geometry conflicts.

### [KNN-NeRF](8672-knn-nerf.md)
A baseline using sparse 3D point clouds as supervision, assigning nearest-point colors to rays lacking ground-truth labels.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Incrementally expands the neural representation's capacity or training dataset to resolve occlusions and capture high-frequency details in large scenes.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Replaces uniform sampling with strategies that concentrate resources on regions of high geometric or photometric complexity.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
Decouples initialization from random assumptions, leveraging external geometric scaffolds or semantic masks for sparse-view reconstruction.

### [NerfingMVS](9518-nerfingmvs.md)
Optimizes NeRF over implicit volumes guided by learning-based depth priors from SfM+MVS to eliminate shape-radiance ambiguity.

### [NeRV Training](10253-nerv-training.md)
Optimizes a representation decoupling geometry, appearance, and visibility, using a separate visibility MLP supervised by consistency terms.

### [Aug-NeRF](10294-aug-nerf.md)
Injects adversarial perturbations into coordinates, features, and outputs to implicitly regularize the model for better generalization.

### [Cross-Guided Optimization of Radiance Fields (CROP)](10371-cross-guided-optimization-of-radiance-fields-crop.md)
Performs multi-view image super-resolution during optimization to improve high-resolution novel view synthesis.

### [Application-Specific Pipelines](11200-application-specific-pipelines.md)
Specialized pipelines for tasks like monocular pose estimation or text-guided editing, decoupling standard optimization objectives.

### [NeRF-GT Baseline Training Setup](11349-nerf-gt-baseline-training-setup.md)
Optimizes NeRF against High Dynamic Range (HDR) ground truth to synthesize HDR imagery.

### [Scene Representation and Architecture](11368-scene-representation-and-architecture.md)
Defines the neural network structures (MLP/Grid) approximating the radiance field, focusing on function approximation.

### [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)
Reduces computational cost by restricting evaluation to sparse structures (visual hulls) or using lightweight caches.

### [Deceptive-NeRF/3DGS Pipeline](11860-deceptive-nerf-3dgs-pipeline.md)
Enhances reconstruction by densifying input observations with synthetic pseudo-observations generated by a diffusion model.

### [MELON](11909-melon.md)
Simultaneously reconstructs NeRF and estimates poses using an encoder and a "modulo loss" to handle rotational symmetries.

### [Depth-Guided NeRF Training via Earth Mover’s Distance](11911-depth-guided-nerf-training-via-earth-mover-s-distance.md)
Supervises ray termination distributions using EMD and monocular depth priors to improve geometric understanding.

### [Urban Radiance Fields](11973-urban-radiance-fields.md)
Incorporates depth, sight, and sky segmentation losses with LiDAR data for large-scale urban reconstruction.

### [NeRFReN](11981-nerfren.md)
Separates radiance fields into transmitted and reflected components to accurately represent reflective surfaces like glass.

### [DFF (Decomposing NeRF for Editing Via Feature Field Distillation)](12068-dff-decomposing-nerf-for-editing-via-feature-field-distillat.md)
Distills features from a 2D teacher network into the NeRF for semantic segmentation and editing.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S21]. This baseline relies on 'oracle' scale information that does not exist for most datasets and performs worse than mip-NeRF while being significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 SGD steps to reduce shock from switching regimes [S22, unofficial]. While training from scratch sometimes works, two-stage training is consistently effective [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S23]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S23].

### Incremental Registration
A training strategy that registers frames incrementally following temporal order, leveraging the intuition that camera motions in videos are smooth and adjacent poses are close [S24]. At the start, it jointly optimizes NeRF parameters $\Theta$ and poses $\{P_1, P_2\}$ from the first two frames. After every $k$ iterations, it adds a new frame $I_i$ and initializes its pose $P_i$ by the previous pose $P_{i-1}$. It freezes the learning rate on poses and NeRF until all frames are registered, applying a decay schedule afterward [S24]. Typical values: $k=1k$ iterations for CO3D/HO3D, 500 for LLFF; freeze duration 100k for HO3D/CO3D, 30k for LLFF [S24].

### RapNeRF
A NeRF model tailored for view extrapolation, contrasting with standard NeRFs that excel at interpolation [S25]. Training begins with a baseline NeRF to recover a rough mesh. A Ray Atlas (RA) is created by extracting a rough 3D mesh from a pretrained NeRF and mapping training ray directions onto its vertices. Random Ray Casting (RRC) is employed: for a training ray hitting a surface point $v = o + tz d$, a backward ray is cast from $v$ toward a new origin $o'$ with uniformly sampled angular perturbations. RRC and RA augment training rays with a predetermined probability [S25].

### NELF
A neural volumetric rendering framework modeling scene appearance using light transport vectors, enabling realistic relighting and view synthesis of human portraits from only five input images [S25]. A UNet-style CNN extracts per-view features. An MLP regresses volume density and transport vectors. Environment maps are estimated to disentangle lighting. It is trained on synthetic data and adapted to real images via a domain adaptation module [S25].

### Depth Completion Network NeRF
Extracts a sparse point cloud using COLMAP. Introduces a depth loss based on predicted depth and uncertainty in addition to the standard volumetric loss. Trains on RGB-D data from ScanNet and Matterport3D with Gaussian noise added to the depth [S25].

### JaxNeRF
A re-implementation of the original NeRF in JAX, used as a benchmark comparison for early works seeking to improve on training and rendering speed [S25]. It is slightly faster and more suited for distributed computing than the original TensorFlow implementation of NeRF [S25].

### SNeS
A method improving geometry by learning probable symmetries for partly symmetric and partly hidden in-scene objects through soft symmetry constraints on geometry and material properties [S25].

### NeRF with Per-Frame Latent Codes
A method using per-frame learned latent codes and adding facial expression as a 76-dimensional coefficient of a morphable model constructed from Face2Face [S25].

### Online Incremental NeRF Optimization
Plans for the next best views, perceives new observations based on the current scene representation, and adds new observations into the training set incrementally to optimize the representation [S26, unofficial]. It differs from offline methods like DVGO by updating the representation online as new views are collected, rather than optimizing based on a fixed, pre-collected set of images [S26, unofficial].

### DreamFusion
A text-to-3D generation method that optimizes a NeRF to render images with high scores from a text-to-image diffusion model, requiring no 3D training data [S27]. Introduced in 2022 [S27].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot N_{samples} \cdot C_{MLP})$, where $N_{iters}$ is total iterations, $N_{rand}$ is batch size, $N_{samples}$ is samples per ray, and $C_{MLP}$ is the cost of the neural network forward pass [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot N_{samples} \cdot C_{MLP} + VRAM_{chunk})$. Memory is dominated by the activation storage for the batch during backpropagation and the pre-computed rays buffer [S5, S14, unofficial].
*   **Measured Performance:** In the original implementation, training takes approximately 5 hours on a single GPU for 100k iterations on synthetic scenes [S5, unofficial].
*   **Scaling:** Performance scales with batch size up to GPU memory limits; larger batches improve throughput but require chunking or gradient accumulation [S15, unofficial].
*   **Strengths:** Simple to implement; robust convergence with standard optimizers; produces high-quality novel views for dense input sets [S5, S14, unofficial].
*   **Limitations:** Slow convergence for high-frequency details; sensitive to learning rate schedules; requires significant VRAM for large scenes or high resolution [S5, S17, unofficial].
*   **When to Use:** Standard novel view synthesis with dense multi-view input; baseline for comparing new architectural or sampling techniques [S5, S14, unofficial].
*   **When Not to Use:** Sparse-view settings (requires regularization/initialization variants); real-time applications (requires acceleration variants); scenes with unknown poses (requires pose optimization variants) [S1][S2, unofficial][S24].

## Background
The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, unofficial]. It established the standard protocol for optimizing coordinate-based neural networks for 3D scene representation. The method builds on the differentiable volume rendering equation, using stochastic gradient descent with Adam to minimize the pixel-wise MSE loss [S5, unofficial]. It has become the foundational training procedure for subsequent NeRF variants, including mip-NeRF, Instant-NGP, and many others [S11, S15, unofficial].

## Sources
[S1] Recursive-NeRF: An Efficient and Dynamically Growing NeRF, https://doi.org/10.1109/tvcg.2022.3204608, fetched 2026-10-26
[S2] GreatDrake/reparameterized-volume-sampling, https://github.com/GreatDrake/reparameterized-volume-sampling, fetched 2026-10-26 (unofficial)
[S3] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S4] omkarchittar/Neural_Radiance_Fields, https://github.com/omkarchittar/Neural_Radiance_Fields, fetched 2026-10-26 (unofficial)
[S5] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S6] yenchenlin/nerf-pytorch, https://github.com/yenchenlin/nerf-pytorch, fetched 2026-10-26 (unofficial)
[S7] yashbhalgat/HashNeRF-pytorch, https://github.com/yashbhalgat/HashNeRF-pytorch, fetched 2026-10-26 (unofficial)
[S8] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S9] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S10] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S11] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S12] 3D volumetric rendering with NeRF - Keras, https://keras.io/examples/vision/nerf/, fetched 2026-10-26
[S13] suvadityamuk/NeRF-Experiments, https://github.com/suvadityamuk/NeRF-Experiments, fetched 2026-10-27 (unofficial)
[S14] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S15] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
[S16] DESY-FS-PETRA/HoloINR, https://github.com/DESY-FS-PETRA/HoloINR, fetched 2026-10-27 (unofficial)
[S17] 7.6. Neural Radiance Fields for Drones, https://www.roboticsbook.org/S76_drone_learning.html, fetched 2026-10-29 (unofficial)
[S18] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
[S19] TöRF: Time-of-Flight Radiance Fields for Dynamic Scene View Synthesis, https://doi.org/10.48550/arxiv.2109.15271, fetched 2026-10-29 (unofficial)
[S20] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S21] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S22] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
[S23] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S24] ICON: Incremental CONfidence for Joint Pose and Radiance Field Optimization, https://doi.org/10.1109/cvpr52733.2024.00517, fetched 2026-10-26
[S25] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S26] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
[S27] Differentiable Rendering - 可微渲染：从2D 学习3D, https://www.ultralytics.com/zh/glossary/differentiable-rendering, fetched 2026-10-30
