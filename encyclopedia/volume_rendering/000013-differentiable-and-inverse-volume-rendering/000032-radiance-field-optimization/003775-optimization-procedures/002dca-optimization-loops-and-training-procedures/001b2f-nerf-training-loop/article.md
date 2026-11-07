# NeRF Training Loop

Also called: End-to-End NeRF Training, HashNeRF Training Loop, HoloINR Training Loop, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric difference between rendered images and ground-truth training views by updating the parameters of a neural radiance field [S5, S14, unofficial]. It solves the inverse problem of reconstructing a continuous 3D scene representation from discrete 2D images by sampling rays, querying the network, and backpropagating gradients [S8]. It sets itself apart from [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) by including adaptive sampling, occupancy grid pruning, and gradient-based weight updates specific to the training phase [S1, S8].

## How it works
The training loop optimizes a coordinate MLP (or other radiance field representation) to minimize the Mean Squared Error (MSE) between predicted and observed RGB colors [S3, S14, unofficial].

### Inputs and Outputs
*   **Inputs:** A dataset of $N$ training images with known camera poses (extrinsics $C2W$ and intrinsics $K$) [S2, S11, unofficial].
*   **Outputs:** Optimized network weights $\theta$ (and auxiliary parameters like occupancy grids or appearance embeddings) [S8][S15, unofficial].

### Data Structures
*   **Rays:** A batch of rays is represented as a tensor of shape $[B, 3, 3]$ or $[B, 6]$ depending on implementation, where $B$ is the batch size ($N_{rand}$) [S5, S14, unofficial]. The channels typically correspond to ray origin $(x,y,z)$, direction $(x,y,z)$, and optionally near/far bounds [S5, S14, unofficial].
*   **Pre-computed Rays:** To accelerate data loading, rays are often pre-computed into a flat array `rays_rgb` of shape $[(N-1) \times H \times W, 3, 3]$ containing origins, directions, and target RGB colors, which is shuffled and sliced per iteration [S5, S14, unofficial].
*   **Checkpoint:** A dictionary storing `model` (state dict), `optimizer` (state dict), and `epoch` or `step` index [S10, S11, unofficial].

### Numbered Steps

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the neural network model (e.g., NeRF MLP) and optimizer (typically Adam) [S5, S11, unofficial].
    *   Initialize learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   If using occupancy grids (e.g., Instant-NGP), initialize the grid with zeros or specific resolution [S15, unofficial].
    *   [G] Initialize random seed for reproducibility.

2.  **Ray Sampling:**
    *   Select a batch of $N_{rand}$ random pixels from the training images [S4, S14, unofficial].
    *   Construct ray bundles (origin, direction) for these pixels using camera intrinsics and extrinsics [S4, S14, unofficial].
    *   [G] If using NDC (Normalized Device Coordinates) for forward-facing scenes, transform ray origins and directions [S6, unofficial].
    *   Sample $K$ points along each ray between near and far bounds. Bounds are dataset-specific (e.g., Blender: near=2.0, far=6.0; LLFF: computed from bounding depths) [S2, S13, unofficial].
    *   Apply stratified sampling or jittering to perturb sample positions along the ray [S5, unofficial].

3.  **Forward Pass (Differentiable Volume Rendering):**
    *   Query the neural network at each sample point to obtain density ($\sigma$) and color ($\mathbf{c}$) [S5, S6, unofficial].
    *   Compute transmittance $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ and weights $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$ [S5, unofficial].
    *   Composite the final color: $\hat{C}(\mathbf{r}) = \sum_{i=1}^{K} w_i \mathbf{c}_i$ [S5, unofficial].
    *   If using a coarse-to-fine scheme (two networks), render a coarse image, sample additional points based on coarse weights, and render a fine image [S5, S11, unofficial].

4.  **Loss Computation:**
    *   Compute the MSE loss between the rendered color $\hat{C}(\mathbf{r})$ and the ground-truth color $C(\mathbf{r})$:
        $$ \mathcal{L}_{\text{MSE}} = \frac{1}{B} \sum_{i=1}^{B} (\hat{C}_i - C_i)^2 $$
        [S14, S17, unofficial]
    *   If a coarse network is used, add the coarse loss: $\mathcal{L} = \mathcal{L}_{\text{fine}} + \mathcal{L}_{\text{coarse}}$ [S11, S18, unofficial].
    *   [G] Optionally add regularization terms (e.g., Eikonal, Total Variation) if defined in the specific variant [S8][S16, unofficial].

5.  **Backpropagation and Update:**
    *   Zero the gradients of the optimizer [S18, unofficial].
    *   Compute gradients: $\nabla_\theta \mathcal{L}$ via automatic differentiation [S18, unofficial].
    *   Step the optimizer: $\theta \leftarrow \theta - \eta \nabla_\theta \mathcal{L}$ [S18, unofficial].
    *   Update the learning rate according to the scheduler [S18, unofficial].

6.  **Logging and Checkpointing:**
    *   Log loss and PSNR metrics at regular intervals [S11, S14, unofficial].
    *   Render validation images at regular intervals [S11, unofficial].
    *   Save model checkpoints at regular intervals [S11, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_training_loop(dataset, model, optimizer, scheduler, num_iters, batch_size, render_interval, save_interval):
    """
    Standard NeRF training loop.
    """
    # 1. Initialization
    # Pre-compute rays if batching is enabled
    if use_batching:
        rays_rgb = precompute_rays(dataset) # Shape: [TotalPixels, 3, 3]
        rays_rgb = shuffle(rays_rgb)
        i_batch = 0
    
    model.train()
    
    for step in range(num_iters):
        # 2. Ray Sampling
        if use_batching:
            # Slice pre-computed batch
            batch = rays_rgb[i_batch : i_batch + batch_size]
            i_batch += batch_size
            if i_batch >= len(rays_rgb):
                rays_rgb = shuffle(rays_rgb)
                i_batch = 0
            # batch shape: [batch_size, 3, 3] -> [origins, dirs, targets]
            rays_o = batch[:, 0, :]
            rays_d = batch[:, 1, :]
            targets = batch[:, 2, :]
        else:
            # Random sampling per iteration
            img_idx = random.randint(0, len(dataset) - 1)
            rays_o, rays_d, targets = sample_random_rays_from_image(dataset[img_idx], batch_size)
            
        # 3. Forward Pass
        optimizer.zero_grad()
        
        # Render coarse and fine if applicable
        outputs = model.render(rays_o, rays_d, near=2.0, far=6.0)
        rgb_fine = outputs['rgb']
        
        # 4. Loss Computation
        loss_fine = mse_loss(rgb_fine, targets)
        loss = loss_fine
        
        if 'rgb_coarse' in outputs:
            loss_coarse = mse_loss(outputs['rgb_coarse'], targets)
            loss = loss + loss_coarse
            
        # 5. Backpropagation
        loss.backward()
        optimizer.step()
        scheduler.step()
        
        # 6. Logging and Checkpointing
        if step % 100 == 0:
            psnr = mse2psnr(loss.item())
            log_metric('loss', loss.item())
            log_metric('psnr', psnr)
            
        if step % render_interval == 0:
            render_validation_images(model, dataset)
            
        if step % save_interval == 0:
            save_checkpoint(model, optimizer, step)
            
    return model
```

### Parameters
*   **Learning Rate (`lr`):** Typically $5 \times 10^{-4}$ for Adam [S11, unofficial].
*   **Batch Size (`N_rand`):** Typically 1024 or 2048 rays per iteration [S5, S11, unofficial].
*   **Number of Iterations (`n_iters`):** Typically 100,000 to 1,000,000 steps [S11, S14, unofficial].
*   **Decay:** Exponential decay with `decay_rate` = 0.1 and `decay_steps` = `lrate_decay` * 1000 [S5, S18, unofficial].

### Edge Cases and Pitfalls
*   **VRAM Overflow:** Rendering full images is memory-intensive. Mitigated by chunking rays or using small batch sizes [S5, S15, unofficial].
*   **Instability in Growing Networks:** Directly initializing grown sub-networks randomly causes density collapse. Initialize alpha weights to match the parent network [S1].
*   **Gradient Freeing:** Higher-order derivatives (for logging gradients/laplacians) must be computed before `loss.backward()` [S3, unofficial].
*   **Checkpoint Resumption:** When resuming, the scheduler's last epoch must be set to `start_epoch - 1` to maintain correct decay [S9, unofficial].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
Training Ray Marching and Integration is the differentiable forward pass that synthesizes RGB images from a neural radiance field for gradient-based optimization. It solves the problem of efficiently sampling 3D space along camera rays, querying a neural network for density and color, and integrating these values via alpha compositing to produce a loss-computable image. It sets itself apart from [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md) by computing gradients for backpropagation and from [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) by including the adaptive sampling and occupancy grid pruning steps specific to the training loop.

### [Training Configuration](2332-training-configuration.md)
Training Configuration defines the static hyperparameters, optimizer settings, and loss function weights required to stabilize and accelerate the optimization of differentiable volume rendering models. It solves the problem of translating theoretical radiance field formulations into convergent training procedures by specifying exact network structures, sampling strategies, and optimization schedules. Unlike [Training Loop Setup](1522-training-loop-setup.md), which defines the dynamic runtime sequence of data loading and gradient updates, Training Configuration provides the immutable parameters that govern the optimization landscape. It differs from [Regularization and Loss Functions](8586-regularization-and-loss-functions.md) by focusing on the *values* of the weights and the *optimizer* mechanics rather than the mathematical form of the loss terms themselves.

### [Mip-NeRF 360](3306-mip-nerf-360.md)
Mip-NeRF 360 is a method for synthesizing photorealistic views of unbounded, real-world scenes where the camera may rotate 360 degrees and content exists at arbitrary distances. It solves the problems of aliasing, training instability, and geometric ambiguity inherent in standard NeRF when applied to large-scale environments. It sets itself apart from its parent [NeRF Training Loop](970-nerf-training-loop.md) by integrating a non-linear scene parameterization, online distillation between two MLPs, and a distortion-based regularizer, achieving a 57% reduction in mean squared error compared to standard mip-NeRF.

### [Mip-NeRF 360 Sampling and Training](3393-mip-nerf-360-sampling-and-training.md)
Mip-NeRF 360 Sampling and Training extends the Mip-NeRF framework to unbounded, 360-degree scenes by introducing a disparity-based ray parameterization, a contraction function for coordinate space, and a coarse-to-fine training loop with a proposal network. It solves the artifacts of "floaters" and "background collapse" common in unbounded scene reconstruction by regularizing the density distribution and ensuring efficient sampling through online distillation. It differs from standard NeRF by using linear-in-disparity sampling and from basic Mip-NeRF by handling unbounded scenes via coordinate contraction and disparity parameterization.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [MVSNeRF](6197-mvsnerf.md)
MVSNeRF is a method for fast, generalizable radiance field reconstruction from multi-view stereo (MVS) images. Unlike standard NeRF which requires per-scene optimization, MVSNeRF uses a feedforward neural network to predict the radiance field directly from input images, significantly reducing training time and iteration counts. It sets itself apart by learning a data prior that accelerates convergence, enabling rapid inference without iterative scene-specific optimization.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
This family of methods addresses the problem of optimizing Neural Radiance Fields (NeRF) when input camera poses are unknown, erroneous, or when intrinsic parameters (such as focal length) are not provided. Standard NeRF training assumes precise, pre-computed camera extrinsics and intrinsics; these methods remove that dependency by treating camera parameters as trainable variables within a fully differentiable pipeline, enabling joint optimization by minimizing the reconstruction loss. They set themselves apart from standard NeRF training by introducing mechanisms—such as coarse-to-fine frequency scheduling, local/global geometric alignment, or monocular depth priors—to stabilize the optimization landscape, which is otherwise ill-posed and prone to local minima when poses are free variables.

### [MegaNeRF](8580-meganerf.md)
Mega-NeRF is a framework for training large-scale 3D scenes that support interactive human-in-the-loop fly-throughs, addressing challenges of modeling thousands of images with varying lighting, large model capacities, and fast rendering. It differs from its siblings by using spatial partitioning to enable data parallelism for scalable training, whereas methods like DeRF and KiloNeRF use partitioning to speed up inference of a pre-trained single network. It improves upon [NeRF++](6197-nerf-pp.md) by modeling the foreground with an ellipsoid that tightly encloses camera poses and adding altitude constraints to skip underground sampling.

### [BlockNeRF](8581-blocknerf.md)
BlockNeRF is a method that decomposes a large-scale scene into a set of spatial blocks for separate NeRF training using a 'total-part-total' approach to solve large-scale city building complex rendering problems. It decouples rendering time from scene size to allow rendering to scale to arbitrarily large environments. It sets itself apart from standard NeRF by partitioning the scene geometry and appearance into localized MLPs, addressing the high-frequency information loss and slow inference inherent in single-model approaches for urban-scale data.

### [Specific Model Architectures](8585-specific-model-architectures.md)
Specific Model Architectures defines the structural variations of the neural networks used to represent the radiance field function $f_\theta(\mathbf{x}, \mathbf{d})$, including changes to input dimensions, output channels, connectivity, and auxiliary modules. While the parent [NeRF Training Loop](970-nerf-training-loop.md) defines the optimization procedure, these architectures determine how geometric and photometric information is encoded and decoded. This family distinguishes itself from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) (which handles sampling) and [Training Configuration](2332-training-configuration.md) (which handles hyperparameters) by focusing on the internal topology of the MLP or grid-based representations. Key distinctions include replacing dense MLPs with spatial partitions [KiloNeRF](8942-kilonerf.md), integrating external priors like depth maps [DASNeRF](8668-dasnerf.md), or modifying the output to support generative tasks [HyperNeRFGAN](8765-hypernerfgan.md).

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

### [Two-Stage Sampling with Occupancy Grid and Nonlinear Function](9146-two-stage-sampling-with-occupancy-grid-and-nonlinear-functio.md)
This method is a sampling strategy for neural implicit surface reconstruction that combines an Occupancy Grid to skip empty regions during a coarse stage and a nonlinear piecewise exponential function to guide sampling points closer to the object surface within occupied regions during a fine stage. It solves the problem of inefficient sampling in neural radiance fields by reducing training time and sample count while maintaining high-quality surface reconstruction. Unlike NeuS, which uses uniform sampling followed by inverse transform sampling that scatters points randomly near the surface, and Instant-NGP, which uses uniform sampling within occupied grid cells, this method uses a nonlinear function to bias samples towards the surface within the occupied regions.

### ReconFusion
Uses a Diffusion Prior to constrain unobserved views [S19, unofficial].

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S20]. For every pixel, cast multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel. Render each ray with NeRF. Use the mean of those rendered values as the predicted color in the loss function [S20]. The full-resolution supersampling baseline relies on 'oracle' scale information that does not exist for most datasets [S20]. The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S20].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S21, unofficial]. Phase 1: Train the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S21, unofficial]. Phase 2: Train all network components end-to-end. At the start of this phase, reset the optimizer state and perform a linear warmup for 1,000 steps to reduce shock from switching regimes [S21, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S21, unofficial].

### ViP-NeRF
A method that introduces visibility priors to supervise density and improve rendering quality under sparse input view conditions (fewer than 50-100 views), preventing artifacts caused by insufficient training data [S22]. 1) Visibility regularization: Regularize NeRF with visibility priors. 2) Visibility prior: Estimate whether each pixel in a main view is visible in a secondary view. 3) Effective prediction of visibility: Compute visibility using planar scanned volumes without pre-training [S22]. Create a plane scan volume by warping different depth planes of a secondary view image to the main view. Compare the warped planes with the main view image to obtain an error map. Identify the minimum error among all planes [S22]. Threshold the minimum error of all planes to obtain a visibility prior map corresponding to the primary view image. Use this map to supervise the density during NeRF training [S22]. On the RealEstate-10K dataset, ViP-NeRF achieves a PSNR index 3-4 points higher than the latest paper in 2, 3, and 4 input views [S22].

### Multi-view SDS
A plug-and-play method that creates multi-view supervision during SDS optimization to address the Janus problem caused by one-by-one view sampling [S23]. Rendered images from 4 sampled views are stitched into a large 2x2 image [S23]. Gradients are calculated under the condition of the multi-view normal map [S23]. The method converts a 3D asset into differentiable representations including NeRF, 3DGS, and DMTet through rendering supervision [S23]. The representations are then optimized to improve the quality of the 3D asset [S23].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot N_{samples} \cdot C_{net})$, where $N_{iters}$ is total iterations, $N_{rand}$ is batch size, $N_{samples}$ is points per ray, and $C_{net}$ is network query cost [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot N_{samples} \cdot C_{net})$ for the forward pass, plus storage for the model and optimizer states [S5, unofficial].
*   **Performance:** Standard NeRF training on Blender scenes takes ~100,000 iterations [S11, unofficial]. HashNeRF variants can converge faster due to efficient sampling [S7, unofficial].
*   **Strengths:** End-to-end differentiable; handles complex geometry and appearance [S5, unofficial].
*   **Limitations:** Slow convergence; requires significant VRAM for large batches; prone to local minima without good initialization [S1][S17, unofficial].
*   **When to Use:** When high-quality novel view synthesis is required and computational resources are available [S5, unofficial].
*   **When Not to Use:** For real-time applications without acceleration (e.g., Instant-NGP) or when camera poses are unknown without pose optimization extensions [S1][S15, unofficial].

## Background
The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, unofficial]. It established the standard paradigm of optimizing a coordinate MLP via stochastic gradient descent to minimize photometric error. Subsequent developments include HashNeRF [S7, unofficial], Mip-NeRF [S20], and various acceleration techniques like occupancy grids [S15, unofficial].

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
