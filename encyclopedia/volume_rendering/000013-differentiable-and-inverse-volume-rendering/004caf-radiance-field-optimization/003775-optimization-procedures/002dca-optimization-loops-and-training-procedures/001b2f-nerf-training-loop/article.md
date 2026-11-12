# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric reconstruction error between rendered images and ground-truth observations by updating the parameters of a continuous volumetric representation (typically a Multi-Layer Perceptron). It solves the inverse problem of recovering scene geometry and appearance from 2D images by leveraging the differentiability of the volume rendering equation [S5, S6, S14, unofficial]. It sets itself apart from evaluation rendering by using stochastic ray sampling and gradient-based updates rather than deterministic full-image synthesis, and from [Optimization Configuration](2133-optimization-configuration-adam-exponential-decay-selective-.md) by encompassing the complete cycle of data loading, forward rendering, loss computation, and weight updates [S5, S14, unofficial].

## How it works
The method optimizes a neural radiance field $\mathcal{F}_\Theta$ parameterized by $\Theta$ (weights and biases) to predict volume density $\sigma$ and view-dependent color $\mathbf{c}$ for any 3D point $\mathbf{x}$ and viewing direction $\mathbf{d}$ [S5, S6, unofficial].

### Inputs and Outputs
*   **Inputs:** A dataset of $N$ images with known camera poses (extrinsics $\mathbf{C}$ and intrinsics $\mathbf{K}$) and dimensions $H \times W$. The model $\mathcal{F}_\Theta$ (initially random or pre-initialized).
*   **Outputs:** Optimized parameters $\Theta$ that minimize the difference between rendered and ground-truth colors.

### Data Structures
*   **Ray Bundle:** A tensor of shape $[B, 3]$ or $[B, 6]$ containing ray origins $\mathbf{o}$ and directions $\mathbf{d}$ for a batch of $B$ rays. If view-dependent effects are modeled, directions are included [S5, S14, unofficial].
*   **Target Colors:** A tensor of shape $[B, 3]$ containing the ground-truth RGB values sampled from the input images at the ray origins' pixel coordinates [S5, S14, unofficial].
*   **Pre-computed Rays (Optional):** A flat array of shape $[(N-1)HW, 3, 3]$ storing flattened pixel indices, ray origins, ray directions, and target RGB values, shuffled for efficient batching [S5, S14, unofficial].

### Numbered Steps
1.  **Initialization:** Initialize the neural network parameters $\Theta$ (e.g., zeros or small random values) and the optimizer state (e.g., Adam) [S5, S14, S15, unofficial]. If using a coarse-to-fine architecture, initialize both coarse and fine networks [S11, unofficial].
2.  **Ray Sampling:** For each iteration, sample a batch of $B$ rays. This is typically done by randomly selecting pixel coordinates $(u, v)$ from the training images and computing the corresponding ray origins and directions using the camera matrices [S5, S6, S14, unofficial].
    *   *Edge Case:* For forward-facing scenes, rays are often normalized to NDC (Normalized Device Coordinates) space with near/far bounds (e.g., near=0, far=1) [S2, S5, unofficial]. For non-forward-facing scenes, bounds are computed from the scene geometry (e.g., bounding box depths) [S2, S5, unofficial].
3.  **Volume Integration (Forward Pass):** For each ray in the batch, sample $K$ points along the ray segment between near and far bounds. Query the neural network for density $\sigma_i$ and color $\mathbf{c}_i$ at each point [S5, S6, unofficial]. Compute the rendered color $\hat{C}(\mathbf{r})$ using the standard emission-absorption model:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=1}^K T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $\delta_i$ is the distance between adjacent samples, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [S5, S6, unofficial].
    *   *Coarse-to-Fine:* If using a two-stage approach, first render with coarse samples, then resample more densely around high-weight regions (based on coarse weights) for the fine network [S5, S11, unofficial].
4.  **Loss Computation:** Compute the Mean Squared Error (MSE) between the rendered color $\hat{C}(\mathbf{r})$ and the ground-truth color $\mathbf{C}(\mathbf{r})$ [S5, S6, S14, unofficial]:
    $$
    \mathcal{L}_{\text{MSE}} = \frac{1}{B} \sum_{b=1}^B \| \hat{C}(\mathbf{r}_b) - \mathbf{C}(\mathbf{r}_b) \|^2
    $$
    *   *Variants:* Additional losses may be added, such as Total Variation (TV) loss on the density field to encourage smoothness, or Eikonal loss for SDF-based methods [S7, unofficial][S8]. For ToF data, a composite loss including phasor error is used [S19, unofficial].
5.  **Backpropagation and Update:** Compute gradients $\nabla_\Theta \mathcal{L}$ via automatic differentiation and update the parameters using the optimizer (typically Adam) [S5, S6, S14, unofficial].
6.  **Learning Rate Scheduling:** Adjust the learning rate according to a schedule, such as exponential decay or cosine annealing, to stabilize convergence in later stages [S5, unofficial][S8][S11, unofficial].
7.  **Logging and Checkpointing:** Periodically log loss metrics (PSNR, SSIM) and save model checkpoints [S5, S11, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nerf(model, optimizer, scheduler, dataset, num_iterations, batch_size, render_kwargs):
    """
    Standard NeRF training loop.
    
    Args:
        model: Neural Radiance Field model (MLP).
        optimizer: Optimizer (e.g., Adam).
        scheduler: Learning rate scheduler.
        dataset: Iterable yielding (images, poses, intrinsics).
        num_iterations: Total number of training steps.
        batch_size: Number of rays per batch (N_rand).
        render_kwargs: Dictionary containing near, far, N_importance, perturb, etc.
    """
    # Pre-compute rays if batching is used (optional optimization)
    # rays_rgb = precompute_rays(dataset) 
    
    for i in range(num_iterations):
        # 1. Ray Sampling
        # Option A: Sample from dataset directly
        images, poses, intrinsics = next(dataset)
        # Sample random pixel coordinates (u, v)
        u, v = random_pixel_coords(images.shape[1], images.shape[2], batch_size)
        # Convert to ray origins and directions
        rays_o, rays_d = get_rays_from_pixels(u, v, poses, intrinsics)
        # Get ground truth colors
        target_colors = sample_colors_at_pixels(images, u, v)
        
        # Option B: Sample from pre-computed batch (if implemented)
        # rays_o, rays_d, target_colors = sample_from_batch(rays_rgb, batch_size)

        # Stack into batch tensor
        batch_rays = torch.stack([rays_o, rays_d], dim=0) # Shape [2, B, 3]
        
        # 2. Forward Pass (Volume Rendering)
        # Render the batch of rays
        # Returns: rgb (B, 3), extras (dict containing coarse rgb, weights, etc.)
        rgb, extras = model.render(batch_rays, **render_kwargs)
        
        # 3. Loss Computation
        # Primary photometric loss
        loss = torch.nn.functional.mse_loss(rgb, target_colors)
        
        # Optional: Coarse-to-fine loss
        if 'rgb_coarse' in extras:
            loss += torch.nn.functional.mse_loss(extras['rgb_coarse'], target_colors)
            
        # Optional: Regularization (e.g., TV loss, Eikonal loss)
        # loss += tv_loss(model) * tv_weight
        
        # 4. Backpropagation and Update
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # 5. Learning Rate Scheduling
        scheduler.step()
        
        # 6. Logging
        if i % log_interval == 0:
            psnr = -10 * torch.log10(loss.item())
            print(f"Iter {i}: Loss {loss.item():.6f}, PSNR {psnr:.2f}")
            
        # 7. Checkpointing
        if i % checkpoint_interval == 0:
            save_checkpoint(model, optimizer, i)
```

### Parameters and Typical Values
*   **Learning Rate (lr):** $5 \times 10^{-4}$ to $1 \times 10^{-3}$ [S11, S13, unofficial].
*   **Batch Size (N_rand):** 1024 to 2048 rays per iteration [S5, S11, unofficial].
*   **Number of Iterations:** 100,000 to 1,000,000 [S11, S14, unofficial].
*   **Samples per Ray (N_samples):** 128 (coarse) + 128 (fine) [S5, S11, unofficial].
*   **Positional Encoding Frequency (L):** 10 for position, 4 for direction [S11, unofficial].
*   **Near/Far Bounds:** Near=2.0, Far=6.0 for Blender synthetic data; computed from scene bounds for real data [S2, S5, unofficial].
*   **Perturbation:** Jitter amount for stratified sampling (0.0 to 1.0) to prevent aliasing [S2, S5, unofficial].

### Edge Cases and Pitfalls
*   **VRAM Overflow:** Rendering large batches can exceed GPU memory. Mitigated by chunking rays during the forward pass [S5, S14, unofficial].
*   **Gradient Freeing:** Higher-order derivatives (e.g., for gradient-based logging) must be computed before `loss.backward()` to avoid freeing the autograd graph [S3, unofficial].
*   **Resuming Training:** When loading checkpoints, the optimizer state and scheduler last epoch must be restored to ensure correct scheduling [S9, unofficial].
*   **Instability in Dynamic Growth:** In growing NeRF variants, randomly initialized sub-networks can collapse to zero density. Initializing grown sub-network weights to match the parent network prevents this [S1].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
The differentiable forward pass that synthesizes RGB images and depth maps from a Neural Radiance Field (NeRF) during optimization. It solves the problem of computing gradients for the neural network parameters by sampling points along camera rays, querying volume density and color, and integrating them via alpha compositing. It sets itself apart from evaluation rendering by using stochastic sampling with perturbation and adaptive step sizes to facilitate gradient-based optimization, whereas evaluation uses deterministic rendering for reproducible metrics.

### [Training Configuration](2332-training-configuration.md)
Training Configuration defines the static and dynamic hyperparameters, optimization schedules, and environment settings required to converge a differentiable volume rendering model. It solves the problem of balancing convergence speed, reconstruction fidelity, and computational stability by specifying optimizer states, learning rate decay strategies, batch sampling rates, and regularization weights. It sets itself apart from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the gradient descent mechanics rather than the forward rendering pass, and from [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md) by defining the temporal evolution of the optimization state rather than the initial spatial representation.

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
Regularization and Loss Functions define the objective landscape for optimizing Neural Radiance Fields (NeRFs), extending the standard photometric reconstruction loss to address specific failure modes such as overfitting in sparse views, geometric ambiguity ("floaters"), semantic incoherence, and style mismatches. This family sets itself apart from general optimization procedures by providing specialized mathematical formulations that constrain the density $\sigma$ and color $\mathbf{c}$ fields, or the rendered outputs, to adhere to prior knowledge (geometric, semantic, or statistical) rather than relying solely on pixel-wise alignment with ground truth images.

### [KNN-NeRF](8672-knn-nerf.md)
KNN-NeRF is a baseline method for training Neural Radiance Fields (NeRFs) using sparse 3D point clouds as supervision instead of multi-view images. It solves the problem of generating ground-truth color labels for rays that do not pass through existing points by assigning the color of the nearest point in the cloud to the ray. Unlike standard NeRF which relies on pixel colors from images, KNN-NeRF constructs a dense training signal from sparse geometry, though it struggles with sharp object borders due to the low-frequency bias of neural networks and the sparsity of the input.

### [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md)
Progressive and Dynamic Scene Growth refers to a family of training strategies that expand the capacity, data, or topology of a neural radiance field during optimization, rather than optimizing a static representation on a fixed dataset. This family solves the problems of representing complex, large-scale, or dynamic scenes with limited initial data, and mitigates issues such as "foggy" densities, floating artifacts, and catastrophic forgetting in online settings. It sets itself apart from static NeRF variants by coupling the network architecture or training data directly to the training error or temporal sequence, growing $k$ new branches per step based on uncertainty, iteratively expanding reference views for panoramic scenes [8666-progressive-view-synthesis-training.md], or continuously updating the field as new camera views arrive [12066-online-incremental-nerf-optimization.md].

### [Adaptive Ray Sampling](8870-adaptive-ray-sampling.md)
Adaptive ray sampling refers to a family of training procedures in neural radiance fields (NeRF) that replace uniform ray sampling with strategies that concentrate computational resources on regions of high geometric or photometric complexity. It solves the inefficiency of uniform sampling, which wastes evaluations on empty space or smooth regions, by dynamically adjusting sample locations or ray selection based on density, error, depth priors, or spatial partitions. This family is distinguished from [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on *where* to sample rather than *how* to integrate, and from [Coarse-to-Fine NeRF](3406-coarse-to-fine-nerf.md) by encompassing broader strategies including error-based mining, depth-guided selection, and superpixel partitioning, not just hierarchical point clustering.

### [Sparse View and Initialization Methods](9394-sparse-view-and-initialization-methods.md)
This family addresses the degradation of NeRF quality—such as blurred geometry, slow convergence, and floaters—when trained on sparse input views (typically 15–20 images or fewer than 50–100 views). It distinguishes itself from standard NeRF training by decoupling the initialization and regularization processes from the default random or empty-space assumptions, instead leveraging external geometric scaffolds, semantic masks, or visibility priors to anchor the optimization.

### [NeRV Training](10253-nerv-training.md)
NeRV training optimizes a continuous volumetric representation that decouples scene geometry (shape) from material appearance (reflectance) and lighting interactions (visibility) to enable efficient relighting and view synthesis. It differs from standard NeRF training by introducing a separate "visibility MLP" that predicts ray visibility and termination depth, supervised by consistency terms with the primary shape MLP rather than ground-truth labels, allowing the simulation of direct and one-bounce indirect illumination within the training loop.

### [Aug-NeRF](10294-aug-nerf.md)
Aug-NeRF is a training framework for Neural Radiance Fields (NeRF) that injects worst-case adversarial perturbations into three distinct levels of the pipeline—input coordinates, intermediate features, and pre-rendering output—to implicitly regularize the model with physical grounds. It addresses the generalization gap in NeRF where interpolation between seen views yields inconsistent and non-smooth geometries. Unlike explicit regularizers like Laplacian or Total Variation which constrain MLP flexibility aggressively, Aug-NeRF uses data-adaptive smoothness priors to enhance generalization to unseen views without sacrificing representation flexibility.

### [Scene Representation and Architecture](11368-scene-representation-and-architecture.md)
Scene Representation and Architecture defines the neural network topologies, data structures, and input encodings used to approximate the continuous radiance field in differentiable volume rendering. While the canonical NeRF uses a monolithic Multi-Layer Perceptron (MLP) to map 6D coordinates to density and color, this family encompasses methods that modify the network topology, input representation, or underlying data structures to address limitations in rendering speed, memory consumption, generalization, dynamic scene modeling, and single-view synthesis. These architectures set themselves apart from the canonical baseline by introducing mechanisms such as sparse grids, tensor decompositions, transformer attention, or explicit geometric priors, enabling applications ranging from real-time rendering to 3D-aware image generation.

### [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)
This family of methods accelerates NeRF training and rendering by reducing the computational cost of evaluating the neural radiance field. It addresses the bottleneck of querying high-dimensional MLPs for every sample point along every ray. Unlike standard uniform or adaptive sampling over the entire scene bounding box, these methods restrict evaluation to sparse structures (visual hulls) or use lightweight approximations (caches) to predict radiance, thereby reducing the number of expensive neural network queries.

### [Camera Pose and Extrinsic/Intrinsic Optimization](12051-camera-pose-and-extrinsic-intrinsic-optimization.md)
This family of methods addresses the problem of optimizing camera parameters (extrinsics and intrinsics) simultaneously with Neural Radiance Field (NeRF) weights, eliminating the reliance on pre-computed Structure-from-Motion (SfM) pipelines like COLMAP. By treating camera poses as learnable parameters, these methods enable 3D reconstruction from sparse or uncalibrated image sets. What sets this family apart from siblings like [Time-dependent NeRF with Continuous Camera Motion](12399-time-dependent-nerf-with-continuous-camera-motion.md) is the explicit optimization of discrete camera poses per image rather than continuous motion functions, and from [Camera Preconditioning](12545-camera-preconditioning.md) by focusing on the joint gradient-based update of poses and scene geometry rather than preconditioning the optimizer for faster convergence.

### [Monocular Depth Estimation Adaptation and Supervision](12353-monocular-depth-estimation-adaptation-and-supervision.md)
This family of methods integrates pre-trained Monocular Depth Estimation (MDE) networks into the NeRF training loop to resolve the shape-radiance ambiguity and scale uncertainty inherent in sparse-view or single-view reconstruction. Unlike standard NeRF which relies solely on photometric consistency, these methods use MDE outputs as geometric priors to guide volume sampling, regularize density distributions, or directly adapt the MDE network to the scene's absolute geometry. They are distinguished from sibling methods like [DäRF](12331-d-rf.md) by their specific strategies for handling the relative nature of monocular depth, ranging from sparse prior fine-tuning to distribution-based Earth Mover’s Distance supervision [11911-depth-guided-nerf-training-via-earth-mover-s-distance.md].

### [NeRF Editing and Inpainting Procedures](12494-nerf-editing-and-inpainting-procedures.md)
This family of procedures modifies a pre-trained or training Neural Radiance Field (NeRF) to alter its content, such as removing objects, changing colors, or filling occluded regions, without requiring full retraining from scratch. Unlike standard training loops that optimize for photometric consistency on input views, these methods introduce external priors (2D inpainting, diffusion models, or semantic features) and specialized loss functions or rendering pipelines to enforce 3D consistency while applying edits. They differ from sibling methods like [Progressive and Dynamic Scene Growth](8869-progressive-and-dynamic-scene-growth.md) by focusing on content modification rather than scene expansion, and from [Regularization and Loss Functions](8586-regularization-and-loss-functions.md) by integrating specific editing objectives (e.g., semantic segmentation, text prompts) into the optimization or rendering process.

### [FlipNeRF Training Pipeline](12501-flipnerf-training-pipeline.md)
FlipNeRF is a training procedure for few-shot novel view synthesis that enhances the mip-NeRF representation by introducing flipped reflection rays into the training loop. It distinguishes itself from standard NeRF pipelines by enforcing consistency between original and geometrically flipped rays through specific uncertainty-aware and bottleneck feature losses, rather than relying solely on color reconstruction.

### [NeRF Ensemble with Conjoint Pruning](12608-nerf-ensemble-with-conjoint-pruning.md)
NeRF Ensemble with Conjoint Pruning is a training strategy that improves novel view synthesis quality by combining two Neural Radiance Field (NeRF) models of different grid resolutions, specifically using the Direct Voxel Grid Optimization (DVGO) architecture. It addresses the high computational requirements and optimization difficulties of fine-grained voxel grids by achieving superior performance compared to single models while maintaining a similar parameter count and memory footprint. Unlike vanilla ensembling, which may yield suboptimal results, this method employs a specific three-stage process involving independent training, fine-tuning, and conjoint pruning to prevent performance degradation.

### [MLI-NeRF](12709-mli-nerf.md)
MLI-NeRF is a two-stage optimization procedure that learns an intrinsic-aware Neural Radiance Field (NeRF) capable of novel view synthesis, relighting, and intrinsic decomposition (separating reflectance and shading) without relying on statistical priors or ground-truth intrinsic labels. It differs from standard NeRF variants by explicitly modeling multiple light sources to generate physics-based pseudo-labels, and it differs from prior intrinsic NeRF methods (like [IntrinsicNeRF](10253-nerv-training.md)) by leveraging 3D geometric constraints and multi-light visibility rather than chromaticity or semantic statistical constraints. The method solves the ill-posed nature of inverse rendering by decoupling the geometry learning (Stage 1) from the intrinsic component separation (Stage 2), using pseudo-labels derived from the Stage 1 model to guide the decomposition.

### [Diffusion-guided Optimization](13230-diffusion-guided-optimization.md)
Diffusion-guided optimization methods enhance Neural Radiance Field (NeRF) training by integrating pretrained 2D or 3D diffusion models as priors to regularize geometry, fill missing views, or generate novel 3D content from text. Unlike standard NeRF training which relies solely on photometric reconstruction losses from observed images, these methods leverage the generative knowledge of diffusion models to constrain the optimization landscape. They are distinguished from sibling methods like [KNN-NeRF](8672-knn-nerf.md) or [Cross-view Self-enhancement](13434-cross-view-self-enhancement.md) by their explicit use of score distillation or latent diffusion losses rather than nearest-neighbor statistics or self-supervised consistency alone.

### [Application-Specific Methods](13294-application-specific-methods.md)
Application-Specific Methods adapt the general NeRF training loop to solve domain-specific inverse problems by modifying the neural field architecture, loss functions, and optimization strategy. Unlike standard NeRF, which optimizes a single MLP for photometric reconstruction, these pipelines decompose fields to isolate physical properties, enforce domain priors (geometric, semantic, or physical), and employ iterative multi-stage optimization to handle complex constraints.

### [Cross-view Self-enhancement](13434-cross-view-self-enhancement.md)
Cross-view Self-enhancement (CSE) is a training strategy for semantic NeRFs that replaces view-inconsistent 2D semantic labels (e.g., from CLIP) with 3D-consistent pseudo-maps rendered from the model itself. It addresses the ambiguity where 2D vision-language models fail to maintain consistent object identities across diverse perspectives, which degrades multi-view consistent rendering. Unlike methods that rely solely on external supervision, CSE leverages the inherent 3D consistency of NeRFs to iteratively refine and synthesize semantic supervision, ensuring that the semantic field learns coherent structures across all views.

### [NeRF-Art](13438-nerf-art.md)
NeRF-Art is a text-guided method for stylizing pre-trained Neural Radiance Fields (NeRFs) by simultaneously altering both appearance and geometry without requiring mesh guidance. It distinguishes itself from prior approaches by employing a hybrid global-local contrastive learning strategy based on CLIP, which ensures strong style adherence and cross-view consistency, addressing the insufficiency of direct CLIP similarity constraints.

### [Neural Implicit Surfaces with PBR](13481-neural-implicit-surfaces-with-pbr.md)
This method integrates a Signed Distance Function (SDF) with a Physically-Based Rendering (PBR) material model to achieve accurate scene representation, explicitly separating specular and diffuse reflections. It differs from standard NeRF by deriving normals from the SDF gradient rather than direct optimization, and from basic SDF methods by employing a precomputed BRDF LUT for specular calculation. It builds on [NeRF Training Loop](970-nerf-training-loop.md) but replaces the generic radiance field with a geometry-aware implicit surface representation.

### [Block-NeRF](13537-block-nerf.md)
Block-NeRF is a method for city-scale NeRF-based reconstruction from street-level images that handles transient appearance and moving objects by dividing neighborhoods into overlapping blocks. It scales NeRF to city-scale scenes by spatially decomposing the environment into manageable blocks, training independent NeRFs per block, and compositing them at render time. It sets itself apart from standard NeRF by enabling the reconstruction of large outdoor environments (e.g., San Francisco's Mission Bay) that exceed the memory and computational limits of a single global model, while specifically addressing the challenge of transient objects (pedestrians, cars) common in street-level imagery.

### [NeRF in the Wild](13541-nerf-in-the-wild.md)
NeRF in the Wild (NeRF-W) extends standard neural radiance fields to handle unconstrained photo collections by modeling per-image appearance variations and transient objects. It addresses the limitations of original NeRF, which assumes static geometry and consistent lighting, by introducing latent embeddings for each input image.

### [ContraNeRF](13659-contranerf.md)
ContraNeRF is a generalizable neural radiance field method designed to improve synthetic-to-real novel view synthesis by introducing geometry-aware contrastive learning to learn multi-view consistent features with geometric constraints, and cross-view attention to enhance geometry perception. It sets itself apart from vanilla NeRF by eliminating per-scene optimization, allowing it to generalize to unseen scenes, and distinguishing itself from other generalizable methods by explicitly addressing the sharp but inaccurate volume density artifacts common in synthetic-to-real transfer through contrastive geometry constraints.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, multiple jittered rays are sampled uniformly at random through the spatial footprint of the pixel. The mean of the rendered values is used as the predicted color in the loss function [S21]. This approach relies on 'oracle' scale information that does not exist for most datasets and performs worse than mip-NeRF while being significantly more expensive [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets. Phase 2 trains all network components end-to-end, starting with a linear warmup for 1,000 SGD steps to reduce shock from switching regimes [S22, unofficial]. This two-stage approach is consistently effective where training from scratch often fails [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S23].

### JaxNeRF
A re-implementation of the original NeRF in JAX, used as a benchmark comparison for early works seeking to improve on training and rendering speed [S24]. It is slightly faster and more suited for distributed computing than the original TensorFlow implementation of NeRF [S24].

### Gradient Detachment in Training
A training strategy where the object field component is detached from the backbone to prevent gradient backpropagation to the rendering part, ensuring independent optimization [S25, unofficial]. With detachment, gradients do not backpropagate to the NeRF backbone, achieving better scene rendering and decomposition quality. Without detachment, gradients from the object field component backpropagate to the NeRF backbone, causing mutual influence and a significant drop in rendering quality [S25, unofficial]. The optimal hyper-parameter for empty 3D point losses ($\Delta d$) is found to be 0.05 meters [S25, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot K \cdot C)$ per iteration, where $N$ is the batch size (number of rays), $K$ is the number of samples per ray, and $C$ is the cost of the neural network forward pass [S5, S14, unofficial].
*   **Memory Complexity:** $O(N \cdot K)$ to store intermediate rendering states (transmittance, weights) for backpropagation, plus $O(|\Theta|)$ for model parameters [S5, S14, unofficial].
*   **Measured Performance:** Training typically requires 100,000 to 1,000,000 iterations, taking several hours to days depending on hardware and scene complexity [S11, S14, unofficial]. For example, training on 199 down-sampled images with a batch size of 1024 results in ~7,800 iterations per epoch [S17, unofficial].
*   **Scaling:** Performance scales with batch size, but larger batches hinder data loading performance and may reduce throughput due to GPU occupancy limits. A batch size of $2^{10}$ (1024) is often chosen as a trade-off [S15, unofficial].
*   **Strengths:** Simple to implement, leverages standard deep learning frameworks, produces high-quality novel views when sufficient input views are available.
*   **Limitations:** Slow convergence, prone to overfitting and "floaters" in sparse views, high memory usage for large scenes, and sensitivity to hyperparameters [S1][S17, unofficial].
*   **When to Use:** Standard multi-view reconstruction with dense input views and static scenes.
*   **When Not to Use:** Real-time rendering requirements, sparse view scenarios (without extensions), or dynamic scenes (without temporal extensions).

## Background
The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, unofficial]. It builds on the concept of coordinate-based neural networks for implicit scene representation [S3, unofficial][S20]. The method leverages the differentiability of volume rendering to optimize the neural network parameters via gradient descent, minimizing the photometric error between rendered and observed images [S5, S6, unofficial]. It has become a foundational technique in 3D computer vision, inspiring numerous variants for acceleration, generalization, and dynamic scene modeling [S1, S24].

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
[S24] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S25] DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images, https://arxiv.org/abs/2208.07227v2, fetched 2026-10-26 (unofficial)
