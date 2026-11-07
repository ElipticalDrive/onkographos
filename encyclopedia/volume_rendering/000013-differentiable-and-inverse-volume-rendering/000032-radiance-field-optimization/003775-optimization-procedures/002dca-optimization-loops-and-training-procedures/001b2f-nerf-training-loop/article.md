# NeRF Training Loop

Also called: End-to-End NeRF Training, HashNeRF Training Loop, HoloINR Training Loop, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric difference between rendered images and ground-truth observations by updating the parameters of a neural radiance field [S1][S5, S6, unofficial][S8][S10, S14, unofficial]. It solves the inverse problem of reconstructing a continuous 3D scene representation from a set of 2D images by repeatedly sampling rays, querying the network for density and color, integrating these values via volume rendering, and backpropagating the loss to update the weights [S5, S6, unofficial][S8]. It sets itself apart from evaluation methods by computing gradients for backpropagation and from static configuration methods by defining the dynamic runtime sequence of data loading, sampling, and gradient updates [S1][S2, unofficial][S8][S11, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of $N$ training images with known camera poses (origin $\mathbf{o}_i$, direction $\mathbf{d}_i$) and intrinsics; a neural network model $f_\theta$ (MLP or hash-based) parameterized by $\theta$; an optimizer (typically Adam) [S2, S5, S6, unofficial][S8][S10, S14, unofficial].
*   **Outputs:** Optimized network weights $\theta$; optionally, a saved checkpoint of the model state [S8][S10, S11, unofficial].

**Data Structures:**
*   **Rays Array:** Pre-computed array of shape $[(N-1) \times H \times W, 3, 3]$ storing flattened pixel indices, ray origins, ray directions, and target RGB colors [S5, S14, unofficial]. Axis 0 is the flattened pixel index; Axis 1 is the channel type (0: origin, 1: direction, 2: target RGB); Axis 2 is the XYZ/RGB components. Data type is `float32` [S5, S14, unofficial].
*   **Checkpoint:** A dictionary containing keys `'model'` (or `'coarse'`/`'fine'` for two-stage models) with the network state dictionary, `'optimizer'` with the optimizer state dictionary, and `'epoch'` or `'step'` (integer) [S10, S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S10, S11, unofficial].
    *   Initialize the neural network model $f_\theta$. For standard NeRF, this is an MLP; for HashNeRF, it involves initializing the hash grid and MLP layers [S7, S15, unofficial]. For growing networks, initialize alpha linear weights of grown sub-networks to match the parent to avoid instability [S1].
    *   Initialize the optimizer (e.g., Adam) with learning rate $\eta$ [S5, S6, unofficial][S8][S10, S13, S17, unofficial].
    *   Initialize the learning rate scheduler (e.g., ExponentialDecay, CosineAnnealing, or LambdaLR) [S5, unofficial][S8][S10, S11, unofficial].
    *   Precompute the rays array by stacking ray origins, directions, and target RGB colors from all training images, reshaping to a flat batch, and shuffling [S5, S14, unofficial].

2.  **Iteration Loop:**
    For each iteration $i$ from 1 to $N_{\text{iters}}$:
    a.  **Ray Sampling:**
        *   If batching is enabled, slice the pre-computed rays array from index $i_{\text{batch}}$ to $i_{\text{batch}} + N_{\text{rand}}$ [S5, S14, unofficial].
        *   Transpose the batch to shape $[3, B, 3]$ where axis 0 is channel type (origin, direction, target) [S5, S14, unofficial].
        *   Split into `batch_rays` (origins/directions) and `target_s` (colors) [S5, S14, unofficial].
        *   Increment $i_{\text{batch}}$. If $i_{\text{batch}}$ exceeds the total number of rays, shuffle the rays array and reset $i_{\text{batch}}$ to 0 [S5, S11, S14, unofficial].
        *   If batching is not enabled, randomly select $N_{\text{rand}}$ unique pixel coordinates from a randomly selected training image (optionally applying a central crop if $i < \text{precrop\_iters}$) [S14, unofficial].
    b.  **Forward Pass:**
        *   Pass `batch_rays` to the rendering function (e.g., `render_rays`) [S5, S6, S7, S11, S13, unofficial].
        *   The rendering function samples points along the rays, queries the network for density ($\sigma$) and color ($\mathbf{c}$), and integrates them using alpha compositing to produce predicted RGB $\hat{\mathbf{C}}$ [S1][S5, S6, unofficial].
        *   If a coarse-to-fine scheme is used, the coarse network output $\hat{\mathbf{C}}_0$ is also returned [S5, S11, unofficial].
        *   If chunking is used, process rays in chunks of size `chunk` to limit VRAM usage [S5, S7, unofficial].
    c.  **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) loss between predicted RGB $\hat{\mathbf{C}}$ and target colors $\mathbf{C}$:
            $$ \mathcal{L}_{\text{photo}} = \frac{1}{B} \sum_{j=1}^{B} (\hat{\mathbf{C}}_j - \mathbf{C}_j)^2 $$
            [S3, S4, S6, unofficial][S12][S13, S14, S17, unofficial].
        *   If coarse output $\hat{\mathbf{C}}_0$ is available, compute its MSE loss $\mathcal{L}_0$ and add it to the total loss: $\mathcal{L} = \mathcal{L}_{\text{photo}} + \mathcal{L}_0$ [S6, S7, S11, S18, unofficial].
        *   Optionally add regularization terms (e.g., sparsity, Total Variation, Eikonal) based on weights [S7, unofficial][S8][S16, unofficial].
    d.  **Backpropagation:**
        *   Zero the gradients of the optimizer [S3, S18, unofficial].
        *   Compute gradients: $\nabla_\theta \mathcal{L} = \frac{\partial \mathcal{L}}{\partial \theta}$ via automatic differentiation [S3, S13, S18, unofficial].
        *   Note: Higher-order derivative logging (e.g., gradients, Laplacians) must occur *before* `loss.backward()` as the backward pass frees the autograd graph [S3, unofficial].
    e.  **Update:**
        *   Step the optimizer: $\theta \leftarrow \theta - \eta \nabla_\theta \mathcal{L}$ [S3, S18, unofficial].
        *   Update the learning rate according to the scheduler [S5, unofficial][S8][S10, S11, S18, unofficial].
        *   For exponential decay: $\eta_{\text{new}} = \eta_{\text{old}} \times \text{decay\_rate}^{(\text{global\_step} / \text{decay\_steps})}$ [S5, S18, unofficial].
    f.  **Logging and Checkpointing:**
        *   Log loss and PSNR every `i_print` steps [S2, S3, S11, unofficial].
        *   Render validation images every `i_img` or `render_interval` steps [S2, unofficial][S8][S9, S11, unofficial].
        *   Save checkpoint every `i_weights` or `checkpoint_interval` steps [S2, unofficial][S8][S9, S11, unofficial].

**Parameters with Typical Values:**
*   $N_{\text{rand}}$ (Batch size): 1024 or 2048 rays per iteration [S5, S11, S14, unofficial].
*   Learning Rate ($\eta$): $5 \times 10^{-4}$ for Adam [S11, unofficial]; $0.01$ for SimpleDVGO [S17, unofficial].
*   Decay Rate: 0.1 [S6, S7, S18, unofficial].
*   Decay Steps: $\text{args.lrate\_decay} \times 1000$ [S5, S6, S7, S18, unofficial].
*   Total Iterations ($N_{\text{iters}}$): 100,000 [S11, unofficial]; 1,000,000 [S14, unofficial].
*   Density Grid Resolution: 128 [S15, unofficial].
*   Max Memory: 2500 MB [S15, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Mitigated by splitting ray batches into chunks via `chunk` argument in the render call [S5, S7, unofficial].
*   **Empty Checkpoint:** If a checkpoint is loaded but the state step is 0, it is considered empty; log an error and return [S15, unofficial].
*   **Missing Checkpoint:** If the specified checkpoint path does not exist, log an error and return [S15, unofficial].
*   **Instability in Growing Networks:** Direct growth of randomly initialized networks causes density to drop to 0; initialize sub-network weights from the parent [S1].
*   **Gradient Graph Freeing:** Computing logging metrics that require gradients (e.g., Laplacian) must happen before `loss.backward()` [S3, unofficial].
*   **Learning Rate Drop:** Stop training if the optimizer learning rate drops below $10^{-7}$ [S16, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader

def train_nerf_loop(model, dataset, config):
    """
    Standard NeRF training loop.
    
    Args:
        model: Neural Radiance Field model (MLP or HashNeRF).
        dataset: Dataset yielding (images, poses) or pre-computed rays.
        config: Configuration object containing hyperparameters.
    """
    # 1. Initialization
    optimizer = optim.Adam(model.parameters(), lr=config.lr)
    
    # Learning rate scheduler (Exponential Decay example)
    if config.lrate_decay > 0:
        scheduler = optim.lr_scheduler.LambdaLR(
            optimizer, 
            lr_lambda=lambda step: config.lr * (config.decay_rate ** (step / (config.lrate_decay * 1000)))
        )
    else:
        scheduler = None
        
    # Precompute rays if not using dynamic loading
    if config.use_batching:
        rays_rgb = precompute_rays(dataset) # Shape: [N_pixels, 3, 3]
        i_batch = 0
        shuffle_rays(rays_rgb)
        
    mse_loss = nn.MSELoss()
    n_iters = config.n_iters
    batch_size = config.batch_size
    
    for step in range(n_iters):
        # 2. Ray Sampling
        if config.use_batching:
            # Slice batch from pre-computed array
            end_idx = i_batch + batch_size
            if end_idx > rays_rgb.shape[0]:
                shuffle_rays(rays_rgb)
                i_batch = 0
                end_idx = batch_size
                
            batch_data = rays_rgb[i_batch:end_idx] # [B, 3, 3]
            batch_rays = batch_data[:, :2, :]      # [B, 2, 3] (origins, dirs)
            target_s = batch_data[:, 2, :]         # [B, 3] (RGB)
            i_batch = end_idx
        else:
            # Dynamic sampling from dataset
            batch_rays, target_s = sample_rays_from_dataset(dataset, batch_size)
            
        # 3. Forward Pass
        optimizer.zero_grad()
        
        # Render rays
        # Returns predicted RGB and optionally coarse RGB if coarse-to-fine is used
        output = model.render(batch_rays) 
        pred_rgb = output['rgb']
        
        # 4. Loss Computation
        loss = mse_loss(pred_rgb, target_s)
        
        # Add coarse loss if available
        if 'rgb_coarse' in output:
            loss += mse_loss(output['rgb_coarse'], target_s)
            
        # Add regularization if configured
        if config.has_regularization:
            loss += compute_regularization(model, output)
            
        # 5. Backpropagation
        loss.backward()
        
        # 6. Update
        optimizer.step()
        if scheduler is not None:
            scheduler.step()
            
        # 7. Logging/Checkpointing (pseudo-code)
        if step % config.log_interval == 0:
            log_metrics(step, loss.item())
        if step % config.checkpoint_interval == 0:
            save_checkpoint(model, optimizer, step)
            
    return model
```

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images from a neural radiance field for gradient-based optimization. It solves the problem of efficiently sampling 3D space along camera rays, querying a neural network for density and color, and integrating these values via alpha compositing to produce a loss-computable image. It sets itself apart from [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md) by computing gradients for backpropagation and from [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) by including the adaptive sampling and occupancy grid pruning steps specific to the training loop.

### [Training Configuration](2332-training-configuration.md)
Defines the static hyperparameters, optimizer settings, and structural constraints required to stabilize the optimization of differentiable volume rendering models. It solves the problem of translating theoretical radiance field formulations into convergent training procedures by specifying exact network architectures, sampling strategies, and loss weights. Unlike [Training Loop Setup](1522-training-loop-setup.md), which defines the dynamic runtime sequence of data loading and gradient updates, this method specifies the immutable parameters governing the optimization landscape. It differs from [Regularization and Priors](6281-regularization-and-priors.md) by focusing on the numerical optimization settings (e.g., learning rates, batch sizes) rather than the specific penalty terms added to the loss function.

### [Mip-NeRF 360](3306-mip-nerf-360.md)
A method for synthesizing photorealistic views of unbounded, real-world scenes where the camera may rotate 360 degrees and content exists at arbitrary distances. It solves the problems of aliasing, training instability, and geometric ambiguity inherent in standard NeRF when applied to large-scale environments. It sets itself apart from its parent [NeRF Training Loop](970-nerf-training-loop.md) by integrating a non-linear scene parameterization, online distillation between two MLPs, and a distortion-based regularizer, achieving a 57% reduction in mean squared error compared to standard mip-NeRF.

### [Mip-NeRF 360 Sampling and Training](3393-mip-nerf-360-sampling-and-training.md)
Extends the Mip-NeRF framework to unbounded, 360-degree scenes by introducing a disparity-based ray parameterization, a contraction function for coordinate space, and a coarse-to-fine training loop with a proposal network. It solves the artifacts of "floaters" and "background collapse" common in unbounded scene reconstruction by regularizing the density distribution and ensuring efficient sampling through online distillation. It differs from standard NeRF by using linear-in-disparity sampling and from basic Mip-NeRF by handling unbounded scenes via coordinate contraction and disparity parameterization.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [MVSNeRF](6197-mvsnerf.md)
A method for fast, generalizable radiance field reconstruction from multi-view stereo (MVS) images. Unlike standard NeRF which requires per-scene optimization, MVSNeRF uses a feedforward neural network to predict the radiance field directly from input images, significantly reducing training time and iteration counts. It sets itself apart by learning a data prior that accelerates convergence, enabling rapid inference without iterative scene-specific optimization.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
Addresses the problem of optimizing NeRF scene representations when input camera poses are inaccurate, unknown, or when intrinsic parameters are not provided. Standard NeRF training assumes perfect ground-truth camera poses; deviations cause the optimization to converge to incorrect geometry or fail entirely. These methods solve this by treating camera parameters as trainable variables within a fully differentiable pipeline, jointly minimizing the reconstruction loss with respect to both the NeRF weights and the camera poses. They are distinguished from standard training loops by the inclusion of pose update steps and specific strategies (such as coarse-to-fine scheduling or external priors) to stabilize the non-convex joint optimization landscape.

### [Specific Model Architectures](8585-specific-model-architectures.md)
Refers to the family of neural network structures and representations used to parameterize the radiance field $f_\theta(\mathbf{x}, \mathbf{d})$ within the NeRF training loop. While the standard NeRF uses a simple MLP mapping 3D coordinates and viewing directions to density and color, this family encompasses modifications to the network topology, input representations, and output mechanisms to address limitations such as training speed, generalization, memory efficiency, and handling of complex phenomena like lighting changes or reflections. These architectures set themselves apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the *function* $f_\theta$ itself rather than the *sampling* strategy or *loss* functions.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Encompasses loss functions and regularization techniques applied during the optimization of Neural Radiance Fields (NeRFs) to mitigate the ill-posed nature of inverse volume rendering, particularly in sparse-view or unbounded settings. While standard NeRF training relies solely on photometric consistency ($\mathcal{L}_{\text{photo}}$), these methods augment the objective with geometric, statistical, or diffusion-based priors to suppress artifacts such as "floaters," overfitting, and view-dependent hallucinations. They set themselves apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) (which focuses on sampling efficiency) and [Training Configuration](2332-training-configuration.md) (which focuses on hyperparameter schedules) by directly modifying the gradient signal to enforce structural plausibility.

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
A training strategy for single-view NeRF that avoids geometry conflicts between visible and invisible regions by computing a mask of conflicted regions and eliminating them from NeRF training. It sets itself apart from standard single-view NeRF methods by integrating a collaborative RGBD inpainting step with a progressive erasing mechanism to handle the ambiguity of unseen geometry, building on the [NeRF Training Loop](970-nerf-training-loop.md) by dynamically expanding the reference view set.

### [NeRF-FF](8669-nerf-ff.md)
A plug-in strategy that mitigates the degradation of NeRF reconstruction quality caused by out-of-focus (defocus blurred) regions in input images. It solves the problem of training NeRFs on partially blurred data by estimating binary image masks that identify in-focus regions, allowing the subsequent NeRF model to omit out-of-focus pixels during training. Unlike methods that use dynamic ray generation or blur kernels, NeRF-FF relies on static ray assumptions, making it compatible with runtime-optimized NeRF variants like Instant-NGP, nerfacto, and splatfacto.

### [KNN-NeRF](8672-knn-nerf.md)
A baseline method for training NeRFs using sparse 3D point clouds as supervision instead of multi-view images. It solves the problem of generating ground-truth color labels for rays that do not pass through existing points by assigning the color of the nearest point in the cloud to the ray. Unlike standard NeRF which relies on pixel colors from images, KNN-NeRF constructs a dense training signal from sparse geometry, though it struggles with sharp object borders due to the low-frequency bias of neural networks and the sparsity of the input.

### [Hard Sample Mining](8673-hard-sample-mining.md)
An optimization procedure for neural radiance fields that improves training efficiency and reconstruction quality by selectively updating the network based on pixel-level error. It solves the problem of uniform computational waste in standard NeRF training, where samples contributing little to the gradient are treated equally to those with high error. Unlike sibling methods such as [Probability-based Ray Sampling with Quadtree Subdivision](7481-probability-based-ray-sampling-with-quadtree-subdivision.md) or [Nerfacc](https://github.com/NVlabs/nerfacc) which prune low-weight samples based on density or transmittance, HSM ranks samples using the propagated pixel loss to subsample a "hard" minibatch that retains most of the update information.

### [NeRF-ID](8867-nerf-id.md)
A differentiable volume rendering optimization method that replaces the heuristic coarse-to-fine sampling procedure of vanilla NeRF with a learnable "proposer" module. It solves the problem of suboptimal sample placement by enabling end-to-end training of the coarse network, proposer, and fine network jointly, thereby improving view synthesis quality. Unlike its sibling [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) which relies on handcrafted probability density functions derived from coarse densities, NeRF-ID uses a neural network to directly predict fine sample locations from coarse features.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Addresses the challenge of reconstructing large-scale or complex 3D scenes from limited input data by incrementally expanding the neural representation's capacity or the training dataset. Unlike static NeRF architectures that require pre-defining the entire hierarchy or fixed network size, these methods adaptively grow the model structure or add synthetic reference views during training to resolve occlusions and capture high-frequency details. This family distinguishes itself from sibling methods like [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) (which focuses on ray selection) and [Sparse NeRF Reconstruction with Geometric and Semantic Initialization](8779-sparse-nerf-reconstruction-with-geometric-and-semantic-initi.md) (which relies on external priors) by actively modifying the model's topology or the set of supervision signals to improve reconstruction quality in complex regions.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Refers to a family of strategies in neural radiance field (NeRF) training that dynamically adjust the distribution of sample points along viewing rays to improve reconstruction quality and training efficiency. Unlike uniform sampling, which treats all regions of a ray equally, these methods allocate computational resources to areas of high geometric or photometric complexity, such as object boundaries or high-frequency details. This family addresses the inefficiency of uniform sampling in sparse or complex scenes by clustering samples where they contribute most to the final rendered image.

### [Drone-NeRF](8972-drone-nerf.md)
A method for large-scale 3D scene reconstruction from drone-captured imagery that addresses the specific challenges of overhead drone views, including exposure variations, shadow artifacts from vertical surfaces, and computational efficiency in large scenes. It differs from standard NeRF approaches by employing a block-parallel training strategy where sub-scenes are trained independently and then merged, while explicitly handling exposure discrepancies via appearance embeddings and removing occlusion shadows through an expanded bounding box projection technique. Unlike methods that treat the entire scene as a single volume, Drone-NeRF partitions the space into sub-bounding boxes, optimizing them separately before a final merging phase that ensures color uniformity and depth continuity across block boundaries.

### [Two-Stage Sampling with Occupancy Grid and Nonlinear Function](9146-two-stage-sampling-with-occupancy-grid-and-nonlinear-functio.md)
A sampling strategy for neural implicit surface reconstruction that combines an Occupancy Grid to skip empty regions during a coarse stage and a nonlinear piecewise exponential function to guide sampling points closer to the object surface within occupied regions during a fine stage. It solves the problem of inefficient sampling in neural radiance fields by reducing training time and sample count while maintaining high-quality surface reconstruction. Unlike NeuS, which uses uniform sampling followed by inverse transform sampling that scatters points randomly near the surface, and Instant-NGP, which uses uniform sampling within occupied grid cells, this method uses a nonlinear function to bias samples towards the surface within the occupied regions.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
Addresses the degradation of NeRF quality—such as blurred geometry, slow convergence, and floaters—when trained on sparse input views (typically 15–20 images or fewer than 50–100 views). It distinguishes itself from standard NeRF training by decoupling the initialization and regularization processes from the default random or empty-space assumptions, instead leveraging external geometric scaffolds, semantic masks, or visibility priors to anchor the optimization.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)

### ReconFusion
Uses a Diffusion Prior to constrain unobserved views [S19, unofficial].

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S20]. For every pixel, cast multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel. Render each ray with NeRF. Use the mean of those rendered values as the predicted color in the loss function [S20]. The full-resolution supersampling baseline relies on 'oracle' scale information that does not exist for most datasets [S20]. Performance: The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S20].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S21, unofficial]. Phase 1: Train the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S21, unofficial]. Phase 2: Train all network components end-to-end. At the start of this phase, reset the optimizer state and perform a linear warmup for 1,000 steps to reduce shock from switching regimes [S21, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S21, unofficial].

### Multi-view SDS
A plug-and-play method that creates multi-view supervision during SDS optimization to address the Janus problem caused by one-by-one view sampling [S22]. Rendered images from 4 sampled views are stitched into a large 2x2 image. Gradients are calculated under the condition of the multi-view normal map. The method converts a 3D asset into differentiable representations including NeRF, 3DGS, and DMTet through rendering supervision. The representations are then optimized to improve the quality of the 3D asset [S22].

## Operating characteristics
*   **Time Complexity:** $O(N_{\text{iters}} \times N_{\text{rand}} \times N_{\text{samples}})$, where $N_{\text{iters}}$ is the number of training steps, $N_{\text{rand}}$ is the batch size (rays per step), and $N_{\text{samples}}$ is the number of samples per ray [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{\text{rand}} \times N_{\text{samples}} \times \text{model\_size})$ for the forward pass, plus $O(N_{\text{pixels}})$ if pre-computing all rays [S5, S14, unofficial]. VRAM usage is mitigated by chunking rays [S5, S7, unofficial].
*   **Measured Performance:** Training on 128x128 images for 250 epochs [S4, unofficial]. On a dataset of 199 images, each epoch processes ~8M rays; with batch size 1024, this results in ~7,800 iterations per epoch [S17, unofficial].
*   **Scaling:** Larger batch sizes (e.g., $2^{14}$) hinder data loading performance and prevent full GPU occupancy, while smaller batch sizes (e.g., $2^{10}$) cap iterations per second, reducing throughput [S15, unofficial].
*   **Strengths:** Stochastic optimization reduces compute per iteration compared to full-image rendering [S5, unofficial]. Pre-computing rays allows efficient batching [S5, S14, unofficial].
*   **Limitations:** Rendering quality is limited by coarse voxel grid resolution and lack of advanced ray sampling in basic implementations, resulting in blurry images [S17, unofficial].
*   **When to Use:** When reconstructing a single scene from a set of images with known poses.
*   **When Not to Use:** When camera poses are unknown (use [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)) or when generalizing to new scenes without per-scene optimization (use [MVSNeRF](6197-mvsnerf.md)).

## Background
The NeRF Training Loop was introduced in the seminal paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" by Mildenhall et al. (2020) [S5, S18, unofficial]. It builds on the concept of coordinate-based neural networks for continuous scene representation [S3, unofficial]. The method has since been extended to handle unbounded scenes (Mip-NeRF 360), sparse views, and differentiable rendering in various frameworks [S1][S11, S15, unofficial]. It is widely used in computer vision and graphics for novel view synthesis, 3D reconstruction, and augmented reality [S1][S11, unofficial].

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
