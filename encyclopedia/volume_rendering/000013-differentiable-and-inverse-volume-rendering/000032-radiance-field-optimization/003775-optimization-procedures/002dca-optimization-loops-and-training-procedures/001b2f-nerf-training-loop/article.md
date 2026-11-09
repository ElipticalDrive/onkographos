# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric reconstruction error between rendered images and ground-truth observations by updating the parameters of a coordinate-based neural radiance field [S5, S6, S14, unofficial]. It solves the inverse rendering problem by treating the scene representation as a differentiable function $f_\theta(\mathbf{x}, \mathbf{d})$ that maps 3D coordinates and viewing directions to volume density and color [S1][S5, unofficial]. It sets itself apart from its siblings by defining the core stochastic gradient descent cycle—ray sampling, forward volume rendering, loss computation, and weight update—that serves as the foundation for all specialized variants like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) and [Training Configuration](2332-training-configuration.md).

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of $N$ training images with known camera poses (extrinsics $C$ and intrinsics $K$) [S5, S11, unofficial]. The neural network parameters $\theta$ (weights and biases of the MLP) [S5, unofficial].
*   **Outputs:** Updated network parameters $\theta$ that minimize the difference between rendered and observed pixel colors [S5, unofficial].

**Data Structures:**
*   **Rays:** A batch of rays is represented as a tensor of shape $[B, 3, 3]$ or $[B, 6]$ depending on implementation, where $B$ is the batch size (number of rays) [S5, S14, unofficial].
    *   Axis 0 (or first 3 elements): Ray origins $\mathbf{o}$ (xyz).
    *   Axis 1 (or next 3 elements): Ray directions $\mathbf{d}$ (xyz), typically normalized [S5, S14, unofficial].
    *   Axis 2 (or last 3 elements): Target RGB colors $\mathbf{c}_{gt}$ [S5, S14, unofficial].
*   **Pre-computed Rays Array:** To avoid repeated ray generation, a flat array `rays_rgb` of shape $[(N-1) \cdot H \cdot W, 3, 3]$ is often pre-computed, where $N$ is the number of training images, and $H, W$ are image dimensions [S5, S14, unofficial]. This allows random sampling of any pixel from any training image [S5, unofficial].
*   **Checkpoint:** A dictionary containing `model` (state dict), `optimizer` (state dict), and `epoch` or `step` counter [S10, S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the neural network (MLP) with random weights [S5, S15, unofficial]. For progressive growth methods, initialize alpha linear weights of grown sub-networks to match the parent to avoid instability [S1].
    *   Initialize the optimizer (typically Adam) with a learning rate $lr$ [S5, S11, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   Pre-compute the `rays_rgb` array by iterating over all training images, generating rays for every pixel, and storing origins, directions, and target RGB values [S5, S14, unofficial]. Shuffle this array [S5, S11, unofficial].

2.  **Iteration Loop:**
    For each iteration $i$ from 1 to $N_{iters}$:
    a.  **Ray Sampling:**
        *   If batching is enabled, slice a batch of $N_{rand}$ rays from the `rays_rgb` array using the current batch index [S5, S14, unofficial].
        *   Transpose the batch to shape $[3, N_{rand}, 3]$ (origin, direction, target) [S5, S14, unofficial].
        *   If the batch index exceeds the array length, shuffle the `rays_rgb` array and reset the index [S5, S11, S14, unofficial].
        *   *Optional:* Apply perturbation to sample positions if `perturb > 0` to break symmetry and improve reconstruction [S5, unofficial].
        *   *Optional:* Apply precropping for the first $P$ iterations to focus on central image details [S5, unofficial].

    b.  **Forward Pass (Rendering):**
        *   For each ray in the batch, sample $K$ points along the ray segment $[t_{near}, t_{far}]$ [S5, unofficial].
        *   Query the neural network $f_\theta$ at each point $\mathbf{x}$ and direction $\mathbf{d}$ to obtain raw output (density $\sigma$ and color $\mathbf{c}$) [S5, unofficial].
        *   Compute transmittance $T_i$ and weights $w_i$ using the emission-absorption model [S5, S18, unofficial].
        *   Composite the final color $\hat{\mathbf{C}}$ for each ray [S5, unofficial].
        *   *Optional:* If using a coarse-fine two-stage model, render with the coarse network, sample additional points based on coarse weights, and render with the fine network [S5, S11, unofficial].

    c.  **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) between the predicted color $\hat{\mathbf{C}}$ and the target color $\mathbf{c}_{gt}$ [S5, S14, unofficial]:
            $$ \mathcal{L}_{MSE} = \frac{1}{B} \sum_{j=1}^{B} \| \hat{\mathbf{C}}_j - \mathbf{c}_{gt, j} \|^2 $$
        *   *Optional:* If a coarse network was used, add the coarse loss $\mathcal{L}_{coarse}$ to the total loss [S6, S11, S18, unofficial].
        *   *Optional:* Add regularization terms (e.g., sparsity, Total Variation, Eikonal) weighted by hyperparameters [S7, unofficial][S8].
        *   Total Loss: $\mathcal{L} = \mathcal{L}_{MSE} + \lambda_{coarse}\mathcal{L}_{coarse} + \dots$

    d.  **Backpropagation and Update:**
        *   Zero the gradients of the optimizer [S18, unofficial].
        *   Compute gradients: $\nabla_\theta \mathcal{L}$ [S18, unofficial].
        *   *Pitfall:* If logging higher-order derivatives (gradient, Laplacian) is required, compute them *before* calling `backward()` [S3, unofficial].
        *   Step the optimizer: $\theta \leftarrow \theta - \eta \nabla_\theta \mathcal{L}$ [S18, unofficial].
        *   Update the learning rate according to the scheduler [S5, S18, unofficial].

    e.  **Logging and Checkpointing:**
        *   Log loss and PSNR every $I_{print}$ steps [S3, S11, unofficial].
        *   Render validation images every $I_{img}$ steps [S2, S11, unofficial].
        *   Save model checkpoint every $I_{weights}$ steps [S2, S11, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim
import numpy as np

def train_nerf_loop(model, rays_rgb, optimizer, scheduler, n_iters, batch_size, render_fn, log_interval=100, save_interval=1000):
    """
    Standard NeRF training loop.
    
    Args:
        model: Neural Radiance Field MLP.
        rays_rgb: Pre-computed array of shape [N_pixels, 3, 3] (origin, dir, target_rgb).
        optimizer: PyTorch optimizer (e.g., Adam).
        scheduler: Learning rate scheduler.
        n_iters: Total number of training iterations.
        batch_size: Number of rays per batch (N_rand).
        render_fn: Function that takes rays and model, returns rendered RGB.
        log_interval: Steps between logging.
        save_interval: Steps between saving checkpoints.
    """
    # Initialize batch index
    batch_idx = 0
    n_pixels = rays_rgb.shape[0]
    
    # Convert rays_rgb to tensor for efficient slicing if using PyTorch
    # Assuming rays_rgb is float32
    rays_tensor = torch.from_numpy(rays_rgb).float()
    
    for i in range(n_iters):
        # 1. Ray Sampling
        # Slice batch
        end_idx = batch_idx + batch_size
        if end_idx > n_pixels:
            # Shuffle and reset if exhausted
            indices = torch.randperm(n_pixels)
            rays_tensor = rays_tensor[indices]
            batch_idx = 0
            end_idx = batch_idx + batch_size
            
        batch_data = rays_tensor[batch_idx:end_idx] # Shape: [batch_size, 3, 3]
        batch_idx += batch_size
        
        # Split into rays (origin, dir) and targets
        # batch_data[:, 0, :] -> origins
        # batch_data[:, 1, :] -> directions
        # batch_data[:, 2, :] -> target_rgb
        rays_o = batch_data[:, 0, :]
        rays_d = batch_data[:, 1, :]
        target_rgb = batch_data[:, 2, :]
        
        # 2. Forward Pass
        # Pack rays for render_fn (implementation specific, e.g., [N, 6] or [N, 3, 3])
        # Here we assume render_fn accepts origins and directions separately or a combined tensor
        # and returns predicted RGB of shape [batch_size, 3]
        predicted_rgb = render_fn(model, rays_o, rays_d)
        
        # 3. Loss Computation
        # MSE Loss
        loss = nn.functional.mse_loss(predicted_rgb, target_rgb)
        
        # Optional: Add coarse loss if model returns it
        # if 'rgb_coarse' in extras:
        #     loss += nn.functional.mse_loss(extras['rgb_coarse'], target_rgb)
        
        # 4. Backpropagation
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # 5. Scheduler Update
        scheduler.step()
        
        # 6. Logging and Checkpointing
        if i % log_interval == 0:
            psnr = -10.0 * torch.log10(loss)
            print(f"Iter {i}: Loss {loss.item():.6f}, PSNR {psnr.item():.2f}")
            
        if i % save_interval == 0:
            torch.save({
                'step': i,
                'model_state_dict': model.state_dict(),
                'optimizer_state_dict': optimizer.state_dict(),
                'scheduler_state_dict': scheduler.state_dict()
            }, f"checkpoint_{i}.pth")

```

**Parameters with Typical Values:**
*   $N_{rand}$ (Batch Size): 1024 or 2048 rays per iteration [S5, S11, unofficial].
*   Learning Rate ($lr$): $5 \times 10^{-4}$ for Adam [S11, unofficial].
*   Decay: Exponential decay with rate 0.1 every 1000 steps (or $args.lrate\_decay \times 1000$) [S5, S18, unofficial].
*   Positional Encoding Frequencies: 10 for position, 4 for direction [S11, unofficial].
*   Total Iterations ($N_{iters}$): 100,000 to 1,000,000 [S11, S14, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering all pixels at once exceeds GPU memory. Mitigated by chunking rays or using small batch sizes ($N_{rand}$) [S5, S15, unofficial].
*   **Gradient Freeing:** Higher-order derivatives (for logging gradients/Laplacians) must be computed before `loss.backward()` [S3, unofficial].
*   **Empty Checkpoints:** Loading a checkpoint with step 0 is an error condition [S15, unofficial].
*   **Scheduler Resumption:** When resuming, the scheduler's last epoch must be set to `start_epoch - 1` to maintain correct decay [S9, unofficial].
*   **Instability in Growth:** Randomly initialized sub-networks in growing NeRFs can collapse to zero density; weights should be inherited from the parent [S1].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images from a neural radiance field (NeRF) during optimization. It solves the problem of computing gradients for scene parameters by sampling points along camera rays, querying a neural network for volume density and color, and integrating them via alpha compositing. It sets itself apart from evaluation by using stochastic perturbations, adaptive sampling strategies (e.g., occupancy grid pruning), and chunked processing to manage memory, whereas evaluation disables these for deterministic results.

### [Training Configuration](2332-training-configuration.md)
Training Configuration defines the hyperparameters, optimization routines, and architectural constraints required to stabilize and accelerate the differentiable volume rendering optimization process. It addresses the sensitivity of radiance field training to initialization, sampling density, and activation function choices. Unlike general optimization loops, it specifies implementation-specific details such as learning rate schedules, loss weights, and batch sampling strategies that are critical for convergence in methods like mip-NeRF, Instant-NGP, and NeuralVDB.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
Camera Pose and Parameter Optimization refers to the family of methods that jointly optimize the parameters of a Neural Radiance Field (NeRF) and the camera poses (extrinsics) and/or intrinsics (e.g., focal length) from a set of input images. It solves the problem of reconstructing 3D scenes when camera parameters are unknown, inaccurate, or when the system requires online pose estimation for navigation. Unlike baseline NeRF, which relies on pre-estimated poses from Structure-from-Motion (SfM) pipelines like COLMAP, these methods treat camera parameters as learnable variables updated via gradient descent alongside the NeRF weights. This family encompasses static scene reconstruction with unknown poses, hybrid 2D/3D registration, and online robot navigation filters.

### [Specific Model Architectures](8585-specific-model-architectures.md)
This family encompasses specific neural network architectures and structural modifications applied to the Neural Radiance Field (NeRF) representation to address limitations in rendering speed, geometric accuracy, generalization, and semantic understanding. While the parent [NeRF Training Loop](970-nerf-training-loop.md) defines the optimization procedure, these architectures define the function $f_\theta(\mathbf{x}, \mathbf{d})$ itself. They are distinguished from siblings like [AdaNeRF](6195-adanerf.md) (adaptive sampling) and [Zip-NeRF](9538-zip-nerf-training-procedure.md) (hash grids) by their focus on the internal network topology, input/output mappings, and auxiliary modules (e.g., discriminators, diffusion priors, or geometric heads) rather than just the sampling strategy or data structure.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
This family encompasses the mathematical terms added to the standard photometric reconstruction loss to constrain the optimization of Neural Radiance Fields (NeRFs), addressing issues such as under-constrained geometry, floaters, and color artifacts in sparse or few-view settings. It distinguishes itself from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the objective function's structure rather than the sampling strategy, and from [Training Configuration](2332-training-configuration.md) by defining the specific gradients that drive parameter updates.

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
Progressive Inpainting-and-Erasing (PERF) is a training strategy for single-view NeRF that avoids geometry conflicts between visible and invisible regions by computing a mask of conflicted regions and eliminating them from NeRF training. It sets itself apart from standard single-view NeRF methods by integrating a collaborative RGBD inpainting step with a progressive erasing mechanism to handle the ambiguity of unseen geometry, building on the [NeRF Training Loop](970-nerf-training-loop.md) by dynamically expanding the reference view set.

### [KNN-NeRF](8672-knn-nerf.md)
KNN-NeRF is a baseline method for training Neural Radiance Fields (NeRFs) using sparse 3D point clouds as supervision instead of multi-view images. It solves the problem of generating ground-truth color labels for rays that do not pass through existing points by assigning the color of the nearest point in the cloud to the ray. Unlike standard NeRF which relies on pixel colors from images, KNN-NeRF constructs a dense training signal from sparse geometry, though it struggles with sharp object borders due to the low-frequency bias of neural networks and the sparsity of the input.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Progressive and Dynamic Scene Growth addresses the challenge of reconstructing large-scale or complex 3D scenes from limited input data by incrementally expanding the neural representation's capacity or the training dataset. Unlike static NeRF architectures that require pre-defining the entire hierarchy or fixed network size, these methods adaptively grow the model structure or add synthetic reference views during training to resolve occlusions and capture high-frequency details. This family distinguishes itself from sibling methods like [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) (which focuses on ray selection) and [Sparse NeRF Reconstruction with Geometric and Semantic Initialization](8779-sparse-nerf-reconstruction-with-geometric-and-semantic-initi.md) (which relies on external priors) by actively modifying the model's topology or the set of supervision signals to improve reconstruction quality in complex regions.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Adaptive ray sampling refers to a family of optimization procedures in neural radiance field (NeRF) training that dynamically determine the spatial distribution of sample points along viewing rays to improve reconstruction quality and training efficiency. Unlike uniform sampling, which treats all regions of a ray equally, these methods allocate computational resources to areas of high geometric or photometric complexity, such as object surfaces or textured regions. This family is distinguished from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by its focus on the *selection* of sample locations based on learned or heuristic priors, rather than the integration technique itself.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
This family addresses the degradation of NeRF quality—such as blurred geometry, slow convergence, and floaters—when trained on sparse input views (typically 15–20 images or fewer than 50–100 views). It distinguishes itself from standard NeRF training by decoupling the initialization and regularization processes from the default random or empty-space assumptions, instead leveraging external geometric scaffolds, semantic masks, or visibility priors to anchor the optimization.

### [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md)
Scene Representation and Partitioning addresses the scalability limits of global NeRF models, which suffer from high memory consumption, slow training convergence, and rendering artifacts when applied to large-scale environments or dynamic scenes. By decomposing the 3D spatial domain into disjoint regions (blocks, tiles, or cells), each managed by a dedicated local neural network or data structure, these methods enable distributed training, faster inference, and the handling of scenes spanning thousands of square meters. Unlike global approaches, partitioning allows for independent optimization of local regions, though it introduces challenges regarding boundary continuity and temporal consistency in dynamic settings.

### [Zip-NeRF Training Procedure](9538-zip-nerf-training-procedure.md)
The Zip-NeRF Training Procedure is a two-stage optimization method for inverse rendering that jointly trains a physically-based scene model, environment map, and radiance caches (Zip-NeRF and Fast Cache) to estimate incoming radiance. It solves the bias introduced by approximating incoming radiance using neural caches by employing a control variate strategy and a specific gradient trick for the photometric loss. It sets itself apart from standard NeRF training by decoupling the cache initialization from the joint optimization of the physical model and by using an occlusion-aware importance sampler to reduce variance in the radiance estimation.

### [NeRV Training](10253-nerv-training.md)
NeRV training optimizes a continuous volumetric representation that decouples scene geometry (shape) from material appearance (reflectance) and lighting interactions (visibility) to enable efficient relighting and view synthesis. It differs from standard NeRF training by introducing a separate "visibility MLP" that predicts ray visibility and termination depth, supervised by consistency terms with the primary shape MLP rather than ground-truth labels, allowing the simulation of direct and one-bounce indirect illumination within the training loop.

### [NeRV Rendering](10264-nerv-rendering.md)
NeRV rendering computes the pixel color for a camera ray by integrating reflected radiance along the ray, where the radiance at each sample point is derived from a physically-based shading model that accounts for direct and indirect illumination. It differs from standard NeRF volume rendering by explicitly modeling surface normals and microfacet BRDF parameters via separate MLPs, and by estimating indirect lighting through secondary visibility queries rather than simple volumetric absorption.

### [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md)
This family defines specific neural network architectures and configuration parameters for the density and texture (color) networks within Neural Scene Vector Fields (NSVF) and related radiance field models. It addresses the problem of how to structure the input features (position, direction, embeddings) and the network topology to optimize reconstruction quality and training stability. It sets itself apart from general training loops by focusing on the internal MLP structure and feature concatenation strategies rather than the optimization schedule or ray sampling strategy.

### [Aug-NeRF](10294-aug-nerf.md)
Aug-NeRF is a training framework for Neural Radiance Fields (NeRF) that injects worst-case adversarial perturbations into three distinct levels of the pipeline—input coordinates, intermediate features, and pre-rendering output—to implicitly regularize the model with physical grounds. It addresses the generalization gap in NeRF where interpolation between seen views yields inconsistent and non-smooth geometries. Unlike explicit regularizers like Laplacian or Total Variation which constrain MLP flexibility aggressively, Aug-NeRF uses data-adaptive smoothness priors to enhance generalization to unseen views without sacrificing representation flexibility.

### [Cross-Guided Optimization of Radiance Fields (CROP)](10371-cross-guided-optimization-of-radiance-fields-crop.md)
Cross-Guided Optimization of Radiance Fields (CROP) is a framework that performs multi-view image super-resolution (MVSR) on training-view images during the radiance field optimization process to improve high-resolution novel view synthesis (HRNVS). It fuses single-image super-resolution (SISR) features with voxel-based uncertainty fields to ensure multi-view consistency and high-frequency details. Unlike methods that rely solely on the spectral characteristics of coordinate-based networks for upscaling, CROP explicitly bridges 2D SISR and 5D radiance fields by iteratively refining input images based on rendering errors.

### [Application-Specific Pipelines](11200-application-specific-pipelines.md)
Application-Specific Pipelines are specialized training and optimization procedures designed to address distinct downstream tasks—such as rapid reconstruction, pose estimation, or scene editing—rather than generic radiance field optimization. They diverge from standard NeRF training by integrating external priors (e.g., multi-view stereo data, diffusion models, or dynamics models) and modifying the loss landscape or rendering loop to suit specific constraints like speed, geometric accuracy, or semantic control.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, multiple jittered rays are sampled uniformly at random through the spatial footprint of the pixel, rendered with NeRF, and averaged to predict the color in the loss function [S21]. This approach relies on 'oracle' scale information that does not exist for most datasets and performs worse than mip-NeRF while being significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 SGD steps to reduce shock from switching regimes [S22, unofficial]. This two-stage approach is consistently effective where training from scratch often fails [S22, unofficial].

### Multi-view SDS
A plug-and-play method that creates multi-view supervision during SDS optimization to address the Janus problem caused by one-by-one view sampling [S23]. Rendered images from 4 sampled views are stitched into a large 2x2 image, and gradients are calculated under the condition of the multi-view normal map [S23]. The method converts a 3D asset into differentiable representations including NeRF, 3DGS, and DMTet through rendering supervision, which are then optimized to improve the quality of the 3D asset [S23].

### Guided NeRF Optimization for MVS
A multi-view depth estimation method that utilizes conventional reconstruction (sparse SfM+MVS) and learning-based priors over Neural Radiance Fields (NeRF) [S24]. It performs test-time optimization directly over implicit volumes to eliminate the need for accurate pixel matching across views, addressing the shape-radiance ambiguity of NeRF in indoor scenes [S24]. Step 1 runs COLMAP to acquire per-view sparse depth maps by projecting fused 3D point clouds after multi-view stereo [S24]. Step 2 adapts a monocular depth network to the target scene by finetuning it on the sparse depth maps from COLMAP using a scale-invariant loss [S24].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S25]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S25]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S25].

### Volumetric Supervision
A training strategy that uses high-quality volumetric reconstructions from conventional tomography (available at start/end times) to supervise the neural network training at intermediate times with sparse projections [S26, unofficial]. It obtains rich projection data at $t=0$ and $t=1$, reconstructs 3D volumes using conventional tomography, and during training at intermediate times, computes the normalized correlation coefficient ($C_{cor}$) between the network's predicted volume and the CT-reconstructed volume [S26, unofficial]. The volumetric loss is minimized as $L_{vol} = -C_{cor}(y, \hat{y})$ [S26, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot K)$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the batch size (rays per step), and $K$ is the number of samples per ray [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot K)$ for the forward pass activations, plus $O(\theta)$ for the model parameters and optimizer states [S5, S18, unofficial]. VRAM usage is a primary constraint, often requiring chunking of ray batches [S5, S15, unofficial].
*   **Measured Performance:** Training typically runs for 100,000 to 1,000,000 iterations [S11, S14, unofficial]. For a scene with ~200 images, training may take several hours on a modern GPU [S17, unofficial].
*   **Scaling:** Performance scales with batch size up to GPU memory limits; larger batches improve throughput but risk OOM errors [S15, unofficial].
*   **Strengths:** Simple to implement, differentiable end-to-end, produces high-quality novel views [S5, S14, unofficial].
*   **Limitations:** Slow convergence due to low-frequency bias of MLPs, high memory usage for large scenes, sensitive to hyperparameters [S1][S17, unofficial].
*   **When to Use:** When high-quality view synthesis is required and computational resources are sufficient for iterative optimization [S5, unofficial].
*   **When Not to Use:** For real-time applications or extremely large scenes where partitioning or explicit representations are needed [S1][S3, unofficial].

## Background
The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, S18, unofficial]. It established the standard paradigm of optimizing a coordinate-based MLP to represent a 3D scene by minimizing the photometric loss between rendered and observed images [S5, unofficial]. The method builds on earlier work in implicit neural representations and differentiable rendering [S3, S18, unofficial]. It is widely used in computer vision and graphics for novel view synthesis, 3D reconstruction, and scene editing [S1][S5, unofficial].

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
[S23] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S24] Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo, https://doi.org/10.1109/tpami.2023.3263464, fetched 2026-10-27
[S25] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S26] Neural rendering enables dynamic tomography, https://doi.org/10.48550/arxiv.2410.20558, fetched 2026-10-26 (unofficial)
