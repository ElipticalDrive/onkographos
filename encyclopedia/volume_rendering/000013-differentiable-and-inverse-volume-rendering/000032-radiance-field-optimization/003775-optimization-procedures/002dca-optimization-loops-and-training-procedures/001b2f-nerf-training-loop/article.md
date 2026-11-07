# NeRF Training Loop

Also called: NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Fields Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric difference between rendered images and ground-truth training views by updating the parameters of a neural radiance field [S1][S5, unofficial][S6, unofficial]. It solves the inverse problem of reconstructing a continuous 3D scene representation from a discrete set of 2D images by treating the scene as a differentiable volume [S7, unofficial][S8]. It sets itself apart from [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md) by computing gradients for backpropagation and from [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) by including the adaptive sampling and occupancy grid pruning steps specific to the training loop [S1][S2, unofficial][S8].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of $N$ training images with known camera poses (extrinsics $C$) and intrinsics ($K$), where each image has height $H$, width $W$, and focal length $f$ [S2, unofficial][S5, unofficial]. The neural network parameters $\theta$ (weights and biases of the MLPs) [S5, unofficial].
*   **Outputs:** Optimized network parameters $\theta^*$ that minimize the reconstruction error [S5, unofficial].

**Data Structures:**
*   **Rays:** A ray is defined by origin $\mathbf{o}$ and direction $\mathbf{d}$. For a batch of $B$ rays, data is often pre-computed into a flat array `rays_rgb` of shape $[(N-1)HW, 3, 3]$ (axis 0: pixel index; axis 1: [origin, direction, target_rgb]; axis 2: components) [S5, unofficial][S14, unofficial]. Alternatively, rays are generated on-the-fly from pixel coordinates [S4, unofficial][S14, unofficial].
*   **Samples:** Points $\mathbf{x}_i$ along rays, parameterized by distance $t_i$ from the camera center, such that $\mathbf{x}_i = \mathbf{o} + t_i \mathbf{d}$ [S5, unofficial].
*   **Checkpoint:** A dictionary containing 'model' (state dict), 'optimizer' (state dict), and 'epoch' or 'step' [S10, unofficial][S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the neural network(s) $\theta$ [S5, unofficial][S15, unofficial]. For progressive growth methods, initialize alpha linear weights of grown sub-networks to match the parent to avoid instability [S1].
    *   Initialize the optimizer (typically Adam) with learning rate $lr$ [S5, unofficial][S11, unofficial].
    *   Initialize the learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, unofficial][S11, unofficial].
    *   Pre-compute all rays (origin, direction) and target pixels if batching is enabled [S11, unofficial].

2.  **Ray Sampling:**
    *   For each iteration, sample a batch of $N_{rand}$ rays [S5, unofficial][S14, unofficial].
    *   If using batching, slice the pre-computed `rays_rgb` array or randomly select pixel coordinates from training images [S5, unofficial][S14, unofficial].
    *   If `precrop_iters` is active, sample from a central crop of the image [S2, unofficial][S14, unofficial].
    *   Generate sample points $t_i$ along each ray within the near/far bounds $[t_{near}, t_{far}]$ [S5, unofficial]. Bounds are dataset-dependent: e.g., Blender uses $[2.0, 6.0]$, LLFF uses bounding depths or NDC $[0.0, 1.0]$ [S2, unofficial][S13, unofficial].
    *   Apply stratified sampling or jittering if `perturb > 0` to break symmetry [S2, unofficial][S5, unofficial].

3.  **Forward Pass (Differentiable Volume Rendering):**
    *   Query the neural network $f_\theta(\mathbf{x}, \mathbf{d})$ at each sample point to obtain volume density $\sigma$ and color $\mathbf{c}$ [S5, unofficial][S6, unofficial].
    *   Compute transmittance $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ and weights $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$ [S5, unofficial].
    *   Composite the color: $\hat{C}(\mathbf{r}) = \sum_{i=1}^K w_i \mathbf{c}_i$ [S5, unofficial].
    *   If using a coarse-fine strategy (two-stage), render with the coarse network, sample additional points based on coarse weights, and render with the fine network [S5, unofficial][S11, unofficial].

4.  **Loss Computation:**
    *   Compute the Mean Squared Error (MSE) between the predicted color $\hat{C}(\mathbf{r})$ and the ground-truth color $C(\mathbf{r})$ [S3, unofficial][S5, unofficial][S14, unofficial]:
        $$ \mathcal{L}_{\text{MSE}} = \frac{1}{B} \sum_{b=1}^B \| \hat{C}_b - C_b \|^2 $$
    *   If coarse output $\hat{C}_0$ is available, add its loss: $\mathcal{L} = \mathcal{L}_{\text{MSE}}(\hat{C}, C) + \mathcal{L}_{\text{MSE}}(\hat{C}_0, C)$ [S6, unofficial][S11, unofficial].
    *   Optionally add regularization terms (e.g., sparsity, total variation, eikonal) [S7, unofficial][S8].

5.  **Backpropagation and Update:**
    *   Compute gradients $\nabla_\theta \mathcal{L}$ via automatic differentiation [S5, unofficial][S18, unofficial].
    *   **Pitfall:** Higher-order derivatives (e.g., for logging gradients/laplacians) must be computed *before* calling `loss.backward()` to avoid freeing the autograd graph [S3, unofficial].
    *   Update parameters: $\theta \leftarrow \theta - \eta \nabla_\theta \mathcal{L}$ using the optimizer [S5, unofficial][S18, unofficial].
    *   Update learning rate according to the scheduler [S5, unofficial][S18, unofficial].

6.  **Logging and Checkpointing:**
    *   Log loss and PSNR ($\text{PSNR} = -10 \log_{10}(\mathcal{L}_{\text{MSE}})$) at intervals `i_print` [S2, unofficial][S14, unofficial].
    *   Render validation images at intervals `i_img` or `render_interval` [S2, unofficial][S8].
    *   Save checkpoints at intervals `i_weights` or `checkpoint_interval` [S2, unofficial][S8].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def nerf_training_loop(
    dataset, 
    model, 
    optimizer, 
    scheduler, 
    num_iters, 
    batch_size=1024, 
    near=2.0, 
    far=6.0, 
    perturb=1.0
):
    """
    Standard NeRF training loop.
    """
    # 1. Initialization
    model.train()
    rays_rgb = precompute_rays(dataset) # Shape: [N_pixels, 3, 3]
    i_batch = 0
    
    for step in range(num_iters):
        # 2. Ray Sampling
        # Slice batch from pre-computed rays or generate on-the-fly
        if i_batch + batch_size < rays_rgb.shape[0]:
            batch_data = rays_rgb[i_batch : i_batch + batch_size]
            i_batch += batch_size
        else:
            # Shuffle and restart when exhausted
            np.random.shuffle(rays_rgb)
            batch_data = rays_rgb[0 : batch_size]
            i_batch = batch_size
            
        # batch_data shape: [B, 3, 3] -> [origins, dirs, targets]
        rays_o = batch_data[:, 0, :] # [B, 3]
        rays_d = batch_data[:, 1, :] # [B, 3]
        targets = batch_data[:, 2, :] # [B, 3]
        
        # Sample points along rays
        t_vals = torch.linspace(0., 1., 128).to(rays_o.device)
        z_vals = near * (1. - t_vals) + far * (t_vals)
        
        # Stratified sampling for training
        if perturb > 0.:
            mids = .5 * (z_vals[..., 1:] + z_vals[..., :-1])
            upper = torch.cat([mids, z_vals[..., -1:]], -1)
            lower = torch.cat([z_vals[..., :1], mids], -1)
            t_rand = torch.rand(z_vals.shape).to(rays_o.device)
            z_vals = lower + (upper - lower) * t_rand
            
        pts = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * z_vals.unsqueeze(2) # [B, 128, 3]
        
        # 3. Forward Pass
        optimizer.zero_grad()
        
        # Query network for density and color
        # raw shape: [B, 128, 4] (RGB + sigma)
        raw = model(pts, rays_d) 
        
        # Volume Rendering
        alpha = raw2alpha(raw[..., 3], torch.diff(z_vals, dim=1)) # [B, 127]
        weights = alpha * torch.cumprod(1. - alpha + 1e-10, dim=1, exclusive=True)
        rgb_pred = torch.sum(weights[..., None] * raw[..., :3], dim=1) # [B, 3]
        
        # 4. Loss Computation
        loss = torch.nn.functional.mse_loss(rgb_pred, targets)
        
        # 5. Backpropagation
        loss.backward()
        optimizer.step()
        scheduler.step()
        
        # 6. Logging
        if step % 100 == 0:
            psnr = -10. * torch.log(loss) / torch.log(torch.tensor(10.0))
            print(f"Step {step} | Loss {loss.item():.6f} | PSNR {psnr.item():.2f}")
            
    return model
```

**Parameters with Typical Values:**
*   `batch_size` ($N_{rand}$): 1024 or 2048 rays per iteration [S5, unofficial][S11, unofficial].
*   `lr` (Adam learning rate): $5 \times 10^{-4}$ [S11, unofficial].
*   `num_iters`: 100,000 to 1,000,000 steps [S11, unofficial][S14, unofficial].
*   `near`, `far`: Scene-dependent. Blender: $[2.0, 6.0]$ [S2, unofficial][S13, unofficial]. LLFF: computed from bounding depths or NDC $[0.0, 1.0]$ [S2, unofficial].
*   `perturb`: 1.0 for training (jitter), 0.0 for evaluation [S2, unofficial].
*   `decay_rate`: 0.1 [S6, unofficial][S18, unofficial].
*   `decay_steps`: `args.lrate_decay * 1000` [S5, unofficial][S18, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Mitigated by chunking ray batches during the forward pass [S5, unofficial][S14, unofficial].
*   **Instability in Progressive Growth:** Directly growing a randomly initialized network causes density to drop to zero; sub-network weights must inherit from the parent [S1].
*   **Checkpoint Resumption:** When resuming, the scheduler's last epoch must be set to `start_epoch - 1` to ensure correct scheduling [S9, unofficial]. If a checkpoint step is 0, it is treated as empty [S15, unofficial].
*   **Gradient Freeing:** Higher-order derivatives for logging must be computed before `loss.backward()` [S3, unofficial].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
Training Ray Marching and Integration is the differentiable forward pass that synthesizes RGB images from a neural radiance field for gradient-based optimization. It solves the problem of efficiently sampling 3D space along camera rays, querying a neural network for density and color, and integrating these values via alpha compositing to produce a loss-computable image. It sets itself apart from [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md) by computing gradients for backpropagation and from [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) by including the adaptive sampling and occupancy grid pruning steps specific to the training loop.

### [Training Configuration](2332-training-configuration.md)
Training Configuration defines the static hyperparameters, optimizer settings, and structural constraints required to stabilize the optimization of differentiable volume rendering models. It solves the problem of translating theoretical radiance field formulations into convergent training procedures by specifying exact network architectures, sampling strategies, and loss weights. Unlike [Training Loop Setup](1522-training-loop-setup.md), which defines the dynamic runtime sequence of data loading and gradient updates, this method specifies the immutable parameters governing the optimization landscape. It differs from [Regularization and Priors](6281-regularization-and-priors.md) by focusing on the numerical optimization settings (e.g., learning rates, batch sizes) rather than the specific penalty terms added to the loss function.

### [Mip-NeRF 360](3306-mip-nerf-360.md)
Mip-NeRF 360 is a method for synthesizing photorealistic views of unbounded, real-world scenes where the camera may rotate 360 degrees and content exists at arbitrary distances. It solves the problems of aliasing, training instability, and geometric ambiguity inherent in standard NeRF when applied to large-scale environments. It sets itself apart from its parent [NeRF Training Loop](970-nerf-training-loop.md) by integrating a non-linear scene parameterization, online distillation between two MLPs, and a distortion-based regularizer, achieving a 57% reduction in mean squared error compared to standard mip-NeRF.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [MVSNeRF](6197-mvsnerf.md)
MVSNeRF is a method for fast, generalizable radiance field reconstruction from multi-view stereo (MVS) images. Unlike standard NeRF which requires per-scene optimization, MVSNeRF uses a feedforward neural network to predict the radiance field directly from input images, significantly reducing training time and iteration counts. It sets itself apart by learning a data prior that accelerates convergence, enabling rapid inference without iterative scene-specific optimization.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
This family of methods addresses the problem of optimizing Neural Radiance Field (NeRF) representations when camera poses (extrinsics) and/or intrinsic parameters are unknown, inaccurate, or require refinement. Standard NeRF training assumes perfect camera poses; these methods jointly optimize the network weights $\Theta$ and camera parameters $\pi$ (pose and intrinsics) within a differentiable pipeline. They set themselves apart from standard NeRF by treating camera parameters as learnable variables, allowing the model to self-calibrate or correct errors. Key distinctions among siblings include the use of coarse-to-fine frequency annealing ([BARF](6872-barf-bundle-adjusting-neural-radiance-fields.md)), monocular depth priors ([NoPe-NeRF](8705-nope-nerf.md), [AltNeRF](8763-altnerf.md)), or local-global alignment constraints ([L2G-NeRF](8671-l2g-nerf.md)).

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

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Progressive and Dynamic Scene Growth addresses the challenge of reconstructing large-scale or complex 3D scenes from limited input data by incrementally expanding the neural representation's capacity or the training dataset. Unlike static NeRF architectures that require pre-defining the entire hierarchy or fixed network size, these methods adaptively grow the model structure or add synthetic reference views during training to resolve occlusions and capture high-frequency details. This family distinguishes itself from sibling methods like [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) (which focuses on ray selection) and [Sparse NeRF Reconstruction with Geometric and Semantic Initialization](8779-sparse-nerf-reconstruction-with-geometric-and-semantic-initi.md) (which relies on external priors) by actively modifying the model's topology or the set of supervision signals to improve reconstruction quality in complex regions.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Adaptive Ray Sampling refers to a family of strategies in neural radiance field (NeRF) training that dynamically adjust the distribution of sample points along viewing rays to improve reconstruction quality and training efficiency. Unlike uniform sampling, which treats all space along a ray equally, these methods concentrate computational resources in regions of high geometric or photometric complexity, such as object surfaces or high-frequency details. This family is distinct from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing specifically on *where* to sample rather than *how* to integrate, and from [Mip-NeRF 360](3306-mip-nerf-360.md) by typically operating in Euclidean space with point-wise samples rather than conical frustums.

### [Drone-NeRF](8972-drone-nerf.md)
Drone-NeRF is a method for large-scale 3D scene reconstruction from drone-captured imagery that addresses the specific challenges of overhead drone views, including exposure variations, shadow artifacts from vertical surfaces, and computational efficiency in large scenes. It differs from standard NeRF approaches by employing a block-parallel training strategy where sub-scenes are trained independently and then merged, while explicitly handling exposure discrepancies via appearance embeddings and removing occlusion shadows through an expanded bounding box projection technique. Unlike methods that treat the entire scene as a single volume, Drone-NeRF partitions the space into sub-bounding boxes, optimizing them separately before a final merging phase that ensures color uniformity and depth continuity across block boundaries.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
This family addresses the degradation of NeRF quality—such as blurred geometry, slow convergence, and floaters—when trained on sparse input views (typically 15–20 images or fewer than 50–100 views). It distinguishes itself from standard NeRF training by decoupling the initialization and regularization processes from the default random or empty-space assumptions, instead leveraging external geometric scaffolds, semantic masks, or visibility priors to anchor the optimization.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Scene Representation and Partitioning addresses the scalability limits of single-NeRF models by decomposing large or dynamic scenes into localized sub-representations. It enables training and rendering of arbitrarily large environments or complex temporal dynamics by distributing computational load across multiple specialized networks, rather than relying on a single monolithic MLP. It differs from siblings like [Mip-NeRF 360](3306-mip-nerf-360.md) which focuses on multi-scale sampling within a single representation, and [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) which optimizes point density within a single model, by fundamentally altering the scene's structural representation to allow parallelism or dynamic handling.

### ReconFusion
Uses a Diffusion Prior to constrain unobserved views [S20, unofficial].

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, cast multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel. Render each ray with NeRF. Use the mean of those rendered values as the predicted color in the loss function [S21]. The full-resolution supersampling baseline relies on 'oracle' scale information that does not exist for most datasets [S21]. Performance: The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1: Train the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets. Phase 2: Train all network components end-to-end. At the start of this phase, reset the optimizer state and perform a linear warmup for 1,000 steps to reduce shock from switching regimes [S22, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S22, unofficial].

### Multi-view SDS
A plug-and-play method that creates multi-view supervision during SDS optimization to address the Janus problem caused by one-by-one view sampling [S23]. Rendered images from 4 sampled views are stitched into a large 2x2 image. Gradients are calculated under the condition of the multi-view normal map. The method converts a 3D asset into differentiable representations including NeRF, 3DGS, and DMTet through rendering supervision. The representations are then optimized to improve the quality of the 3D asset [S23].

### Guided NeRF Optimization for MVS
A multi-view depth estimation method that utilizes conventional reconstruction (sparse SfM+MVS) and learning-based priors over Neural Radiance Fields (NeRF). It performs test-time optimization directly over implicit volumes to eliminate the need for accurate pixel matching across views, addressing the shape-radiance ambiguity of NeRF in indoor scenes [S24]. Step 1: Run COLMAP on the test scene to acquire per-view sparse depth maps by projecting fused 3D point clouds after multi-view stereo. Step 2: Adapt a monocular depth network to the target scene by finetuning it on the sparse depth maps from COLMAP using a scale-invariant loss [S24].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S25]. Incorporate an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S25]. Density distribution: $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S25].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot N_{samples} \cdot D)$, where $N_{iters}$ is total iterations, $N_{rand}$ is batch size (rays), $N_{samples}$ is points per ray, and $D$ is network depth/width [S5, unofficial][S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot N_{samples} \cdot D)$ for the forward pass activations, plus $O(N_{iters})$ for optimizer states if not checkpointed [S5, unofficial].
*   **Measured Performance:** In standard Blender scenes, training takes ~100,000 iterations [S11, unofficial]. With batch size 1024, this can take several hours on a modern GPU [S17, unofficial].
*   **Scaling:** Scales linearly with batch size up to VRAM limits. Larger batches (e.g., $2^{14}$) improve throughput but may hinder data loading performance [S15, unofficial].
*   **Strengths:** End-to-end differentiable; handles complex lighting and geometry implicitly; high-quality novel view synthesis.
*   **Limitations:** Slow training convergence; prone to "floaters" in empty space; requires careful hyperparameter tuning (learning rate, decay) [S1][S22, unofficial].
*   **When to Use:** High-quality view synthesis from dense multi-view images.
*   **When Not to Use:** Real-time rendering requirements (use [Instant-NGP](link) or [3DGS](link)); sparse view settings without regularization ([Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)).

## Background
The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, unofficial]. It established the paradigm of optimizing a coordinate-based MLP to represent a 3D scene as a continuous volumetric function, trained via stochastic gradient descent on pixel-wise MSE loss [S5, unofficial][S14, unofficial]. Early implementations used Adam with exponential decay [S5, unofficial][S6, unofficial]. Subsequent work introduced coarse-fine sampling [S5, unofficial], positional encoding [S5, unofficial], and various regularization techniques to improve convergence and quality [S7, unofficial][S8].

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
[S20] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S21] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S22] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
[S23] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S24] Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo, https://doi.org/10.1109/tpami.2023.3263464, fetched 2026-10-27
[S25] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
