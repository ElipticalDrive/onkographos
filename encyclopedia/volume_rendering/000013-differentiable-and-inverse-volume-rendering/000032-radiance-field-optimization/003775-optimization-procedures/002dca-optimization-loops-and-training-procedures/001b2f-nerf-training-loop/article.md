# NeRF Training Loop

Also called: End-to-End NeRF Training, HashNeRF Training Loop, HoloINR Training Loop, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric difference between rendered images and ground-truth observations by updating the parameters of a neural radiance field [S1][S5, S6, unofficial][S8][S10, S11, S14, unofficial]. It solves the inverse problem of reconstructing a continuous 3D scene representation from a set of 2D images by treating the radiance field as a differentiable function [S5, S6, S14, unofficial]. It sets itself apart from [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) by including adaptive sampling, occupancy grid pruning, and gradient-based weight updates specific to the training phase [S1][S2, unofficial][S8].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of $N$ training images with known camera poses (intrinsics $K$, extrinsics $C2W$) and pixel coordinates; a neural network model $f_\theta$ (MLP or hash-based) parameterized by $\theta$; hyperparameters including learning rate, batch size, and sampling counts [S2, S4, S5, S10, S11, unofficial].
*   **Outputs:** Optimized network weights $\theta$ and optionally a saved checkpoint of the model state [S8][S10, S11, unofficial].

**Data Structures:**
*   **Rays:** A batch of rays is represented as a tensor of shape $[B, 3, 3]$ or $[B, 6]$ (depending on implementation), where $B$ is the batch size ($N_{rand}$) [S5, S14, unofficial]. The channels typically contain:
    1.  Ray origin $(o_x, o_y, o_z)$
    2.  Ray direction $(d_x, d_y, d_z)$
    3.  Target ground-truth RGB color $(r, g, b)$ [S5, S14, unofficial].
*   **Pre-computed Rays Array:** To avoid repeated ray generation, many implementations pre-compute all possible rays from all training images into a flat array `rays_rgb` of shape $[(N-1) \cdot H \cdot W, 3, 3]$ (or similar flattened structure) and shuffle it [S5, S14, unofficial].
*   **Checkpoint:** A dictionary containing `model` (state dict), `optimizer` (state dict), and `epoch` or `step` counter [S10, S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S10, S11, unofficial].
    *   Initialize the neural network model $f_\theta$. For standard NeRF, this is an MLP; for HashNeRF, it involves initializing the hash table and feature vectors [S7, S15, unofficial].
    *   Initialize the optimizer (typically Adam or AdamW) with a learning rate $lr$ (e.g., $5 \times 10^{-4}$) [S5, S11, S13, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay, Cosine Annealing, or LambdaLR) [S5, unofficial][S8][S10, S11, unofficial].
    *   If using a coarse-to-fine architecture, initialize both coarse and fine networks [S11, unofficial].

2.  **Iteration Loop:**
    For each iteration $i$ from $1$ to $N_{iters}$ (typically 100,000 to 1,000,000) [S11, S14, unofficial]:

    3.  **Ray Sampling:**
        *   Select a batch of $N_{rand}$ rays (default 1024 or 2048) from the pre-computed rays array or by randomly sampling pixel coordinates from training images [S5, S14, unofficial].
        *   If using batching, increment the batch index and shuffle the rays array when exhausted [S5, S14, unofficial].
        *   Extract ray origins, directions, and target RGB colors into tensors `batch_rays` and `target_s` [S5, S14, unofficial].
        *   *Optional:* Apply perturbations to sampling positions if `perturb > 0` to break symmetry [S5, unofficial].
        *   *Optional:* If using adaptive sampling or occupancy grids, query the grid to determine valid sampling intervals [S15, unofficial].

    4.  **Forward Pass (Differentiable Volume Rendering):**
        *   Pass `batch_rays` through the rendering function (e.g., `render_rays`) [S5, S11, unofficial].
        *   The renderer samples points along each ray, queries the neural network for density ($\sigma$) and color ($\mathbf{c}$), and integrates them using alpha compositing [S1][S5, S6, unofficial].
        *   If using a coarse-to-fine scheme, the coarse network produces an initial color $\hat{C}_c$ and density distribution, which guides the sampling of the fine network to produce $\hat{C}_f$ [S11, unofficial].
        *   The output is the predicted RGB color $\hat{C}$ (and optionally depth, accumulation, and coarse outputs) [S5, S6, S11, unofficial].

    5.  **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) between the predicted color $\hat{C}$ and the target color $C_{gt}$ [S3, S6, unofficial][S12][S13, S14, unofficial].
        $$ \mathcal{L}_{photo} = \frac{1}{B} \sum_{b=1}^{B} \| \hat{C}_b - C_{gt, b} \|^2 $$
        *   If a coarse network is used, compute the coarse loss $\mathcal{L}_c$ and add it to the total loss: $\mathcal{L} = \mathcal{L}_c + \mathcal{L}_f$ [S11, unofficial].
        *   *Optional:* Add regularization terms (e.g., sparsity, total variation, eikonal) if specified in the configuration [S7, unofficial][S8][S16, unofficial].

    6.  **Backpropagation:**
        *   Zero the gradients of the optimizer [S3, S18, unofficial].
        *   Compute gradients: `loss.backward()` [S3, S18, unofficial].
        *   *Pitfall:* If logging higher-order derivatives (gradients, Laplacians) for visualization, this must be done *before* `loss.backward()` as the backward pass frees the computation graph [S3, unofficial].

    7.  **Optimization Step:**
        *   Update model parameters: `optimizer.step()` [S3, S18, unofficial].
        *   Update the learning rate according to the scheduler [S8][S10, S18, unofficial].
        *   *Exponential Decay Example:* $lr_{new} = lr_{old} \times \text{decay\_rate}^{(\text{step} / \text{decay\_steps})}$ [S5, S18, unofficial].

    8.  **Logging and Checkpointing:**
        *   Log loss and PSNR every $i_{print}$ steps (e.g., 100) [S2, S11, unofficial].
        *   Render validation images every $i_{img}$ steps (e.g., 500) [S2, S11, unofficial].
        *   Save model checkpoint every $i_{weights}$ steps (e.g., 10,000) [S2, S11, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_training_loop(model, optimizer, scheduler, dataloader, num_iters, n_rand, device):
    """
    Standard NeRF training loop.
    """
    # 1. Initialization
    model.train()
    global_step = 0
    
    # Pre-compute rays if not using dynamic generation
    # rays_rgb shape: [total_pixels, 3, 3] -> [origin, direction, target_rgb]
    rays_rgb = precompute_rays(dataloader) 
    rays_rgb = rays_rgb.flatten(0, 1) # Flatten to [N, 3, 3]
    
    for step in range(num_iters):
        # 2. Ray Sampling
        # Sample N_rand indices from rays_rgb
        indices = torch.randint(0, rays_rgb.shape[0], (n_rand,))
        batch_rays = rays_rgb[indices] # Shape: [N_rand, 3, 3]
        
        # Split into ray params (origin, dir) and targets (rgb)
        ray_params = batch_rays[:, :2, :] # [N_rand, 2, 3] -> origin, dir
        target_rgb = batch_rays[:, 2, :]  # [N_rand, 3]
        
        # 3. Forward Pass
        optimizer.zero_grad()
        
        # Render the batch
        # render_rays returns predicted_rgb, and optionally coarse_rgb, extras
        predicted_rgb, coarse_rgb, extras = render_rays(model, ray_params, device)
        
        # 4. Loss Computation
        # Primary photometric loss
        loss = torch.nn.functional.mse_loss(predicted_rgb, target_rgb)
        
        # Add coarse loss if available (coarse-to-fine training)
        if coarse_rgb is not None:
            loss_coarse = torch.nn.functional.mse_loss(coarse_rgb, target_rgb)
            loss = loss + loss_coarse
            
        # Optional: Add regularization losses here (e.g., sparsity, TV)
        
        # 5. Backpropagation
        loss.backward()
        
        # 6. Update
        optimizer.step()
        scheduler.step()
        
        # 7. Logging
        if step % 100 == 0:
            psnr = -10.0 * torch.log10(loss.item())
            print(f"Step {step}: Loss {loss.item():.6f}, PSNR {psnr:.2f}")
            
        # 8. Checkpointing
        if step % 10000 == 0:
            save_checkpoint(model, optimizer, step)
            
        global_step += 1
```

**Parameters with Typical Values:**
*   $N_{rand}$ (Batch Size): 1024 or 2048 rays per iteration [S5, S11, S14, unofficial].
*   Learning Rate ($lr$): $5 \times 10^{-4}$ for Adam [S11, unofficial].
*   Total Iterations ($N_{iters}$): 100,000 to 1,000,000 [S11, S14, unofficial].
*   Decay Steps: $args.lrate\_decay \times 1000$ (e.g., 100,000 steps) [S5, S18, unofficial].
*   Decay Rate: 0.1 [S5, S18, unofficial].
*   Positional Encoding Frequency ($L$): 10 for position, 4 for direction [S11, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering all rays at once can exceed GPU memory. Mitigated by chunking rays or using smaller batch sizes ($N_{rand}$) [S5, S15, unofficial].
*   **Instability in Growing Networks:** Directly initializing grown sub-networks randomly causes density to drop to zero. Initialize alpha weights of new networks to match the parent network [S1].
*   **Empty Checkpoints:** If a checkpoint is loaded but the step counter is 0, it is considered invalid/empty [S15, unofficial].
*   **Gradient Graph Freeing:** Higher-order derivative calculations for logging must occur before `loss.backward()` [S3, unofficial].
*   **Learning Rate Scheduling Resumption:** When resuming from a checkpoint, the scheduler's last epoch must be set to `start_epoch - 1` to maintain correct decay [S9, unofficial].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
Training Ray Marching and Integration is the differentiable forward pass that synthesizes RGB images from a neural radiance field for gradient-based optimization. It solves the problem of efficiently sampling 3D space along camera rays, querying a neural network for density and color, and integrating these values via alpha compositing to produce a loss-computable image. It sets itself apart from [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md) by computing gradients for backpropagation and from [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) by including the adaptive sampling and occupancy grid pruning steps specific to the training loop.

### [Training Configuration](2332-training-configuration.md)
Training Configuration defines the static hyperparameters, optimizer settings, and structural constraints required to stabilize the optimization of differentiable volume rendering models. It solves the problem of translating theoretical radiance field formulations into convergent training procedures by specifying exact network architectures, sampling strategies, and loss weights. Unlike [Training Loop Setup](1522-training-loop-setup.md), which defines the dynamic runtime sequence of data loading and gradient updates, this method specifies the immutable parameters governing the optimization landscape. It differs from [Regularization and Priors](6281-regularization-and-priors.md) by focusing on the numerical optimization settings (e.g., learning rates, batch sizes) rather than the specific penalty terms added to the loss function.

### [Mip-NeRF 360](3306-mip-nerf-360.md)
Mip-NeRF 360 is a method for synthesizing photorealistic views of unbounded, real-world scenes where the camera may rotate 360 degrees and content exists at arbitrary distances. It solves the problems of aliasing, training instability, and geometric ambiguity inherent in standard NeRF when applied to large-scale environments. It sets itself apart from its parent [NeRF Training Loop](970-nerf-training-loop.md) by integrating a non-linear scene parameterization, online distillation between two MLPs, and a distortion-based regularizer, achieving a 57% reduction in mean squared error compared to standard mip-NeRF.

### [Mip-NeRF 360 Sampling and Training](3393-mip-nerf-360-sampling-and-training.md)
Mip-NeRF 360 Sampling and Training extends the Mip-NeRF framework to unbounded, 360-degree scenes by introducing a disparity-based ray parameterization, a contraction function for coordinate space, and a coarse-to-fine training loop with a proposal network. It solves the artifacts of "floaters" and "background collapse" common in unbounded scene reconstruction by regularizing the density distribution and ensuring efficient sampling through online distillation. It differs from standard NeRF by using linear-in-disparity sampling and from basic Mip-NeRF by handling unbounded scenes via coordinate contraction and disparity parameterization.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [MVSNeRF](6197-mvsnerf.md)
MVSNeRF is a method for fast, generalizable radiance field reconstruction from multi-view stereo (MVS) images. Unlike standard NeRF which requires per-scene optimization, MVSNeRF uses a feedforward neural network to predict the radiance field directly from input images, significantly reducing training time and iteration counts. It sets itself apart by learning a data prior that accelerates convergence, enabling rapid inference without iterative scene-specific optimization.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
This family of methods addresses the problem of jointly optimizing Neural Radiance Field (NeRF) network parameters and camera poses (intrinsics and extrinsics) when ground-truth camera parameters are unavailable, inaccurate, or require refinement. Standard NeRF training assumes fixed, known camera poses; these methods treat poses as differentiable trainable parameters within the optimization loop, enabling end-to-end bundle adjustment. They are distinguished from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by their explicit gradient updates to camera parameters, and from [Mip-NeRF 360](3306-mip-nerf-360.md) by their focus on pose uncertainty rather than angular resolution.

### [Specific Model Architectures](8585-specific-model-architectures.md)
Specific Model Architectures refers to the family of neural network structures and representations used to parameterize the radiance field $f_\theta(\mathbf{x}, \mathbf{d})$ within the NeRF training loop. While the standard NeRF uses a simple MLP mapping 3D coordinates and viewing directions to density and color, this family encompasses modifications to the network topology, input representations, and output mechanisms to address limitations such as training speed, generalization, memory efficiency, and handling of complex phenomena like lighting changes or reflections. These architectures set themselves apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the *function* $f_\theta$ itself rather than the *sampling* strategy or *loss* functions.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
This family encompasses loss functions and regularization techniques applied during the optimization of Neural Radiance Fields (NeRFs) to mitigate the ill-posed nature of inverse volume rendering, particularly in sparse-view or unbounded settings. While standard NeRF training relies solely on photometric consistency ($\mathcal{L}_{\text{photo}}$), these methods augment the objective with geometric, statistical, or diffusion-based priors to suppress artifacts such as "floaters," overfitting, and view-dependent hallucinations. They set themselves apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) (which focuses on sampling efficiency) and [Training Configuration](2332-training-configuration.md) (which focuses on hyperparameter schedules) by directly modifying the gradient signal to enforce structural plausibility.

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
Progressive Inpainting-and-Erasing (PERF) is a training strategy for single-view NeRF that avoids geometry conflicts between visible and invisible regions by computing a mask of conflicted regions and eliminating them from NeRF training. It sets itself apart from standard single-view NeRF methods by integrating a collaborative RGBD inpainting step with a progressive erasing mechanism to handle the ambiguity of unseen geometry, building on the [NeRF Training Loop](970-nerf-training-loop.md) by dynamically expanding the reference view set.

### [NeRF-FF](8669-nerf-ff.md)
NeRF-FF is a plug-in strategy that mitigates the degradation of Neural Radiance Field (NeRF) reconstruction quality caused by out-of-focus (defocus blurred) regions in input images. It solves the problem of training NeRFs on partially blurred data by estimating binary image masks that identify in-focus regions, allowing the subsequent NeRF model to omit out-of-focus pixels during training. Unlike methods that use dynamic ray generation or blur kernels, NeRF-FF relies on static ray assumptions, making it compatible with runtime-optimized NeRF variants like Instant-NGP, nerfacto, and splatfacto.

### [KNN-NeRF](8672-knn-nerf.md)
KNN-NeRF is a baseline method for training Neural Radiance Fields (NeRFs) using sparse 3D point clouds as supervision instead of multi-view images. It solves the problem of generating ground-truth color labels for rays that do not pass through existing points by assigning the color of the nearest point in the cloud to the ray. Unlike standard NeRF which relies on pixel colors from images, KNN-NeRF constructs a dense training signal from sparse geometry, though it struggles with sharp object borders due to the low-frequency bias of neural networks and the sparsity of the input.

### [Hard Sample Mining](8673-hard-sample-mining.md)
Hard Sample Mining (HSM) is an optimization procedure for neural radiance fields that improves training efficiency and reconstruction quality by selectively updating the network based on pixel-level error. It solves the problem of uniform computational waste in standard NeRF training, where samples contributing little to the gradient are treated equally to those with high error. Unlike sibling methods such as [Probability-based Ray Sampling with Quadtree Subdivision](7481-probability-based-ray-sampling-with-quadtree-subdivision.md) or [Nerfacc](https://github.com/NVlabs/nerfacc) which prune low-weight samples based on density or transmittance, HSM ranks samples using the propagated pixel loss to subsample a "hard" minibatch that retains most of the update information.

### [NeRF-ID](8867-nerf-id.md)
NeRF-ID (NeRF in detail) is a differentiable volume rendering optimization method that replaces the heuristic coarse-to-fine sampling procedure of vanilla NeRF with a learnable "proposer" module. It solves the problem of suboptimal sample placement by enabling end-to-end training of the coarse network, proposer, and fine network jointly, thereby improving view synthesis quality. Unlike its sibling [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) which relies on handcrafted probability density functions derived from coarse densities, NeRF-ID uses a neural network to directly predict fine sample locations from coarse features.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Progressive and Dynamic Scene Growth addresses the challenge of reconstructing large-scale or complex 3D scenes from limited input data by incrementally expanding the neural representation's capacity or the training dataset. Unlike static NeRF architectures that require pre-defining the entire hierarchy or fixed network size, these methods adaptively grow the model structure or add synthetic reference views during training to resolve occlusions and capture high-frequency details. This family distinguishes itself from sibling methods like [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) (which focuses on ray selection) and [Sparse NeRF Reconstruction with Geometric and Semantic Initialization](8779-sparse-nerf-reconstruction-with-geometric-and-semantic-initi.md) (which relies on external priors) by actively modifying the model's topology or the set of supervision signals to improve reconstruction quality in complex regions.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Adaptive Ray Sampling refers to a family of strategies in neural radiance field (NeRF) training that dynamically adjust the distribution of sample points along viewing rays to improve reconstruction quality and training efficiency. Unlike uniform sampling, which treats all regions of a ray equally, these methods allocate computational resources to areas of high geometric or photometric complexity, such as object boundaries or high-frequency details. This family addresses the inefficiency of uniform sampling in sparse or complex scenes by clustering samples where they contribute most to the final rendered image.

### [Drone-NeRF](8972-drone-nerf.md)
Drone-NeRF is a method for large-scale 3D scene reconstruction from drone-captured imagery that addresses the specific challenges of overhead drone views, including exposure variations, shadow artifacts from vertical surfaces, and computational efficiency in large scenes. It differs from standard NeRF approaches by employing a block-parallel training strategy where sub-scenes are trained independently and then merged, while explicitly handling exposure discrepancies via appearance embeddings and removing occlusion shadows through an expanded bounding box projection technique. Unlike methods that treat the entire scene as a single volume, Drone-NeRF partitions the space into sub-bounding boxes, optimizing them separately before a final merging phase that ensures color uniformity and depth continuity across block boundaries.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
This family addresses the degradation of NeRF quality—such as blurred geometry, slow convergence, and floaters—when trained on sparse input views (typically 15–20 images or fewer than 50–100 views). It distinguishes itself from standard NeRF training by decoupling the initialization and regularization processes from the default random or empty-space assumptions, instead leveraging external geometric scaffolds, semantic masks, or visibility priors to anchor the optimization.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Scene Representation and Partitioning addresses the scalability limits of single-NeRF models by decomposing large or dynamic scenes into localized sub-representations. It enables training and rendering of arbitrarily large environments or complex temporal dynamics by distributing computational load across multiple specialized networks, rather than relying on a single monolithic MLP. It differs from siblings like [Mip-NeRF 360](3306-mip-nerf-360.md) which focuses on multi-scale sampling within a single representation, and [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) which optimizes point density within a single model, by fundamentally altering the scene's structural representation to allow parallelism or dynamic handling.

### ReconFusion
ReconFusion uses a Diffusion Prior to constrain unobserved views during the NeRF training loop [S19, unofficial].

### Supersampling Baseline
Supersampling Baseline is a baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S20]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S20]. This baseline relies on 'oracle' scale information that does not exist for most datasets [S20]. The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S20].

### Two-stage training for proposers
Two-stage training for proposers is a training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S21, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S21, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 SGD steps to reduce shock from switching regimes [S21, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S21, unofficial].

### Multi-view SDS
Multi-view SDS is a plug-and-play method that creates multi-view supervision during SDS optimization to address the Janus problem caused by one-by-one view sampling [S22]. Rendered images from 4 sampled views are stitched into a large 2x2 image, and gradients are calculated under the condition of the multi-view normal map [S22]. The method converts a 3D asset into differentiable representations including NeRF, 3DGS, and DMTet through rendering supervision, which are then optimized to improve the quality of the 3D asset [S22].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot N_{samples})$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the batch size (rays per step), and $N_{samples}$ is the number of samples per ray [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot N_{samples} \cdot C)$, where $C$ is the channel size (RGB + density) [S5, unofficial]. VRAM usage is a primary constraint, often mitigated by chunking rays [S5, S15, unofficial].
*   **Measured Performance:** Standard NeRF training on Blender datasets (800x800) typically takes 100,000–1,000,000 iterations [S11, S14, unofficial]. On 128x128 images, training may take 250 epochs [S4, unofficial].
*   **Scaling:** Performance scales with batch size, but larger batches hinder data loading performance and reduce throughput due to GPU occupancy limits; a trade-off (e.g., $2^{10}$) is often chosen [S15, unofficial].
*   **Strengths:** End-to-end differentiable, handles complex lighting and geometry via neural representation.
*   **Limitations:** Slow convergence due to high iteration counts; prone to "floaters" and overfitting without regularization; requires careful hyperparameter tuning (learning rate, decay) [S1][S5, S17, unofficial].
*   **When to Use:** When high-quality novel view synthesis is required and ground-truth 3D geometry is not available.
*   **When Not to Use:** When real-time training or inference is required (consider [Instant-NGP](link) or [3D Gaussian Splatting](link)); when sparse views are insufficient for unique reconstruction without strong priors [S1][S17, unofficial].

## Background
The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, S18, unofficial]. It established the paradigm of optimizing a continuous volumetric function represented by an MLP to minimize photometric error. Subsequent developments include HashNeRF (Müller et al., 2022) which uses hash-based feature grids for faster training [S7, unofficial], and Mip-NeRF (Barron et al., 2021) which introduced anti-aliasing via cone tracing [S20]. The loop has been extended to handle unbounded scenes (Mip-NeRF 360), sparse views, and dynamic scenes [S1][S11, S19, unofficial].

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
[S22] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
