# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric error between rendered images and ground-truth observations by updating the parameters of a neural radiance field [S5, S6, S14, unofficial]. It solves the inverse problem of reconstructing a continuous 3D scene representation from 2D images by treating the radiance field as a differentiable function optimized via gradient descent [S3, S5, unofficial]. It sets itself apart from evaluation by using stochastic ray sampling, adaptive perturbations, and chunked processing to manage memory and enable efficient gradient-based learning, whereas evaluation disables these for deterministic results [S2, S4, S9, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of $N$ training images with known camera poses (extrinsics/intrinsics) [S2, S5, unofficial]. The neural network parameters $\Theta$ (weights of the MLPs) [S5, S11, unofficial].
*   **Outputs:** Optimized network weights $\Theta^*$ that minimize the reconstruction loss [S5, unofficial].

**Data Structures:**
*   **Rays:** A ray is defined by origin $\mathbf{o}$ and direction $\mathbf{d}$. In implementation, rays are often pre-computed and stored in a flat array `rays_rgb` of shape $[(N-1) \times H \times W, 3, 3]$ where axis 1 represents [origin, direction, target_rgb] and axis 2 represents the XYZ/RGB components [S5, S14, unofficial]. Data type is typically `float32` [S5, S14, unofficial].
*   **Batch:** A subset of rays sampled for each optimization step. Shape is typically $[3, B, 3]$ (channels: origin, direction, target; batch size $B$; components) [S14, unofficial].
*   **Checkpoint:** A dictionary containing 'model' (state dict), 'optimizer' (state dict), and 'epoch' or 'step' (int) [S10, S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S9, S11, unofficial].
    *   Initialize the NeRF model(s). For standard NeRF, this involves creating MLPs for density and color [S5, S11, unofficial]. For HashNeRF, initialize hash tables [S7, unofficial]. For NGP, initialize the OccupancyDensityGrid with resolution (e.g., 128) [S15, unofficial].
    *   Initialize the optimizer (typically Adam or AdamW) with a learning rate (e.g., $5 \times 10^{-4}$) [S5, unofficial][S8][S11, S13, unofficial].
    *   Initialize a learning rate scheduler (e.g., ExponentialDecay, CosineAnnealing, or LambdaLR) [S5, unofficial][S8][S11, unofficial].
    *   If resuming from a checkpoint, load model and optimizer states, and set the scheduler's last epoch to `start_epoch - 1` to ensure correct scheduling [S9, S15, unofficial].

2.  **Ray Sampling (Per Iteration):**
    *   Select a batch of $N_{rand}$ rays (e.g., 1024 or 2048) from the pre-computed ray pool [S5, S14, unofficial].
    *   If batching is enabled, slice the pre-computed `rays_rgb` array [S5, S14, unofficial]. If not, randomly select pixel coordinates from training images and generate rays using camera intrinsics [S4, S14, unofficial].
    *   **Perturbation:** If enabled (`--perturb > 0`), add jitter to the sample positions along the ray to break symmetry and improve reconstruction quality [S2, S5, unofficial].
    *   **Near/Far Bounds:** Determine the near and far clipping planes for the ray segment. For synthetic datasets (Blender), fixed bounds (e.g., near=2, far=6) are used [S2, S5, unofficial]. For real-world datasets (LLFF), bounds are computed from the scene's bounding depth [S2, S5, unofficial].

3.  **Forward Pass (Rendering):**
    *   Query the neural network for volume density $\sigma$ and color $\mathbf{c}$ at the sampled points along the rays [S5, S6, unofficial].
    *   Compute the rendered color $\hat{C}(\mathbf{r})$ using the volume rendering equation (alpha compositing) [S5, S6, unofficial].
    *   If using a coarse-to-fine strategy (two networks), render a coarse image first, then sample additional points based on coarse weights for a fine render [S5, S11, unofficial].
    *   **Chunking:** To limit VRAM usage, process rays in chunks (e.g., `chunk=args.chunk`) if the batch size exceeds memory limits [S5, S15, unofficial].

4.  **Loss Computation:**
    *   Compute the Mean Squared Error (MSE) between the rendered color $\hat{C}$ and the ground truth color $C_{gt}$ [S3, S5, S6, S14, unofficial].
        $$ \mathcal{L}_{MSE} = \frac{1}{N} \sum_{i=1}^N (\hat{C}_i - C_{gt,i})^2 $$
    *   If a coarse network was used, add the coarse loss to the total loss [S6, S11, S18, unofficial].
    *   Optionally, add regularization terms (e.g., sparsity, Total Variation, Eikonal) [S7, unofficial][S8][S16, unofficial].
    *   For multi-modal data (e.g., ToF), combine RGB and phasor losses with a weight $\lambda$ [S19, unofficial].

5.  **Backpropagation and Update:**
    *   Zero the gradients of the optimizer [S3, S18, unofficial].
    *   Compute gradients via backpropagation: `loss.backward()` [S3, S18, unofficial].
    *   **Pitfall:** Higher-order derivatives (e.g., for logging gradients or Laplacians) must be computed *before* calling `loss.backward()`, as the backward pass frees the autograd graph [S3, unofficial].
    *   Step the optimizer: `optimizer.step()` [S3, S18, unofficial].
    *   Update the learning rate according to the scheduler [S9, S18, unofficial].

6.  **Logging and Checkpointing:**
    *   Log loss and PSNR metrics at regular intervals (e.g., every 100 steps) [S3, S11, unofficial].
    *   Render validation images at regular intervals (e.g., every 500 steps) [S11, unofficial].
    *   Save model checkpoints at regular intervals (e.g., every 10,000 steps) [S11, S15, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def train_nerf_loop(model, optimizer, scheduler, dataloader, num_epochs, 
                    device, render_fn, loss_fn=nn.MSELoss()):
    """
    Standard NeRF training loop.
    
    Args:
        model: Neural Radiance Field model (MLP).
        optimizer: Optimizer (e.g., Adam).
        scheduler: Learning rate scheduler.
        dataloader: Iterator yielding batches of (rays_o, rays_d, target_rgb).
        num_epochs: Number of training epochs.
        device: 'cpu' or 'cuda'.
        render_fn: Function that takes rays and model, returns rendered RGB.
        loss_fn: Loss function (default MSE).
    """
    model.train()
    
    for epoch in range(num_epochs):
        for batch_idx, (rays_o, rays_d, target_rgb) in enumerate(dataloader):
            # Move data to device
            rays_o = rays_o.to(device)
            rays_d = rays_d.to(device)
            target_rgb = target_rgb.to(device)
            
            # Zero gradients
            optimizer.zero_grad()
            
            # Forward pass: Render rays
            # render_fn handles sampling, network query, and volume integration
            rendered_rgb = render_fn(model, rays_o, rays_d)
            
            # Compute loss
            loss = loss_fn(rendered_rgb, target_rgb)
            
            # Backward pass
            loss.backward()
            
            # Optimizer step
            optimizer.step()
            
            # Update learning rate
            scheduler.step()
            
            # Logging (example)
            if batch_idx % 100 == 0:
                psnr = -10.0 * torch.log10(loss).item()
                print(f"Epoch {epoch}, Batch {batch_idx}, Loss: {loss.item():.6f}, PSNR: {psnr:.2f}")
                
        # Optional: Save checkpoint or render validation images here
```

**Parameters with Typical Values:**
*   **Learning Rate (`lr`):** $5 \times 10^{-4}$ for Adam [S11, unofficial].
*   **Batch Size (`N_rand`):** 1024 or 2048 rays per iteration [S5, S11, unofficial].
*   **Total Iterations:** 100,000 to 1,000,000 steps [S11, S14, unofficial].
*   **Positional Encoding Frequency (`multires`):** 10 for position, 4 for direction [S2, S11, unofficial].
*   **Decay:** Exponential decay with `decay_rate=0.1` over `decay_steps` (e.g., 100,000 steps) [S5, S18, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering all rays at once can exceed GPU memory. Mitigated by chunking rays into smaller batches during the forward pass [S5, S15, unofficial].
*   **Instability in Growing Networks:** Directly growing a randomly initialized sub-network causes density to drop to zero. Solution: Initialize alpha linear weights of grown sub-networks to match the parent [S1].
*   **Empty Checkpoints:** Loading a checkpoint with step 0 is considered an error; the training should restart or log an error [S15, unofficial].
*   **Gradient Graph Freeing:** Computing gradients for logging (e.g., Laplacian) must happen before `loss.backward()` [S3, unofficial].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images from a neural radiance field during optimization. It sets itself apart from evaluation by using stochastic perturbations, adaptive sampling strategies (e.g., occupancy grid pruning), and chunked processing to manage memory, whereas evaluation disables these for deterministic results.

### [Training Configuration](2332-training-configuration.md)
Defines the hyperparameters, optimization routines, and architectural constraints required to stabilize and accelerate the differentiable volume rendering optimization process. It addresses the sensitivity of radiance field training to initialization, sampling density, and activation function choices. Unlike general optimization loops, it specifies implementation-specific details such as learning rate schedules, loss weights, and batch sampling strategies that are critical for convergence in methods like mip-NeRF, Instant-NGP, and NeuralVDB.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
Refers to the family of methods that jointly optimize the parameters of a Neural Radiance Field (NeRF) and the camera poses (extrinsics) and/or intrinsics (e.g., focal length) from a set of input images. It solves the problem of reconstructing 3D scenes when camera parameters are unknown, inaccurate, or when the system requires online pose estimation for navigation. Unlike baseline NeRF, which relies on pre-estimated poses from Structure-from-Motion (SfM) pipelines like COLMAP, these methods treat camera parameters as learnable variables updated via gradient descent alongside the NeRF weights. This family encompasses static scene reconstruction with unknown poses, hybrid 2D/3D registration, and online robot navigation filters.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Defines the objective landscape for optimizing Neural Radiance Fields (NeRFs), addressing the ill-posed nature of inverse volume rendering where infinite density-color configurations can produce identical images. This family supplements the standard photometric reconstruction loss with auxiliary terms that enforce geometric plausibility, structural consistency, or alignment with external priors, thereby preventing degenerate solutions such as "floaters," image copying, or over-saturation. It distinguishes itself from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the *what* to optimize (the loss landscape) rather than the *how* of sampling, and from [Training Configuration](2332-training-configuration.md) by providing the specific mathematical formulations rather than hyperparameter schedules.

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
Progressive Inpainting-and-Erasing (PERF) is a training strategy for single-view NeRF that avoids geometry conflicts between visible and invisible regions by computing a mask of conflicted regions and eliminating them from NeRF training. It sets itself apart from standard single-view NeRF methods by integrating a collaborative RGBD inpainting step with a progressive erasing mechanism to handle the ambiguity of unseen geometry, building on the [NeRF Training Loop](970-nerf-training-loop.md) by dynamically expanding the reference view set.

### [KNN-NeRF](8672-knn-nerf.md)
KNN-NeRF is a baseline method for training Neural Radiance Fields (NeRFs) using sparse 3D point clouds as supervision instead of multi-view images. It solves the problem of generating ground-truth color labels for rays that do not pass through existing points by assigning the color of the nearest point in the cloud to the ray. Unlike standard NeRF which relies on pixel colors from images, KNN-NeRF constructs a dense training signal from sparse geometry, though it struggles with sharp object borders due to the low-frequency bias of neural networks and the sparsity of the input.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Addresses the challenge of reconstructing large-scale or complex 3D scenes from limited input data by incrementally expanding the neural representation's capacity or the training dataset. Unlike static NeRF architectures that require pre-defining the entire hierarchy or fixed network size, these methods adaptively grow the model structure or add synthetic reference views during training to resolve occlusions and capture high-frequency details. This family distinguishes itself from sibling methods like [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) (which focuses on ray selection) and [Sparse NeRF Reconstruction with Geometric and Semantic Initialization](8779-sparse-nerf-reconstruction-with-geometric-and-semantic-initi.md) (which relies on external priors) by actively modifying the model's topology or the set of supervision signals to improve reconstruction quality in complex regions.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Refers to a family of training procedures in neural radiance fields (NeRF) that replace uniform ray sampling with strategies that concentrate computational resources on regions of high geometric or photometric complexity. It solves the inefficiency of uniform sampling, which wastes evaluations on empty space or smooth regions, by dynamically adjusting sample locations or ray selection based on density, error, depth priors, or spatial partitions. This family is distinguished from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on *where* to sample rather than *how* to integrate, and from [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) by encompassing broader strategies including error-based mining, depth-guided selection, and superpixel partitioning, not just hierarchical point clustering.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
This family addresses the degradation of NeRF quality—such as blurred geometry, slow convergence, and floaters—when trained on sparse input views (typically 15–20 images or fewer than 50–100 views). It distinguishes itself from standard NeRF training by decoupling the initialization and regularization processes from the default random or empty-space assumptions, instead leveraging external geometric scaffolds, semantic masks, or visibility priors to anchor the optimization.

### [NerfingMVS](9518-nerfingmvs.md)
NerfingMVS is a multi-view depth estimation method that optimizes Neural Radiance Fields (NeRF) directly over implicit volumes, guided by learning-based depth priors derived from sparse Structure-from-Motion and Multi-View Stereo (SfM+MVS) reconstructions. It addresses the shape-radiance ambiguity inherent in NeRF for indoor scenes by eliminating the need for accurate pixel matching across views, instead using external depth estimates to guide the sampling and optimization process. Unlike standard NeRF which samples uniformly or stratified along rays, NerfingMVS incorporates depth priors and their associated confidences into the rendering pipeline to bias sampling towards geometrically plausible regions.

### [NeRV Training](10253-nerv-training.md)
NeRV training optimizes a continuous volumetric representation that decouples scene geometry (shape) from material appearance (reflectance) and lighting interactions (visibility) to enable efficient relighting and view synthesis. It differs from standard NeRF training by introducing a separate "visibility MLP" that predicts ray visibility and termination depth, supervised by consistency terms with the primary shape MLP rather than ground-truth labels, allowing the simulation of direct and one-bounce indirect illumination within the training loop.

### [NeRV Rendering](10264-nerv-rendering.md)
NeRV rendering computes the pixel color for a camera ray by integrating reflected radiance along the ray, where the radiance at each sample point is derived from a physically-based shading model that accounts for direct and indirect illumination. It differs from standard NeRF volume rendering by explicitly modeling surface normals and microfacet BRDF parameters via separate MLPs, and by estimating indirect lighting through secondary visibility queries rather than simple volumetric absorption.

### [Aug-NeRF](10294-aug-nerf.md)
Aug-NeRF is a training framework for Neural Radiance Fields (NeRF) that injects worst-case adversarial perturbations into three distinct levels of the pipeline—input coordinates, intermediate features, and pre-rendering output—to implicitly regularize the model with physical grounds. It addresses the generalization gap in NeRF where interpolation between seen views yields inconsistent and non-smooth geometries. Unlike explicit regularizers like Laplacian or Total Variation which constrain MLP flexibility aggressively, Aug-NeRF uses data-adaptive smoothness priors to enhance generalization to unseen views without sacrificing representation flexibility.

### [Cross-Guided Optimization of Radiance Fields (CROP)](10371-cross-guided-optimization-of-radiance-fields-crop.md)
Cross-Guided Optimization of Radiance Fields (CROP) is a framework that performs multi-view image super-resolution (MVSR) on training-view images during the radiance field optimization process to improve high-resolution novel view synthesis (HRNVS). It fuses single-image super-resolution (SISR) features with voxel-based uncertainty fields to ensure multi-view consistency and high-frequency details. Unlike methods that rely solely on the spectral characteristics of coordinate-based networks for upscaling, CROP explicitly bridges 2D SISR and 5D radiance fields by iteratively refining input images based on rendering errors.

### [Application-Specific Pipelines](11200-application-specific-pipelines.md)
Application-Specific Pipelines are specialized training and optimization procedures designed to address distinct downstream tasks—such as rapid reconstruction, pose estimation, or scene editing—rather than generic radiance field optimization. They diverge from standard NeRF training by integrating external priors (e.g., multi-view stereo data, diffusion models, or dynamics models) and modifying the loss landscape or rendering loop to suit specific constraints like speed, geometric accuracy, or semantic control.

### [NeRF-GT Baseline Training Setup](11349-nerf-gt-baseline-training-setup.md)
NeRF-GT is a baseline training protocol for Neural Radiance Fields that utilizes High Dynamic Range (HDR) input views to synthesize novel HDR imagery. It addresses the limitation of standard NeRF, which typically trains on Low Dynamic Range (LDR) images, by directly optimizing the radiance field against HDR ground truth. It differs from standard LDR-based training by explicitly handling high dynamic range data, though it faces challenges in encoding all scene areas uniformly due to the intensity variance.

### [Scene Representation and Architecture](11368-scene-representation-and-architecture.md)
Scene Representation and Architecture defines the neural network topology and input encoding strategies used to approximate the continuous radiance field $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ within the NeRF training loop. It solves the problem of representing a 3D scene as a differentiable function that can be optimized via gradient descent to match observed 2D images. It sets itself apart from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by defining *what* is being predicted (the field parameters) rather than *how* the prediction is sampled and integrated, and from [Specific Model Architectures](8585-specific-model-architectures.md) by establishing the canonical baseline against which specialized extensions are compared.

### [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)
This family of methods accelerates NeRF training and rendering by reducing the computational cost of evaluating the neural radiance field. It addresses the bottleneck of querying high-dimensional MLPs for every sample point along every ray. Unlike standard uniform or adaptive sampling over the entire scene bounding box, these methods restrict evaluation to sparse structures (visual hulls) or use lightweight approximations (caches) to predict radiance, thereby reducing the number of expensive neural network queries.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S21]. This baseline relies on 'oracle' scale information that does not exist for most datasets [S21]. Performance-wise, the 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 steps to reduce shock from switching regimes [S22, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S23]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S23].

### Incremental Registration
A training strategy that registers frames incrementally following temporal order, leveraging the intuition that camera motions in videos are smooth and adjacent poses are close [S24]. At the start of training, it jointly optimizes NeRF parameters $\Theta$ and poses $\{P_1, P_2\}$ from the first two frames $\{I_1, I_2\}$ [S24]. After every $k$ iterations (e.g., 1k for CO3D/HO3D, 500 for LLFF), it adds a new frame $I_i$ and initializes its pose $P_i$ by the previous pose $P_{i-1}$ [S24]. It freezes the learning rate on poses $\{P_i\}$ and NeRF $\Theta$ until all frames are registered (e.g., 100k iterations for HO3D/CO3D, 30k for LLFF), after which a learning rate decay schedule may be applied [S24].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot N_{samples} \cdot C_{MLP})$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the batch size (rays per step), $N_{samples}$ is the number of samples per ray, and $C_{MLP}$ is the cost of a forward pass through the neural network [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot N_{samples} \cdot C_{MLP} + VRAM_{chunk})$. Memory usage is dominated by the intermediate activations during the forward pass, which can be mitigated by chunking [S5, S15, unofficial].
*   **Measured Performance:** Training on 128x128 images typically takes 250 epochs [S4, unofficial]. For a dataset of 199 images down-sampled by 4, training with a batch size of 1024 results in ~7,800 iterations per epoch [S17, unofficial].
*   **Scaling:** Performance scales with batch size, but larger batches hinder data loading performance and prevent full GPU occupancy, while smaller batches cap iterations per second [S15, unofficial]. A batch size of $2^{10}$ (1024) is often chosen as a trade-off [S15, unofficial].
*   **Strengths:** Stochastic optimization allows for efficient training on large datasets by sampling only a subset of pixels per step [S5, unofficial]. The differentiable nature allows for end-to-end learning of scene representation and camera parameters [S2, unofficial].
*   **Limitations:** High memory consumption due to storing intermediate states for backpropagation [S5, unofficial]. Slow convergence for high-frequency details without positional encoding or advanced architectures [S20]. Prone to "floaters" and overfitting without regularization [S1].
*   **When to Use:** When reconstructing 3D scenes from multi-view images with known or estimable camera poses.
*   **When Not to Use:** When real-time rendering is required during training (use acceleration methods like [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)) or when camera poses are completely unknown and cannot be optimized jointly (use [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md) with caution).

## Background
The NeRF Training Loop originates from the seminal paper "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" by Mildenhall et al. (2020) [S5, unofficial]. It builds upon the concept of coordinate-based neural networks for implicit surface and volume representation [S3, unofficial]. The method was developed to address the limitations of traditional 3D reconstruction techniques (like MVS) in handling complex geometry and appearance, particularly for novel view synthesis. It is widely used in computer vision and graphics for applications such as 3D scene reconstruction, augmented reality, and digital twins [S20].

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
