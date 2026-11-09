# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that trains a coordinate-based neural network (MLP) to represent a 3D scene by minimizing the difference between rendered images and ground-truth observations [S5, S6, S14, unofficial]. It solves the inverse problem of reconstructing a continuous volumetric radiance field from discrete 2D images by sampling random rays, computing differentiable volume rendering predictions, and updating network weights via gradient descent [S5, S6, unofficial]. It sets itself apart from evaluation by using stochastic ray sampling and adaptive strategies (e.g., occupancy grids, coarse-to-fine sampling) to manage memory and accelerate convergence, whereas evaluation disables these for deterministic results [S2, S4, S9, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with known camera poses (extrinsics $P$ and intrinsics $K$), and a neural network architecture (typically an MLP) initialized with random weights.
*   **Outputs:** A trained neural network where the output at any 3D coordinate $\mathbf{x}$ and viewing direction $\mathbf{d}$ is the volume density $\sigma$ and view-dependent color $\mathbf{c}$.

**Data Structures:**
*   **Rays:** A batch of rays is represented as a tensor of shape $[B, 3, 3]$ or $[B, 9]$, where $B$ is the batch size (number of rays). The channels/elements correspond to:
    1.  Ray Origin $(o_x, o_y, o_z)$
    2.  Ray Direction $(d_x, d_y, d_z)$
    3.  Target Ground-Truth Color $(r, g, b)$ [S5, S14, unofficial].
    *   In some implementations, rays are pre-computed into a flat array `rays_rgb` of shape $[(N-1) \cdot H \cdot W, 3, 3]$ containing all possible pixel rays from the training set, shuffled for random access [S5, S14, unofficial].
*   **Model State:** Checkpoints typically store the model `state_dict`, optimizer `state_dict`, and current step/epoch [S10, S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S9, S11, unofficial].
    *   Initialize the neural network (e.g., MLP with positional encoding) [S5, S11, unofficial].
    *   Initialize the optimizer (typically Adam) and learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   Precompute ray bundles (origins, directions) and target colors for all training pixels if batching is enabled [S5, S11, unofficial].

2.  **Ray Sampling:**
    *   For each iteration, sample a batch of $N_{rand}$ rays [S5, S14, unofficial].
    *   If using pre-computed rays, slice the `rays_rgb` array and increment the batch index; shuffle if exhausted [S5, S14, unofficial].
    *   If not pre-computed, randomly select pixel coordinates from training images, convert to ray origins/directions using camera matrices, and gather target RGB values [S14, unofficial].
    *   *Edge Case:* For forward-facing scenes, rays may be converted to Normalized Device Coordinates (NDC) [S2, S6, unofficial]. Near and far bounds ($t_{near}, t_{far}$) are set based on scene geometry (e.g., fixed for Blender, computed from bounding depths for LLFF) [S2, S13, unofficial].

3.  **Forward Pass (Differentiable Volume Rendering):**
    *   Sample $K$ points along each ray between $t_{near}$ and $t_{far}$ [S5, S11, unofficial].
        *   Uniform sampling: $t_i = t_{near} + \frac{i}{K}(t_{far} - t_{near})$ [S5, unofficial].
        *   Stratified sampling: Add uniform noise to intervals between sample points to reduce aliasing [S5, S11, unofficial].
    *   Query the neural network at each point $\mathbf{x}_i$ with position and viewing direction $\mathbf{d}$ to get raw output $(\sigma_i, \mathbf{c}_i)$ [S5, S11, unofficial].
    *   Compute transmittance $T_i$ and weights $w_i$ using the emission-absorption model:
        $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
        $$ w_i = T_i (1 - \exp(-\sigma_i \delta_i)) $$
        where $\delta_i$ is the distance between adjacent samples [S5, S11, unofficial].
    *   Composite the final color $\hat{C}(\mathbf{r})$:
        $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^K w_i \mathbf{c}_i $$
    *   *Coarse-to-Fine:* If using a two-stage approach, render a coarse image, sample additional points based on coarse weights (importance sampling), and render a fine image [S5, S11, unofficial].

4.  **Loss Computation:**
    *   Compute the Mean Squared Error (MSE) between the rendered color $\hat{C}$ and the ground-truth color $C_{gt}$ [S5, S14, unofficial]:
        $$ \mathcal{L}_{rgb} = \frac{1}{B} \sum_{b=1}^B \|\hat{C}_b - C_{gt,b}\|^2 $$
    *   *Optional:* If a coarse network is used, add the coarse loss $\mathcal{L}_{coarse}$ to the total loss [S11, unofficial].
    *   *Optional:* Add regularization terms (e.g., sparsity, TV, Eikonal) if defined in the specific variant [S7, unofficial][S8].

5.  **Backpropagation and Update:**
    *   Zero gradients: `optimizer.zero_grad()` [S18, unofficial].
    *   Compute gradients: `loss.backward()` [S18, unofficial].
        *   *Pitfall:* Higher-order derivatives (e.g., for logging gradients or Laplacians) must be computed *before* `loss.backward()` to avoid freeing the autograd graph [S3, unofficial].
    *   Update weights: `optimizer.step()` [S18, unofficial].
    *   Update learning rate according to the scheduler [S5, S11, unofficial].

6.  **Logging and Checkpointing:**
    *   Log loss and PSNR ($PSNR = -10 \log_{10}(MSE)$) at specified intervals [S11, S14, unofficial].
    *   Render validation images and save checkpoints at specified intervals [S11, unofficial].

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
        model: Neural Radiance Field MLP.
        optimizer: Optimizer (e.g., Adam).
        scheduler: Learning rate scheduler.
        dataloader: Iterator yielding batches of (rays_o, rays_d, target_rgb).
        num_epochs: Number of training epochs.
        device: 'cpu' or 'cuda'.
        render_fn: Differentiable volume rendering function.
        loss_fn: Loss function (default MSE).
    """
    model.train()
    
    for epoch in range(num_epochs):
        for batch in dataloader:
            # 1. Data Loading
            # batch contains: rays_o (B, 3), rays_d (B, 3), target_rgb (B, 3)
            rays_o = batch['rays_o'].to(device)
            rays_d = batch['rays_d'].to(device)
            target_rgb = batch['target_rgb'].to(device)
            
            # 2. Forward Pass (Rendering)
            # render_fn returns predicted RGB (B, 3) and potentially extras (depth, acc)
            # It handles ray sampling, network querying, and alpha compositing internally
            predicted_rgb, extras = render_fn(model, rays_o, rays_d)
            
            # 3. Loss Computation
            # Standard photometric loss
            loss = loss_fn(predicted_rgb, target_rgb)
            
            # Optional: Add coarse loss if two-stage training is active
            if 'rgb_coarse' in extras:
                loss += loss_fn(extras['rgb_coarse'], target_rgb)
                
            # Optional: Add regularization (e.g., sparsity, TV) here if defined
            
            # 4. Backpropagation
            optimizer.zero_grad()
            loss.backward()
            
            # 5. Update Weights
            optimizer.step()
            
            # 6. Update Learning Rate
            scheduler.step()
            
            # 7. Logging (every N steps)
            # psnr = -10 * log10(loss.item())
            # if step % log_interval == 0:
            #     print(f"Epoch {epoch}, Loss: {loss.item():.6f}, PSNR: {psnr:.2f}")
            
            # 8. Checkpointing (every N epochs)
            # if epoch % checkpoint_interval == 0:
            #     save_checkpoint(model, optimizer, epoch)
```

**Parameters with Typical Values:**
*   **Learning Rate (`lr`):** $5 \times 10^{-4}$ for Adam [S11, unofficial].
*   **Batch Size (`N_rand`):** 1024 or 2048 rays per iteration [S5, S11, unofficial].
*   **Samples per Ray (`K`):** 128 (coarse) + 128 (fine) [S5, S11, unofficial].
*   **Total Iterations:** 100,000 to 1,000,000 [S11, S14, unofficial].
*   **Positional Encoding Frequency (`L`):** 10 for position, 4 for direction [S11, unofficial].
*   **Learning Rate Decay:** Exponential decay with `decay_rate=0.1` over `decay_steps` (e.g., 100,000 steps) [S5, S18, unofficial].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
Training Ray Marching and Integration is the differentiable forward pass that synthesizes RGB images from a neural radiance field (NeRF) during optimization. It solves the problem of computing gradients for scene parameters by sampling points along camera rays, querying a neural network for volume density and color, and integrating them via alpha compositing. It sets itself apart from evaluation by using stochastic perturbations, adaptive sampling strategies (e.g., occupancy grid pruning), and chunked processing to manage memory, whereas evaluation disables these for deterministic results.

### [Training Configuration](2332-training-configuration.md)
Training Configuration defines the hyperparameters, optimization routines, and architectural constraints required to stabilize and accelerate the differentiable volume rendering optimization process. It addresses the sensitivity of radiance field training to initialization, sampling density, and activation function choices. Unlike general optimization loops, it specifies implementation-specific details such as learning rate schedules, loss weights, and batch sampling strategies that are critical for convergence in methods like mip-NeRF, Instant-NGP, and NeuralVDB.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
Camera Pose and Parameter Optimization refers to the family of methods that jointly optimize the parameters of a Neural Radiance Field (NeRF) and the camera poses (extrinsics) and/or intrinsics (e.g., focal length) from a set of input images. It solves the problem of reconstructing 3D scenes when camera parameters are unknown, inaccurate, or when the system requires online pose estimation for navigation. Unlike baseline NeRF, which relies on pre-estimated poses from Structure-from-Motion (SfM) pipelines like COLMAP, these methods treat camera parameters as learnable variables updated via gradient descent alongside the NeRF weights. This family encompasses static scene reconstruction with unknown poses, hybrid 2D/3D registration, and online robot navigation filters.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Regularization and Loss Functions define the objective landscape for optimizing Neural Radiance Fields (NeRFs), addressing the ill-posed nature of inverse volume rendering where infinite density-color configurations can produce identical images. This family supplements the standard photometric reconstruction loss with auxiliary terms that enforce geometric plausibility, structural consistency, or alignment with external priors, thereby preventing degenerate solutions such as "floaters," image copying, or over-saturation. It distinguishes itself from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the *what* to optimize (the loss landscape) rather than the *how* of sampling, and from [Training Configuration](2332-training-configuration.md) by providing the specific mathematical formulations rather than hyperparameter schedules.

### [Progressive Inpainting-and-Erasing](8658-progressive-inpainting-and-erasing.md)
Progressive Inpainting-and-Erasing (PERF) is a training strategy for single-view NeRF that avoids geometry conflicts between visible and invisible regions by computing a mask of conflicted regions and eliminating them from NeRF training. It sets itself apart from standard single-view NeRF methods by integrating a collaborative RGBD inpainting step with a progressive erasing mechanism to handle the ambiguity of unseen geometry, building on the [NeRF Training Loop](970-nerf-training-loop.md) by dynamically expanding the reference view set.

### [KNN-NeRF](8672-knn-nerf.md)
KNN-NeRF is a baseline method for training Neural Radiance Fields (NeRFs) using sparse 3D point clouds as supervision instead of multi-view images. It solves the problem of generating ground-truth color labels for rays that do not pass through existing points by assigning the color of the nearest point in the cloud to the ray. Unlike standard NeRF which relies on pixel colors from images, KNN-NeRF constructs a dense training signal from sparse geometry, though it struggles with sharp object borders due to the low-frequency bias of neural networks and the sparsity of the input.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Progressive and Dynamic Scene Growth addresses the challenge of reconstructing large-scale or complex 3D scenes from limited input data by incrementally expanding the neural representation's capacity or the training dataset. Unlike static NeRF architectures that require pre-defining the entire hierarchy or fixed network size, these methods adaptively grow the model structure or add synthetic reference views during training to resolve occlusions and capture high-frequency details. This family distinguishes itself from sibling methods like [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md) (which focuses on ray selection) and [Sparse NeRF Reconstruction with Geometric and Semantic Initialization](8779-sparse-nerf-reconstruction-with-geometric-and-semantic-initi.md) (which relies on external priors) by actively modifying the model's topology or the set of supervision signals to improve reconstruction quality in complex regions.

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Adaptive ray sampling refers to a family of training procedures in neural radiance fields (NeRF) that replace uniform ray sampling with strategies that concentrate computational resources on regions of high geometric or photometric complexity. It solves the inefficiency of uniform sampling, which wastes evaluations on empty space or smooth regions, by dynamically adjusting sample locations or ray selection based on density, error, depth priors, or spatial partitions. This family is distinguished from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on *where* to sample rather than *how* to integrate, and from [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) by encompassing broader strategies including error-based mining, depth-guided selection, and superpixel partitioning, not just hierarchical point clustering.

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

### [Scene Representation and Architecture](11368-scene-representation-and-architecture.md)
Scene Representation and Architecture defines the mathematical mapping and neural network structures that convert 3D spatial coordinates and viewing directions into volumetric properties (density and color). It solves the problem of how to compactly and accurately represent a 3D scene's geometry and appearance for differentiable rendering. It sets itself apart from its siblings by focusing on the *definition* of the scene function itself, whereas [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) handles the numerical evaluation of that function, and [Training Configuration](2332-training-configuration.md) handles the hyperparameters of the optimization process.

### [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)
This family of methods accelerates NeRF training and rendering by reducing the computational cost of evaluating the neural radiance field. It addresses the bottleneck of querying high-dimensional MLPs for every sample point along every ray. Unlike standard uniform or adaptive sampling over the entire scene bounding box, these methods restrict evaluation to sparse structures (visual hulls) or use lightweight approximations (caches) to predict radiance, thereby reducing the number of expensive neural network queries.

### [Joint Optimization of Camera Pose and Decomposed Low-Rank Tensorial Radiance Fields](11498-joint-optimization-of-camera-pose-and-decomposed-low-rank-te.md)
Joint TensoRF is an algorithm for the joint refinement of camera pose and scene geometry represented by decomposed low-rank tensors, using only 2D images as supervision. It addresses the instability of naive joint optimization on voxel-based NeRFs, which lack the spectral bias of MLPs and tend to overemphasize sharp edges, leading to local optima. Unlike standard voxel-based methods, it employs a coarse-to-fine training schedule via convolutional Gaussian filters applied to 2D and 3D radiance fields, enabling robust joint optimization.

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot K)$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the batch size (rays per step), and $K$ is the number of samples per ray [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot K)$ for storing intermediate rendering states during the forward pass, plus $O(P)$ for model parameters, where $P$ is the number of MLP weights [S5, unofficial]. VRAM usage is a primary constraint, often mitigated by chunking rays [S5, S15, unofficial].
*   **Measured Performance:** Training typically requires 100,000 to 1,000,000 iterations, taking several hours to days on a single GPU depending on resolution and scene complexity [S11, S14, unofficial].
*   **Strengths:** End-to-end differentiable; handles complex geometry and appearance; produces high-quality novel views.
*   **Limitations:** Slow convergence due to random sampling; prone to "floaters" (spurious geometry) in empty space; sensitive to hyperparameters (learning rate, encoding frequency) [S1][S17, unofficial].
*   **When to Use:** When high-fidelity novel view synthesis is required and ground-truth 3D geometry is not available.
*   **When Not to Use:** When real-time rendering is required (use [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)) or when camera poses are unknown (use [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)).

## Background
The NeRF training loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, S18, unofficial]. It established the standard paradigm of optimizing a coordinate MLP via stochastic gradient descent on photometric loss. Subsequent developments included coarse-to-fine sampling [S5, unofficial], positional encoding enhancements [S20], and various regularization techniques [S7, unofficial][S8]. The method has become the foundational training procedure for most neural radiance field variants.

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
[S25] HDR-NeRF: High Dynamic Range Neural Radiance Fields, https://xhuangcv.github.io/hdr-nerf/, fetched 2026-10-27
