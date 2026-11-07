# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric error between rendered images and ground-truth observations by updating the parameters of a neural radiance field [S5, unofficial][S6, unofficial][S14, unofficial]. It differs from standard volume rendering by treating the scene representation as a differentiable function $f_\theta$ and using stochastic gradient descent to adjust $\theta$ such that the volume rendering equation produces accurate RGB outputs [S5, unofficial][S14, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A dataset of $N$ training images with known camera poses (extrinsics $C$ and intrinsics $K$) [S5, unofficial][S11, unofficial]. A neural network model $f_\theta$ (typically an MLP) parameterizing density $\sigma$ and color $\mathbf{c}$ [S5, unofficial].
*   **Outputs:** Optimized network weights $\theta$ and a trained radiance field capable of synthesizing novel views [S5, unofficial].

**Data Structures:**
*   **Rays:** A batch of rays is represented as a tensor of shape $[B, 3, 3]$ or $[B, 6]$ (depending on implementation), where $B$ is the batch size ($N_{rand}$) [S5, unofficial][S14, unofficial].
    *   Axis 0 (if shape $[B, 3, 3]$): Channel type (0: Ray Origin, 1: Ray Direction, 2: Target RGB) [S5, unofficial][S14, unofficial].
    *   Axis 1: XYZ/RGB components [S5, unofficial][S14, unofficial].
    *   Alternatively, separate tensors for origins $\mathbf{o}$, directions $\mathbf{d}$, and targets $\mathbf{c}_{gt}$ [S6, unofficial].
*   **Pre-computed Rays:** To accelerate data loading, rays are often pre-computed into a flat array `rays_rgb` of shape $[(N-1) \cdot H \cdot W, 3, 3]$ containing all possible ray origins, directions, and target colors from the training set, shuffled randomly [S5, unofficial][S14, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S8][S11, unofficial].
    *   Initialize the neural network $f_\theta$ (e.g., with He initialization or SIREN-specific weights) [S3, unofficial][S15, unofficial].
    *   Initialize the optimizer (typically Adam) with a learning rate $\eta$ (default $5 \times 10^{-4}$) [S5, unofficial][S11, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, unofficial][S11, unofficial].
    *   Precompute ray bundles for all training pixels if batching is enabled [S5, unofficial][S14, unofficial].

2.  **Iteration Loop:** For iteration $i$ from 1 to $N_{iters}$ (default 1,000,000) [S14, unofficial]:
    a.  **Ray Sampling:**
        *   Select a batch of $N_{rand}$ rays (default 1024 or 2048) from the pre-computed `rays_rgb` array or by randomly sampling pixel coordinates from training images [S5, unofficial][S14, unofficial].
        *   If using precropping, restrict sampling to a central crop of the image for the first $N_{precrop}$ iterations [S2, unofficial].
        *   Extract ray origins $\mathbf{o}$, directions $\mathbf{d}$, and ground-truth colors $\mathbf{c}_{gt}$ for the batch [S5, unofficial][S14, unofficial].
    b.  **Forward Pass (Rendering):**
        *   Sample $N$ points along each ray within the near/far bounds [S5, unofficial].
        *   Query the network $f_\theta$ at each point to obtain density $\sigma$ and color $\mathbf{c}$ [S5, unofficial].
        *   Compute the rendered color $\hat{\mathbf{c}}$ using volume rendering (alpha compositing) [S5, unofficial].
        *   If using a coarse-to-fine strategy, render with a coarse network to get weights, then sample additional points for a fine network [S5, unofficial][S11, unofficial].
    c.  **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) between rendered color $\hat{\mathbf{c}}$ and ground truth $\mathbf{c}_{gt}$ [S5, unofficial][S14, unofficial]:
            $$ \mathcal{L}_{MSE} = \frac{1}{B} \sum_{j=1}^{B} \| \hat{\mathbf{c}}_j - \mathbf{c}_{gt,j} \|^2 $$
        *   If coarse outputs are available, add the coarse loss $\mathcal{L}_{coarse}$ to the total loss [S11, unofficial]:
            $$ \mathcal{L} = \mathcal{L}_{fine} + \mathcal{L}_{coarse} $$
        *   Apply any additional regularization terms (e.g., sparsity, eikonal) if configured [S7, unofficial][S8].
    d.  **Backpropagation and Update:**
        *   Zero the gradients of the optimizer [S18, unofficial].
        *   Compute gradients via backpropagation: $\nabla_\theta \mathcal{L}$ [S18, unofficial].
        *   Step the optimizer: $\theta \leftarrow \theta - \eta \nabla_\theta \mathcal{L}$ [S18, unofficial].
        *   Update the learning rate according to the scheduler [S18, unofficial].
    e.  **Logging and Checkpointing:**
        *   Log loss and PSNR every $N_{log}$ steps [S11, unofficial].
        *   Render validation images every $N_{val}$ steps [S11, unofficial].
        *   Save model checkpoints every $N_{ckpt}$ steps [S11, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def train_nerf(model, train_loader, num_epochs, batch_size, lr, device):
    """
    Standard NeRF training loop.
    
    Args:
        model: Neural Radiance Field model (MLP).
        train_loader: DataLoader yielding batches of (rays_o, rays_d, target_rgb).
        num_epochs: Number of training epochs.
        batch_size: Number of rays per batch (N_rand).
        lr: Learning rate for Adam optimizer.
        device: 'cpu' or 'cuda'.
    """
    model = model.to(device)
    optimizer = optim.Adam(model.parameters(), lr=lr)
    mse_loss = nn.MSELoss()
    
    # Optional: Learning rate scheduler (e.g., Exponential Decay)
    # scheduler = optim.lr_scheduler.ExponentialLR(optimizer, gamma=0.1)

    for epoch in range(num_epochs):
        model.train()
        for rays_o, rays_d, target_rgb in train_loader:
            # Move data to device
            rays_o = rays_o.to(device)
            rays_d = rays_d.to(device)
            target_rgb = target_rgb.to(device)
            
            # Zero gradients
            optimizer.zero_grad()
            
            # Forward pass: Render rays
            # render_rays is a function that implements volume rendering
            # It returns predicted RGB and potentially depth/accumulation
            pred_rgb = render_rays(model, rays_o, rays_d, device)
            
            # Compute Loss
            loss = mse_loss(pred_rgb, target_rgb)
            
            # Backward pass
            loss.backward()
            
            # Optimizer step
            optimizer.step()
            
            # Optional: Update scheduler
            # scheduler.step()

        # End of epoch logging/checkpointing would go here
```

**Parameters with Typical Values:**
*   $N_{rand}$ (Batch Size): 1024 or 2048 rays per iteration [S5, unofficial][S14, unofficial].
*   $N_{iters}$ (Total Steps): 1,000,000 [S14, unofficial].
*   Learning Rate ($\eta$): $5 \times 10^{-4}$ [S11, unofficial].
*   Positional Encoding Frequency ($L$): 10 for position, 4 for direction [S11, unofficial].
*   Decay Rate: 0.1 [S6, unofficial][S18, unofficial].
*   Decay Steps: $args.lrate\_decay \times 1000$ [S5, unofficial][S18, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering all rays at once can exceed GPU memory. Mitigated by chunking rays during the forward pass [S5, unofficial].
*   **Instability in Growing Networks:** Directly initializing grown sub-networks randomly can cause density to collapse to zero. Initialize grown sub-networks' alpha weights to match the parent network [S1].
*   **Gradient Graph Destruction:** Higher-order derivatives (for logging gradients/laplacians) must be computed *before* calling `loss.backward()`, as the backward pass frees the autograd graph [S3, unofficial].
*   **Checkpoint Resumption:** When resuming, ensure the learning rate scheduler's last epoch is set to `start_epoch - 1` to maintain correct decay scheduling [S9, unofficial].
*   **Empty Checkpoints:** If a loaded checkpoint has step 0, it is considered empty and should trigger an error or re-initialization [S15, unofficial].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images from a neural radiance field for gradient-based optimization. It solves the problem of efficiently sampling 3D space along camera rays, querying a neural network for density and color, and integrating these values via alpha compositing to produce a loss-computable image. It sets itself apart from [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md) by computing gradients for backpropagation and from [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) by including the adaptive sampling and occupancy grid pruning steps specific to the training loop.

### [Training Configuration](2332-training-configuration.md)
Defines the static hyperparameters, optimizer settings, and structural constraints required to stabilize the optimization of differentiable volume rendering models. It solves the problem of translating theoretical radiance field formulations into convergent training procedures by specifying exact network architectures, sampling strategies, and loss weights. Unlike [Training Loop Setup](1522-training-loop-setup.md), which defines the dynamic runtime sequence of data loading and gradient updates, this method specifies the immutable parameters governing the optimization landscape. It differs from [Regularization and Priors](6281-regularization-and-priors.md) by focusing on the numerical optimization settings (e.g., learning rates, batch sizes) rather than the specific penalty terms added to the loss function.

### [Mip-NeRF 360](3306-mip-nerf-360.md)
A method for synthesizing photorealistic views of unbounded, real-world scenes where the camera may rotate 360 degrees and content exists at arbitrary distances. It solves the problems of aliasing, training instability, and geometric ambiguity inherent in standard NeRF when applied to large-scale environments. It sets itself apart from its parent [NeRF Training Loop](970-nerf-training-loop.md) by integrating a non-linear scene parameterization, online distillation between two MLPs, and a distortion-based regularizer, achieving a 57% reduction in mean squared error compared to standard mip-NeRF.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [MVSNeRF](6197-mvsnerf.md)
A method for fast, generalizable radiance field reconstruction from multi-view stereo (MVS) images. Unlike standard NeRF which requires per-scene optimization, MVSNeRF uses a feedforward neural network to predict the radiance field directly from input images, significantly reducing training time and iteration counts. It sets itself apart by learning a data prior that accelerates convergence, enabling rapid inference without iterative scene-specific optimization.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
This family of methods addresses the problem of optimizing Neural Radiance Fields (NeRF) when input camera poses (extrinsics) and/or intrinsic parameters (e.g., focal length) are unknown, noisy, or inaccurate. Standard NeRF training assumes ground-truth camera poses as fixed inputs; these methods treat camera parameters as trainable variables within the same differentiable pipeline, jointly optimizing them alongside the NeRF weights. They set themselves apart from siblings like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by explicitly computing gradients with respect to camera transformation matrices or Lie algebra vectors, allowing the geometry and viewpoint to co-evolve.

### [Specific Model Architectures](8585-specific-model-architectures.md)
Specific Model Architectures refers to the family of neural network designs used to parameterize the radiance field function $f_\theta(\mathbf{x}, \mathbf{d})$ within the NeRF optimization loop. While the standard NeRF implementation uses a simple Multi-Layer Perceptron (MLP) to map 3D coordinates and viewing directions to density and color, this family encompasses modifications to that core mapping to address specific limitations such as training speed, generalization, geometric ambiguity, and resolution constraints. These architectures differ from their siblings in the training loop (such as [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) or [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)) by defining *what* is being learned and *how* the spatial data is structured, rather than *how* the samples are selected or integrated.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Regularization and Loss Functions address the severe under-constraint problem in Neural Radiance Fields (NeRFs), where limited or inconsistent input views lead to geometric artifacts (e.g., "floaters"), overfitting, and implausible geometry. This family sets itself apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the objective function rather than the sampling strategy, and from [Mip-NeRF 360](3306-mip-nerf-360.md) by explicitly introducing priors (diffusion, flow, or geometric) to constrain the solution space beyond photometric consistency.

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

### [Zip-NeRF Training Procedure](9538-zip-nerf-training-procedure.md)
The Zip-NeRF Training Procedure is a two-stage optimization method for inverse rendering that jointly trains a physically-based scene model, environment map, and radiance caches (Zip-NeRF and Fast Cache) to estimate incoming radiance. It solves the bias introduced by approximating incoming radiance using neural caches by employing a control variate strategy and a specific gradient trick for the photometric loss. It sets itself apart from standard NeRF training by decoupling the cache initialization from the joint optimization of the physical model and by using an occlusion-aware importance sampler to reduce variance in the radiance estimation.

### [DPPE (Dense Pose Estimation in a Plenoxels Environment)](9640-dppe-dense-pose-estimation-in-a-plenoxels-environment.md)
DPPE is a 6-DoF monocular RGB-only pose estimation procedure for Plenoxels environments that recovers the ground truth camera pose after a perturbation by minimizing photometric errors in re-rendering. It distinguishes itself from analytical derivative-based methods (e.g., Teigen et al.) by using numerical gradient approximation, making it generalizable to any scene representation where re-renderings are not prohibitively expensive, and from NeRF-based methods (e.g., iNeRF) by leveraging Plenoxels' rendering speed to re-render the full image at each optimization step rather than subsampling rays.

### ReconFusion
Uses a Diffusion Prior to constrain unobserved views during the training loop [S21, unofficial].

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function. The full-resolution supersampling baseline relies on 'oracle' scale information that does not exist for most datasets. The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S22].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets. Phase 2 trains all network components end-to-end. At the start of Phase 2, the optimizer state is reset and a linear warmup for 1,000 steps is performed to reduce shock from switching regimes. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S23, unofficial].

### Multi-view SDS
A plug-and-play method that creates multi-view supervision during SDS optimization to address the Janus problem caused by one-by-one view sampling. Rendered images from 4 sampled views are stitched into a large 2x2 image. Gradients are calculated under the condition of the multi-view normal map. The method converts a 3D asset into differentiable representations including NeRF, 3DGS, and DMTet through rendering supervision, which are then optimized to improve the quality of the 3D asset [S24].

### Guided NeRF Optimization for MVS
A multi-view depth estimation method that utilizes conventional reconstruction (sparse SfM+MVS) and learning-based priors over Neural Radiance Fields (NeRF). It performs test-time optimization directly over implicit volumes to eliminate the need for accurate pixel matching across views, addressing the shape-radiance ambiguity of NeRF in indoor scenes. Step 1: Run COLMAP on the test scene to acquire per-view sparse depth maps by projecting fused 3D point clouds after multi-view stereo. Step 2: Adapt a monocular depth network to the target scene by finetuning it on the sparse depth maps from COLMAP using a scale-invariant loss [S25].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S26].

## Operating characteristics
*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot N_{samples})$ per iteration, where $N_{iters}$ is total iterations, $N_{rand}$ is batch size, and $N_{samples}$ is samples per ray [S5, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot N_{samples})$ for intermediate rendering states during backpropagation if using standard AD [S5, unofficial]. Chunking reduces peak VRAM usage [S5, unofficial].
*   **Measured Performance:** Training typically runs for 1,000,000 iterations [S14, unofficial]. On synthetic datasets (Blender), training takes several hours on a single GPU [S11, unofficial].
*   **Scaling:** Performance scales with batch size up to VRAM limits; larger batches improve throughput but may require gradient accumulation [S15, unofficial].
*   **Strengths:** End-to-end differentiable, handles complex lighting and geometry implicitly, produces high-quality novel views [S5, unofficial].
*   **Limitations:** Slow training convergence, high memory usage for large scenes, prone to overfitting on sparse views [S1][S17, unofficial].
*   **When to Use:** When high-quality novel view synthesis is required and ground-truth geometry is not available [S5, unofficial].
*   **When Not to Use:** When real-time rendering is required or when explicit geometry is needed [S17, unofficial].

## Background
The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, unofficial]. It builds upon the concept of coordinate-based neural networks for implicit representations [S3, unofficial]. The method has become a standard baseline for inverse volume rendering and has inspired numerous variants focusing on efficiency, generalization, and robustness [S1][S15, unofficial].

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
[S21] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S22] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S23] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
[S24] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S25] Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo, https://doi.org/10.1109/tpami.2023.3263464, fetched 2026-10-27
[S26] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
