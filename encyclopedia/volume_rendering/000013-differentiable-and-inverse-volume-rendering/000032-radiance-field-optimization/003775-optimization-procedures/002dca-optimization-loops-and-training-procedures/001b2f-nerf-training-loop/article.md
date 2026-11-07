# NeRF Training Loop

Also called: End-to-End NeRF Training, HashNeRF Training Loop, HoloINR Training Loop, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric difference between rendered images and ground-truth observations by updating the parameters of a neural radiance field [S1][S5, S6, S14, unofficial]. It solves the inverse problem of reconstructing a continuous 3D scene representation from a discrete set of 2D images by treating the scene as a differentiable volume [S5, S6, unofficial]. It sets itself apart from evaluation methods by dynamically sampling rays and computing gradients for backpropagation, and from static configuration methods by orchestrating the data loading, forward pass, and optimizer state updates [S1][S2, unofficial][S8].

## How it works
The training loop minimizes a photometric loss $\mathcal{L}$ between the rendered color $\hat{C}(\mathbf{r})$ and the ground-truth color $C(\mathbf{r})$ for a batch of rays $\mathcal{R}$ [G]. The loop generally follows these steps: ray sampling, forward pass, loss computation, backpropagation, and scheduling [G].

### Inputs and Outputs
**Inputs:**
*   Training dataset: A set of $N$ images with known camera poses (extrinsics) and intrinsics [S2, S5, S14, unofficial].
*   Model parameters $\Theta$: Weights of the neural network (e.g., MLP or hash grid) representing the radiance field [S5, S15, unofficial].
*   Optimizer: Typically Adam or AdamW [S5, S6, S13, S17, unofficial].
*   Hyperparameters: Learning rate, batch size ($N_{rand}$), number of iterations, sampling counts [S5, S11, S17, unofficial].

**Outputs:**
*   Optimized model parameters $\Theta^*$ [S5, S11, unofficial].
*   Checkpoints containing model state, optimizer state, and training step [S11, S15, unofficial].

### Data Structures
*   **Rays Array (`rays_rgb`)**: Pre-computed array of shape $[(N-1) \times H \times W, 3, 3]$ storing ray origins, directions, and target RGB colors [S5, S14, unofficial].
    *   Axis 0: Flattened pixel index.
    *   Axis 1: Channel type (0: ray origin, 1: ray direction, 2: observed RGB).
    *   Axis 2: XYZ/RGB components.
    *   Type: `float32` [S5, S14, unofficial].
*   **Checkpoint**: Dictionary containing:
    *   `model` or `coarse`/`fine`: State dictionary of the neural network [S11, S15, unofficial].
    *   `optimizer`: State dictionary of the optimizer [S11, S15, unofficial].
    *   `step` or `epoch`: Current training step or epoch [S11, S15, unofficial].

### Numbered Steps

1.  **Initialization**:
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the neural network model (e.g., NeRF MLP) and optimizer (Adam) [S5, S11, S17, unofficial].
    *   Initialize learning rate scheduler (e.g., Exponential Decay, Cosine Annealing, or LambdaLR) [S5, unofficial][S8][S11, unofficial].
    *   If resuming, load checkpoint and set scheduler last epoch to `start_epoch - 1` [S9, S15, unofficial].

2.  **Ray Sampling**:
    *   Select a batch of $N_{rand}$ rays from the pre-computed `rays_rgb` array or generate them on-the-fly from random pixel coordinates [S5, S14, unofficial].
    *   If using batching, slice `rays_rgb[i_batch:i_batch+N_{rand}]`, transpose to shape $[3, N_{rand}, 3]$ (origin, direction, target), and increment `i_batch` [S5, S14, unofficial].
    *   Shuffle `rays_rgb` and reset `i_batch` when exhausted to ensure random sampling order [S11, unofficial].
    *   Optionally apply precropping (training on central crop for first $P$ iterations) [S14, unofficial].

3.  **Forward Pass**:
    *   Query the neural network with ray origins, directions, and sample points along the ray to predict density ($\sigma$) and color ($\mathbf{c}$) [S5, S6, unofficial].
    *   Perform volume rendering (alpha compositing) to synthesize the predicted RGB color $\hat{C}$ and depth/accumulation [S5, S6, unofficial].
    *   If using coarse-to-fine sampling, render coarse predictions $\hat{C}_{coarse}$ and fine predictions $\hat{C}_{fine}$ [S11, unofficial].

4.  **Loss Computation**:
    *   Compute the Mean Squared Error (MSE) between predicted RGB $\hat{C}$ and ground truth $C$:
        $$ \mathcal{L}_{photo} = \frac{1}{N_{rand}} \sum_{i=1}^{N_{rand}} (\hat{C}_i - C_i)^2 $$
        [S14, S17, unofficial].
    *   If coarse-to-fine training is used, compute coarse loss $\mathcal{L}_{coarse}$ and fine loss $\mathcal{L}_{fine}$, and sum them: $\mathcal{L} = \mathcal{L}_{coarse} + \mathcal{L}_{fine}$ [S11, unofficial].
    *   Optionally add regularization terms (e.g., sparsity, total variation, eikonal) [S7, unofficial][S8][S16, unofficial].

5.  **Backpropagation**:
    *   Compute gradients of the loss with respect to model parameters: $\nabla_\Theta \mathcal{L}$ [S18, unofficial].
    *   Zero gradients before backward pass [S18, unofficial].
    *   Note: Higher-order derivatives (e.g., for logging gradients) must be computed before `loss.backward()` as the autograd graph is freed [S3, unofficial].

6.  **Update**:
    *   Step the optimizer: $\Theta \leftarrow \Theta - \eta \nabla_\Theta \mathcal{L}$ [S18, unofficial].
    *   Update learning rate according to the scheduler [S5, unofficial][S8][S11, unofficial].
    *   Log metrics (loss, PSNR) and render validation images at specified intervals [S11, S14, unofficial].
    *   Save checkpoints at specified intervals [S11, S15, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_training_loop(model, optimizer, scheduler, dataset, config):
    """
    Standard NeRF training loop.
    """
    # 1. Initialization
    # Pre-compute rays if using batching
    if config.use_batching:
        rays_rgb = precompute_rays(dataset) # Shape: [N_pixels, 3, 3]
        i_batch = 0
        N_rand = config.batch_size
    else:
        rays_rgb = None
        
    start_step = 0
    if config.checkpoint_path and exists(config.checkpoint_path):
        state = load_checkpoint(config.checkpoint_path)
        model.load_state_dict(state['model'])
        optimizer.load_state_dict(state['optimizer'])
        start_step = state['step']
        scheduler.last_epoch = start_step - 1

    # 2. Training Loop
    for step in range(start_step, config.num_iterations):
        # 3. Ray Sampling
        if config.use_batching:
            # Slice batch from pre-computed array
            batch_indices = slice(i_batch, i_batch + N_rand)
            batch_rays = rays_rgb[batch_indices] # Shape: [N_rand, 3, 3]
            # Transpose to [3, N_rand, 3] for model input
            batch_rays = batch_rays.transpose(0, 1) 
            rays_o, rays_d, targets = batch_rays[0], batch_rays[1], batch_rays[2]
            
            i_batch += N_rand
            if i_batch + N_rand >= rays_rgb.shape[0]:
                shuffle(rays_rgb)
                i_batch = 0
        else:
            # Sample random pixels and generate rays on-the-fly
            rays_o, rays_d, targets = sample_rays_from_dataset(dataset, config)

        # 4. Forward Pass
        optimizer.zero_grad()
        
        # Render coarse and fine if applicable
        if config.coarse_to_fine:
            rgb_coarse, extras_coarse = model.render(rays_o, rays_d, coarse=True)
            rgb_fine, extras_fine = model.render(rays_o, rays_d, coarse=False, 
                                                 z_samples=extras_coarse['z_samples'])
        else:
            rgb_fine, extras_fine = model.render(rays_o, rays_d, coarse=False)

        # 5. Loss Computation
        loss = 0.0
        if config.coarse_to_fine:
            loss += mse_loss(rgb_coarse, targets)
        loss += mse_loss(rgb_fine, targets)
        
        # Add regularization if defined
        if config.regularization:
            loss += config.reg_weight * compute_regularization(extras_fine, model)

        # 6. Backpropagation
        loss.backward()
        
        # 7. Update
        optimizer.step()
        scheduler.step()
        
        # Logging and Checkpointing
        if step % config.log_interval == 0:
            log_metrics(step, loss.item())
        if step % config.checkpoint_interval == 0:
            save_checkpoint(step, model, optimizer, config.checkpoint_dir)
            
    return model
```

### Parameters
*   **Learning Rate (`lr`)**: Default $5 \times 10^{-4}$ for Adam [S11, unofficial].
*   **Batch Size (`N_rand`)**: Number of rays sampled per iteration. Typical values: 1024 or 2048 [S5, S11, unofficial].
*   **Iterations (`n_iters`)**: Total number of gradient steps. Default: 100,000 [S11, unofficial].
*   **Positional Encoding Frequencies**: `pos_freq` (default 10) for position, `dir_freq` (default 4) for direction [S11, unofficial].
*   **Learning Rate Scheduler**:
    *   Exponential Decay: `decay_steps = args.lrate_decay * 1000`, `decay_rate = 0.1` [S5, S18, unofficial].
    *   LambdaLR: `lr_lambda(epoch) = gamma ** (epoch / step_size)` [S8][S10, unofficial].
    *   Cosine Annealing [S11, unofficial].

### Edge Cases and Pitfalls
*   **VRAM Overflow**: Mitigated by chunking ray batches during the forward pass (`chunk=args.chunk`) [S5, S14, unofficial].
*   **Instability in Growing Networks**: Direct growth of randomly initialized sub-networks causes density to drop to zero; initialize alpha linear weights of grown sub-networks to match the parent [S1].
*   **Empty Checkpoints**: If a checkpoint is loaded but the step is 0, it is considered empty and training should abort or reinitialize [S15, unofficial].
*   **Gradient Computation Order**: Higher-order derivatives (e.g., for logging) must be computed before `loss.backward()` [S3, unofficial].
*   **Learning Rate Decay**: If using exponential decay, ensure `decay_steps` is calculated correctly (e.g., `args.lrate_decay * 1000`) [S5, S18, unofficial].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
Training Ray Marching and Integration is the differentiable forward pass that synthesizes RGB images from a neural radiance field for gradient-based optimization. It solves the problem of efficiently sampling 3D space along camera rays, querying a neural network for density and color, and integrating these values via alpha compositing to produce a loss-computable image. It sets itself apart from [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md) by computing gradients for backpropagation and from [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) by including the adaptive sampling and occupancy grid pruning steps specific to the training loop.

### [Training Configuration](2332-training-configuration.md)
Training Configuration defines the static and dynamic hyperparameters, optimizer settings, and initialization procedures required to optimize a radiance field model. It establishes baseline settings (MLP depth, width, positional encoding frequencies, and hierarchical sampling counts) against which variants adjust specific architectural or optimization details. It differs from [Training Loop Setup](1522-training-loop-setup.md) by focusing on numerical values and mathematical schedules rather than procedural orchestration, and from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by defining parameters that control sampling density and loss weighting rather than the integration algorithm itself [G].

### [Mip-NeRF 360](3306-mip-nerf-360.md)
Mip-NeRF 360 is a method for synthesizing photorealistic views of unbounded, real-world scenes where the camera may rotate 360 degrees and content exists at arbitrary distances. It solves the problems of aliasing, training instability, and geometric ambiguity inherent in standard NeRF when applied to large-scale environments. It sets itself apart from its parent [NeRF Training Loop](970-nerf-training-loop.md) by integrating a non-linear scene parameterization, online distillation between two MLPs, and a distortion-based regularizer, achieving a 57% reduction in mean squared error compared to standard mip-NeRF.

### [Mip-NeRF 360 Sampling and Training](3393-mip-nerf-360-sampling-and-training.md)
Mip-NeRF 360 Sampling and Training extends the Mip-NeRF framework to unbounded, 360-degree scenes by introducing a disparity-based ray parameterization, a contraction function for coordinate space, and a coarse-to-fine training loop with a proposal network. It solves the artifacts of "floaters" and "background collapse" common in unbounded scene reconstruction by regularizing the density distribution and ensuring efficient sampling through online distillation. It differs from standard NeRF by using linear-in-disparity sampling and from basic Mip-NeRF by handling unbounded scenes via coordinate contraction and disparity parameterization.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [MVSNeRF](6197-mvsnerf.md)
MVSNeRF is a method for fast, generalizable radiance field reconstruction from multi-view stereo (MVS) images. Unlike standard NeRF which requires per-scene optimization, MVSNeRF uses a feedforward neural network to predict the radiance field directly from input images, significantly reducing training time and iteration counts. It sets itself apart by learning a data prior that accelerates convergence, enabling rapid inference without iterative scene-specific optimization.

### [Dynamic and Temporal Modeling](7610-dynamic-and-temporal-modeling.md)
Dynamic and Temporal Modeling extends static Neural Radiance Fields (NeRF) to handle scenes that change over time, enabling the synthesis of novel views from video sequences or non-line-of-sight (NLOS) transient data. It solves the problem of temporal coherence and motion representation by either explicitly modeling deformation fields, implicitly tracking correspondence via color statistics, or modeling light transport in the time domain. It differs from static NeRF by introducing a time dimension or temporal dependencies, and from sibling methods like [Mip-NeRF 360](3306-mip-nerf-360.md) which focuses on spherical panoramas, by specifically addressing the 4D $(x, y, z, t)$ or 5D $(x, y, z, \theta, \phi, t)$ nature of dynamic data.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
This family of methods addresses the problem of optimizing Neural Radiance Fields (NeRF) when input camera poses are unknown, erroneous, or when intrinsic parameters (such as focal length) are not provided. Standard NeRF training assumes precise, pre-computed camera extrinsics and intrinsics; these methods remove that dependency by treating camera parameters as trainable variables within a fully differentiable pipeline, enabling joint optimization by minimizing the reconstruction loss. They set themselves apart from standard NeRF training by introducing mechanisms—such as coarse-to-fine frequency scheduling, local/global geometric alignment, or monocular depth priors—to stabilize the optimization landscape, which is otherwise ill-posed and prone to local minima when poses are free variables.

### [MegaNeRF](8580-meganerf.md)
Mega-NeRF is a framework for training large-scale 3D scenes that support interactive human-in-the-loop fly-throughs, addressing challenges of modeling thousands of images with varying lighting, large model capacities, and fast rendering. It differs from its siblings by using spatial partitioning to enable data parallelism for scalable training, whereas methods like DeRF and KiloNeRF use partitioning to speed up inference of a pre-trained single network. It improves upon [NeRF++](6197-nerf-pp.md) by modeling the foreground with an ellipsoid that tightly encloses camera poses and adding altitude constraints to skip underground sampling.

### [BlockNeRF](8581-blocknerf.md)
BlockNeRF is a method that decomposes a large-scale scene into a set of spatial blocks for separate NeRF training using a 'total-part-total' approach to solve large-scale city building complex rendering problems. It decouples rendering time from scene size to allow rendering to scale to arbitrarily large environments. It sets itself apart from standard NeRF by partitioning the scene geometry and appearance into localized MLPs, addressing the high-frequency information loss and slow inference inherent in single-model approaches for urban-scale data.

### [Specific Model Architectures](8585-specific-model-architectures.md)
Specific Model Architectures refers to the family of modifications to the canonical Neural Radiance Field (NeRF) network topology, input/output representations, and parameterization strategies. While the standard NeRF uses a single Multi-Layer Perceptron (MLP) to map 3D coordinates and viewing directions to density and color, this family encompasses methods that replace, decompose, or augment this core component to address limitations in training speed, rendering latency, memory consumption, generalization, and robustness to sparse or unstructured data. These architectures set themselves apart from sibling optimization procedures (such as [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) or [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)) by fundamentally altering the function $f_\Theta$ that defines the radiance field, rather than just the sampling strategy or loss functions used to train it.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
This family encompasses loss functions and regularization techniques applied during the optimization of Neural Radiance Fields (NeRFs) to mitigate the ill-posed nature of inverse volume rendering, particularly in sparse-view or unbounded settings. While standard NeRF training relies solely on photometric consistency ($\mathcal{L}_{\text{photo}}$), these methods augment the objective with geometric, statistical, or diffusion-based priors to suppress artifacts such as "floaters," overfitting, and view-dependent hallucinations. They set themselves apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) (which focuses on sampling efficiency) and [Training Configuration](2332-training-configuration.md) (which focuses on hyperparameter schedules) by directly modifying the gradient signal to enforce structural plausibility.

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
Progressive Inpainting-and-Erasing (PERF) is a training strategy for single-view NeRF that avoids geometry conflicts between visible and invisible regions by computing a mask of conflicted regions and eliminating them from NeRF training. It sets itself apart from standard single-view NeRF methods by integrating a collaborative RGBD inpainting step with a progressive erasing mechanism to handle the ambiguity of unseen geometry, building on the [NeRF Training Loop](970-nerf-training-loop.md) by dynamically expanding the reference view set.

### [NeRFVS](8667-nerfvs.md)
NeRFVS is a method for indoor scene free view synthesis (FVS) that uses a geometry scaffold from an off-the-shelf neural reconstruction method to guide NeRF optimization, reducing distortions and floaters in extrapolation. It enables NeRF to perform free navigation and extrapolation for indoor scenes by using a neural-reconstructed geometric scaffold to guide optimization, boosted by depth and variance constraints with dynamical adjustment based on view coverage. Unlike standard NeRF, which struggles with extrapolation and sparse views, NeRFVS minimizes per-pixel loss to avoid distortions, unlike SVS which suffers from noticeable artifacts.

### [NeRF-FF](8669-nerf-ff.md)
NeRF-FF is a plug-in strategy that mitigates the degradation of Neural Radiance Field (NeRF) reconstruction quality caused by out-of-focus (defocus blurred) regions in input images. It solves the problem of training NeRFs on partially blurred data by estimating binary image masks that identify in-focus regions, allowing the subsequent NeRF model to omit out-of-focus pixels during training. Unlike methods that use dynamic ray generation or blur kernels, NeRF-FF relies on static ray assumptions, making it compatible with runtime-optimized NeRF variants like Instant-NGP, nerfacto, and splatfacto.

### [KNN-NeRF](8672-knn-nerf.md)
KNN-NeRF is a baseline method for training Neural Radiance Fields (NeRFs) using sparse 3D point clouds as supervision instead of multi-view images. It solves the problem of generating ground-truth color labels for rays that do not pass through existing points by assigning the color of the nearest point in the cloud to the ray. Unlike standard NeRF which relies on pixel colors from images, KNN-NeRF constructs a dense training signal from sparse geometry, though it struggles with sharp object borders due to the low-frequency bias of neural networks and the sparsity of the input.

### [Hard Sample Mining](8673-hard-sample-mining.md)
Hard Sample Mining (HSM) is an optimization procedure for neural radiance fields that improves training efficiency and reconstruction quality by selectively updating the network based on pixel-level error. It solves the problem of uniform computational waste in standard NeRF training, where samples contributing little to the gradient are treated equally to those with high error. Unlike sibling methods such as [Probability-based Ray Sampling with Quadtree Subdivision](7481-probability-based-ray-sampling-with-quadtree-subdivision.md) or [Nerfacc](https://github.com/NVlabs/nerfacc) which prune low-weight samples based on density or transmittance, HSM ranks samples using the propagated pixel loss to subsample a "hard" minibatch that retains most of the update information.

### [Sparse NeRF Reconstruction with Geometric and Semantic Initialization](8779-sparse-nerf-reconstruction-with-geometric-and-semantic-initi.md)
This method addresses the blurred geometry, loss of detail, and slow convergence typical of NeRF training when using sparse input (15–20 images). It distinguishes itself from standard NeRF initialization by decoupling geometric structure anchoring via COLMAP point clouds from semantic-guided weight assignment via Mask R-CNN, thereby reducing initial MSE loss and accelerating convergence.

### [NeRF-ID](8867-nerf-id.md)
NeRF-ID (NeRF in detail) is a differentiable volume rendering optimization method that replaces the heuristic coarse-to-fine sampling procedure of vanilla NeRF with a learnable "proposer" module. It solves the problem of suboptimal sample placement by enabling end-to-end training of the coarse network, proposer, and fine network jointly, thereby improving view synthesis quality. Unlike its sibling [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) which relies on handcrafted probability density functions derived from coarse densities, NeRF-ID uses a neural network to directly predict fine sample locations from coarse features.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Progressive and Dynamic Scene Growth addresses the challenge of reconstructing large-scale or complex 3D scenes from limited input data by incrementally expanding the neural representation's capacity or the training dataset. Unlike static NeRF architectures that require pre-defining the entire hierarchy or fixed network size, these methods adaptively grow the model structure or add synthetic reference views during training to resolve occlusions and capture high-frequency details. This family distinguishes itself from sibling methods like [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) (which focuses on ray selection) and [Sparse NeRF Reconstruction with Geometric and Semantic Initialization](8779-sparse-nerf-reconstruction-with-geometric-and-semantic-initi.md) (which relies on external priors) by actively modifying the model's topology or the set of supervision signals to improve reconstruction quality in complex regions.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Adaptive Ray Sampling refers to a family of strategies in neural radiance field (NeRF) training that dynamically adjust the distribution of sample points along viewing rays to improve reconstruction quality and training efficiency. Unlike uniform sampling, which treats all regions of a ray equally, these methods allocate computational resources to areas of high geometric or photometric complexity, such as object boundaries or high-frequency details. This family addresses the inefficiency of uniform sampling in sparse or complex scenes by clustering samples where they contribute most to the final rendered image.

### [Drone-NeRF](8972-drone-nerf.md)
Drone-NeRF is a method for large-scale 3D scene reconstruction from drone-captured imagery that addresses the specific challenges of overhead drone views, including exposure variations, shadow artifacts from vertical surfaces, and computational efficiency in large scenes. It differs from standard NeRF approaches by employing a block-parallel training strategy where sub-scenes are trained independently and then merged, while explicitly handling exposure discrepancies via appearance embeddings and removing occlusion shadows through an expanded bounding box projection technique. Unlike methods that treat the entire scene as a single volume, Drone-NeRF partitions the space into sub-bounding boxes, optimizing them separately before a final merging phase that ensures color uniformity and depth continuity across block boundaries.

### [CBARF](9066-cbarf.md)
CBARF is a 3D reconstruction model that optimizes imperfect camera poses using a cascaded bundle-adjustment (BA) structure to progressively refine poses and rectify erroneous ones. It builds on [NeRF Training Loop](970-nerf-training-loop.md) by introducing a coarse-to-fine optimization strategy that biases learning towards initial high values, distinguishing it from standard BA methods that may overfit or fail to converge from large initial errors.

### ReconFusion
ReconFusion uses a Diffusion Prior to constrain unobserved views during the training loop [S19, unofficial].

### Supersampling Baseline
Supersampling Baseline is a baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S20]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S20]. The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S20]. It relies on 'oracle' scale information that does not exist for most datasets [S20].

### Two-stage training for proposers
Two-stage training for proposers is a training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S21, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 SGD steps to reduce shock from switching regimes [S21, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S21, unofficial].

### ViP-NeRF
ViP-NeRF introduces visibility priors to supervise density and improve rendering quality under sparse input view conditions (fewer than 50-100 views), preventing artifacts caused by insufficient training data [S22]. It creates a plane scan volume by warping different depth planes of a secondary view image to the main view, compares the warped planes with the main view image to obtain an error map, identifies the minimum error among all planes, and thresholds this error to obtain a visibility prior map corresponding to the primary view image, which is then used to supervise the density during NeRF training [S22]. On the RealEstate-10K dataset, ViP-NeRF achieves a PSNR index 3-4 points higher than the latest paper in 2, 3, and 4 input views [S22].

### Multi-view SDS
Multi-view SDS is a plug-and-play method that creates multi-view supervision during SDS optimization to address the Janus problem caused by one-by-one view sampling [S23]. Rendered images from 4 sampled views are stitched into a large 2x2 image, and gradients are calculated under the condition of the multi-view normal map [S23]. The method converts a 3D asset into differentiable representations including NeRF, 3DGS, and DMTet through rendering supervision, which are then optimized to improve the quality of the 3D asset [S23].

## Operating characteristics
*   **Time Complexity**: $O(N_{iter} \cdot N_{rand} \cdot N_{samples})$, where $N_{iter}$ is the number of iterations, $N_{rand}$ is the batch size (number of rays), and $N_{samples}$ is the number of samples per ray [S5, S14, unofficial].
*   **Memory Complexity**: $O(N_{rand} \cdot N_{samples} \cdot D)$, where $D$ is the dimensionality of the model parameters and intermediate activations [S5, unofficial]. VRAM overflow is a common pitfall, mitigated by chunking [S5, unofficial].
*   **Measured Performance**: Training on 128x128 images for 250 epochs [S4, unofficial]. On a dataset of 199 images, each epoch processes ~8M rays with a batch size of 1024, resulting in ~7,800 iterations per epoch [S17, unofficial].
*   **Scaling**: Scaling to large scenes requires partitioning (e.g., BlockNeRF, MegaNeRF) or hierarchical representations (e.g., HashNeRF) [S1][S2, unofficial][S8].
*   **Strengths**: End-to-end differentiable, handles view-dependent effects, produces high-quality novel views [S5, S6, unofficial].
*   **Limitations**: Slow training convergence, high memory usage, prone to artifacts like "floaters" in sparse views [S1][S17, unofficial].
*   **When to Use**: When high-quality novel view synthesis is required and computational resources are available [S5, unofficial].
*   **When Not to Use**: When real-time training or inference is required without acceleration techniques [S15, unofficial].

## Background
The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, unofficial]. It established the standard approach of optimizing a coordinate-based MLP to represent a 3D scene by minimizing the photometric loss between rendered and ground-truth images. The method has since been extended to various domains, including large-scale scenes (MegaNeRF, BlockNeRF), dynamic scenes (Dynamic NeRF), and generalizable reconstruction (MVSNeRF) [S1][S2, unofficial][S8].

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
[S19] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S20] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S21] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
[S22] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S23] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
