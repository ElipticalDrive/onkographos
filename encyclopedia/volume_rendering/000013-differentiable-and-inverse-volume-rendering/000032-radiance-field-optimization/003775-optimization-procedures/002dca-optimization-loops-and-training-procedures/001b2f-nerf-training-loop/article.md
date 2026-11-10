# NeRF Training Loop

Also called: IPE (Interatomic Positional Encoding), Interatomic Positional Encoding, NeRF Optimization Loop, NeRF Training Procedure, Neural Radiance Field (NeRF) Training

## Overview

The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric error between rendered images and ground-truth observations by updating the parameters of a continuous volumetric radiance field [S5, S6, S14, unofficial]. It solves the inverse problem of reconstructing 3D geometry and appearance from 2D images by leveraging the differentiability of the volume rendering equation [44-radiance-field-optimization.md]. What sets it apart from standard volume rendering is the inclusion of stochastic ray sampling, gradient-based weight updates, and specific learning rate schedules designed to handle the non-convex, high-dimensional landscape of neural radiance fields [S5, S14, unofficial].

## How it works

**Inputs and Outputs:**
*   **Inputs:** A set of $N$ training images with known camera poses (extrinsics $P$ and intrinsics $K$), a neural network model $F_\Theta$ (typically an MLP) parameterized by $\Theta$, and hyperparameters (learning rate, batch size, number of iterations).
*   **Outputs:** Optimized network weights $\Theta^*$ and optionally optimized camera poses if included in the optimization [S2, S14, unofficial].

**Data Structures:**
*   **Rays:** A batch of rays is represented as a tensor of shape $[B, 3, 3]$ or $[B, 6]$ depending on implementation, where $B$ is the batch size (number of rays). The first two channels (or first 6 values) store the ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized), and the third channel (or last 3 values) stores the target RGB color $\mathbf{c}_{gt}$ [S5, S14, unofficial].
*   **Pre-computed Rays:** To accelerate data loading, rays and targets are often pre-computed into a flat array `rays_rgb` of shape $[(N-1) \cdot H \cdot W, 3, 3]$ (or similar flattening) containing all possible ray-target pairs from the training set, shuffled at the start of each epoch [S5, S14, unofficial].
*   **Checkpoint:** A dictionary storing `model` (state dict), `optimizer` (state dict), and `epoch` or `step` counter to allow resumption [S10, S11, unofficial].

**Numbered Steps:**

1.  **Initialization:**
    *   Load training and validation datasets [S9, S11, unofficial].
    *   Initialize the neural network model $F_\Theta$ (e.g., with He initialization or SIREN-specific initialization) [S3, S15, unofficial].
    *   Initialize the optimizer (typically Adam) with a base learning rate $lr$ [S5, S11, unofficial].
    *   Initialize a learning rate scheduler (e.g., Exponential Decay or Cosine Annealing) [S5, S11, unofficial].
    *   If using a coarse-to-fine strategy, initialize both coarse and fine networks [S11, unofficial].

2.  **Iteration Loop:** For iteration $i$ from 1 to $N_{iters}$:
    *   **Ray Sampling:**
        *   Select a batch of $N_{rand}$ rays from the pre-computed `rays_rgb` array or sample random pixel coordinates from training images [S5, S14, unofficial].
        *   If `use_batching` is true, slice the pre-computed array; otherwise, randomly select an image and sample $N_{rand}$ pixels without replacement [S5, S14, unofficial].
        *   Construct the ray bundle: origins $\mathbf{o}$, directions $\mathbf{d}$, and target colors $\mathbf{c}_{gt}$ [S7, S14, unofficial].
        *   (Optional) Apply perturbations to sampling positions if `perturb > 0` to break symmetry and improve reconstruction [S5, S2, unofficial].
    *   **Forward Pass (Rendering):**
        *   Sample $K$ points along each ray within the near/far bounds [S5, S14, unofficial].
        *   Query the neural network $F_\Theta$ at these points to obtain volume density $\sigma$ and color $\mathbf{c}$ [S5, S6, unofficial].
        *   Compute the rendered color $\hat{\mathbf{C}}$ using the differentiable volume rendering equation (DVR) [44-radiance-field-optimization.md]:
            $$ \hat{C}(\mathbf{r}) = \sum_{i=1}^K T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
            where $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ [44-radiance-field-optimization.md].
        *   (Optional) If a coarse network was used, store its output $\hat{\mathbf{C}}_{coarse}$ for auxiliary loss [S11, unofficial].
    *   **Loss Computation:**
        *   Compute the Mean Squared Error (MSE) between the rendered color $\hat{\mathbf{C}}$ and the ground truth $\mathbf{c}_{gt}$ [S5, S14, unofficial]:
            $$ \mathcal{L}_{rgb} = \frac{1}{B} \sum_{b=1}^B \|\hat{\mathbf{C}}_b - \mathbf{c}_{gt,b}\|^2 $$
        *   (Optional) Add auxiliary losses, such as coarse network loss $\mathcal{L}_{coarse}$, regularization terms (e.g., Eikonal, sparsity), or depth losses [S8][S11, S16, unofficial].
        *   Total loss: $\mathcal{L} = \mathcal{L}_{rgb} + \lambda_{coarse}\mathcal{L}_{coarse} + \dots$ [S11, S16, unofficial].
    *   **Backpropagation and Update:**
        *   Zero gradients: `optimizer.zero_grad()` [S18, unofficial].
        *   Compute gradients: `loss.backward()` [S18, unofficial].
        *   Update weights: `optimizer.step()` [S18, unofficial].
        *   Update learning rate according to the scheduler [S9, S18, unofficial].
    *   **Logging and Checkpointing:**
        *   Log loss and PSNR metrics every $I_{print}$ steps [S3, S14, unofficial].
        *   Render validation images every $I_{img}$ steps [S2, S11, unofficial].
        *   Save model checkpoint every $I_{weights}$ steps [S2, S11, unofficial].

**Parameters with Typical Values:**
*   $N_{rand}$ (Batch Size): 1024 or 2048 rays per iteration [S5, S11, S14, unofficial].
*   $lr$ (Learning Rate): $5 \times 10^{-4}$ for Adam [S11, unofficial], or $0.01$ for SGD [S17, unofficial].
*   $N_{iters}$ (Iterations): 100,000 to 1,000,000 steps [S11, S14, unofficial].
*   $N_{samples}$ (Samples per ray): 128 (coarse) + 128 (fine) [S11, S17, unofficial].
*   Decay Rate: 0.1 [S6, S18, unofficial].
*   Decay Steps: $args.lrate\_decay \times 1000$ [S5, S18, unofficial].

**Edge Cases and Pitfalls:**
*   **VRAM Overflow:** Rendering all rays at once can exceed GPU memory. Mitigated by chunking rays into smaller batches during the forward pass (`chunk` argument) [S5, S14, unofficial].
*   **Gradient Graph Freeing:** Higher-order derivatives (e.g., for logging gradients or Laplacians) must be computed *before* calling `loss.backward()`, as the backward pass frees the autograd graph [S3, unofficial].
*   **Resuming Training:** When loading a checkpoint, the learning rate scheduler's last epoch must be set to `start_epoch - 1` to ensure correct scheduling upon resumption [S9, unofficial].
*   **Empty Checkpoints:** If a checkpoint is loaded but the step counter is 0, it is treated as an empty checkpoint, and training should restart or error out [S15, unofficial].
*   **Instability in Growth:** In dynamic growing networks, randomly initialized sub-networks can collapse to zero density. Mitigated by initializing alpha weights of grown sub-networks to match the parent [S1].

## Subtopics

### [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md)
Training Ray Marching and Integration is the differentiable forward pass that synthesizes RGB images from a neural radiance field (NeRF) during optimization. It solves the problem of computing gradients for scene parameters by sampling points along camera rays, querying a neural network for volume density and color, and integrating them via alpha compositing. It sets itself apart from evaluation by using stochastic perturbations, adaptive sampling strategies (e.g., occupancy grid pruning), and chunked processing to manage memory, whereas evaluation disables these for deterministic results.

### [Training Configuration](2332-training-configuration.md)
Training Configuration defines the static and dynamic hyperparameters, optimizer settings, and schedule strategies required to stabilize and accelerate the differentiable volume rendering optimization loop. It solves the problem of divergent or slow convergence in non-convex radiance field objectives by managing learning rates, batch sizes, sampling densities, and regularization weights. What sets this method apart from its siblings is that it provides the *initialization and scheduling* parameters (e.g., base learning rates, decay schedules, initial alpha values) rather than the execution logic ([Training Loop Setup](1522-training-loop-setup.md)) or the specific loss formulations ([Regularization and Loss Functions](8586-regularization-and-loss-functions.md)).

### [Neural Volume Scene Setup](4168-neural-volume-scene-setup.md)
Constructs a Mitsuba scene containing a volumetric cube and a custom neural radiance field integrator, initializing the density grid and spherical harmonics coefficients. It differs from generic training setups by directly binding the neural network parameters to the scene graph's heterogeneous volume and integrator components.

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
Camera Pose and Parameter Optimization refers to the family of methods that jointly optimize the parameters of a Neural Radiance Field (NeRF) and the camera poses (extrinsics) and/or intrinsics (e.g., focal length) from a set of input images. It solves the problem of reconstructing 3D scenes when camera parameters are unknown, inaccurate, or when the system requires online pose estimation for navigation. Unlike baseline NeRF, which relies on pre-estimated poses from Structure-from-Motion (SfM) pipelines like COLMAP, these methods treat camera parameters as learnable variables updated via gradient descent alongside the NeRF weights. This family encompasses static scene reconstruction with unknown poses, hybrid 2D/3D registration, and online robot navigation filters.

### [Regularization and Loss Functions](8586-regularization-and-loss-functions.md)
This family encompasses the mathematical terms added to the total loss function during Neural Radiance Field (NeRF) optimization to constrain the solution space, improve geometric accuracy, and mitigate artifacts such as floaters or over-saturation. It solves the ill-posed nature of inverse volume rendering, where photometric consistency alone often leads to ambiguous geometries or poor generalization in sparse-view settings. What sets this family apart from its siblings, such as [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) (which handles the forward rendering process) and [NeRF Training Loop](970-nerf-training-loop.md) (which manages the optimization schedule), is the specific definition of the error metrics and priors applied to the density $\sigma$ and color $\mathbf{c}$ fields.

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

### [Aug-NeRF](10294-aug-nerf.md)
Aug-NeRF is a training framework for Neural Radiance Fields (NeRF) that injects worst-case adversarial perturbations into three distinct levels of the pipeline—input coordinates, intermediate features, and pre-rendering output—to implicitly regularize the model with physical grounds. It addresses the generalization gap in NeRF where interpolation between seen views yields inconsistent and non-smooth geometries. Unlike explicit regularizers like Laplacian or Total Variation which constrain MLP flexibility aggressively, Aug-NeRF uses data-adaptive smoothness priors to enhance generalization to unseen views without sacrificing representation flexibility.

### [Cross-Guided Optimization of Radiance Fields (CROP)](10371-cross-guided-optimization-of-radiance-fields-crop.md)
Cross-Guided Optimization of Radiance Fields (CROP) is a framework that performs multi-view image super-resolution (MVSR) on training-view images during the radiance field optimization process to improve high-resolution novel view synthesis (HRNVS). It fuses single-image super-resolution (SISR) features with voxel-based uncertainty fields to ensure multi-view consistency and high-frequency details. Unlike methods that rely solely on the spectral characteristics of coordinate-based networks for upscaling, CROP explicitly bridges 2D SISR and 5D radiance fields by iteratively refining input images based on rendering errors.

### [Application-Specific Pipelines](11200-application-specific-pipelines.md)
Application-Specific Pipelines are specialized training and optimization procedures tailored to distinct downstream tasks, diverging from the standard NeRF reconstruction loop to address specific constraints such as monocular pose estimation or text-guided scene editing. Unlike the general [NeRF Training Loop](970-nerf-training-loop.md) which optimizes a single radiance field for novel view synthesis, these pipelines decouple or augment the optimization objectives to handle tasks like 6-DoF camera pose recovery in Plenoxels environments or compositional object insertion into existing NeRF scenes.

### [NeRF-GT Baseline Training Setup](11349-nerf-gt-baseline-training-setup.md)
NeRF-GT is a baseline training protocol for Neural Radiance Fields that utilizes High Dynamic Range (HDR) input views to synthesize novel HDR imagery. It addresses the limitation of standard NeRF, which typically trains on Low Dynamic Range (LDR) images, by directly optimizing the radiance field against HDR ground truth. It differs from standard LDR-based training by explicitly handling high dynamic range data, though it faces challenges in encoding all scene areas uniformly due to the intensity variance.

### [Scene Representation and Architecture](11368-scene-representation-and-architecture.md)
Scene Representation and Architecture defines the neural network structures used to approximate the continuous radiance field $F_\Theta: (\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$ in differentiable volume rendering. It solves the problem of representing 3D geometry and appearance as a differentiable function that can be optimized via gradient descent. It sets itself apart from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by focusing on the network topology and input encoding rather than the sampling strategy, and from [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md) by typically assuming a single global representation rather than decomposing the scene.

### [Acceleration and Sparse Structures](11405-acceleration-and-sparse-structures.md)
This family of methods accelerates NeRF training and rendering by reducing the computational cost of evaluating the neural radiance field. It addresses the bottleneck of querying high-dimensional MLPs for every sample point along every ray. Unlike standard uniform or adaptive sampling over the entire scene bounding box, these methods restrict evaluation to sparse structures (visual hulls) or use lightweight approximations (caches) to predict radiance, thereby reducing the number of expensive neural network queries.

### [Deceptive-NeRF/3DGS Pipeline](11860-deceptive-nerf-3dgs-pipeline.md)
Deceptive-NeRF/3DGS is a sparse-view reconstruction method that enhances NeRF or 3D Gaussian Splatting (3DGS) by continuously densifying input observations with synthetic pseudo-observations generated by a diffusion model during training. It distinguishes itself from sibling methods by introducing an uncertainty map derived from view-consistency warping to guide the diffusion process, ensuring that generated pseudo-observations adhere to epipolar constraints and 3D geometry, rather than relying on ensemble learning or variational inference for uncertainty estimation.

### [MELON](11909-melon.md)
MELON is a method for 3D inverse rendering that simultaneously reconstructs a neural radiance field and estimates camera poses from a set of unposed images. It solves the problem of local minima in pose estimation by using an encoder to amortize inference and a "modulo loss" that maps the latent pose space to a quotient set, allowing gradient descent to converge despite rotational symmetries. Unlike adversarial methods such as GNeRF, MELON does not require a discriminator, enabling ab-initio reconstruction with few images.

### [Depth-Guided NeRF Training via Earth Mover’s Distance](11911-depth-guided-nerf-training-via-earth-mover-s-distance.md)
This method improves the geometric understanding of Neural Radiance Fields (NeRFs) in sparse-view settings by supervising the distribution of ray termination distances using Earth Mover’s Distance (EMD) rather than enforcing direct depth matching via L2 loss. It utilizes off-the-shelf monocular depth priors from diffusion models (specifically DiffDP) to provide depth estimates and intrinsic uncertainty maps, weighting the photometric and depth losses to selectively incorporate prior information without propagating prior inaccuracies. Unlike [DäRF](10371-darf.md) which directly supervises rendered depth, or [SCADE](10371-scade.md) which uses multiple ambiguous hypotheses, this approach guides the ray termination distribution to avoid wrong geometry understanding while maintaining photometric quality.

### [Urban Radiance Fields](11973-urban-radiance-fields.md)
Urban Radiance Fields is a method for 3D reconstruction and view synthesis in large-scale urban environments using sparse multi-view images supplemented by LiDAR data. It addresses the challenges of unbounded outdoor scenes, limited camera pose variety, and large-scale geometry by incorporating specific depth, sight, and sky segmentation losses into the standard NeRF training loop. Unlike standard NeRF which relies solely on photometric consistency, this method enforces geometric constraints via LiDAR and semantic constraints via sky segmentation to stabilize training in open environments.

### [Camera Pose and Extrinsic/Intrinsic Optimization](12051-camera-pose-and-extrinsic-intrinsic-optimization.md)

### [DFF (Decomposing NeRF for Editing Via Feature Field Distillation)](12068-dff-decomposing-nerf-for-editing-via-feature-field-distillat.md)
DFF is a method that enables 3D semantic segmentation and editable novel-view synthesis by distilling features from a 2D teacher network (such as LSeg or DINO) into a neural radiance field, allowing the decomposition of scenes into semantic regions. It solves the problem that original NeRF feature spaces are entangled with low-level factors like color or spatial adjacency rather than semantic similarity, making naive decomposition robust. DFF is distinct from sibling methods like [Aug-NeRF](10294-aug-nerf.md) or [Cross-Guided Optimization of Radiance Fields (CROP)](10371-cross-guided-optimization-of-radiance-fields-crop.md) by explicitly training a dedicated feature field via distillation from a pre-trained 2D encoder, rather than relying on intrinsic network features or external geometric constraints alone.

### Supersampling Baseline
A baseline method for evaluating NeRF on multiscale data by training on full-resolution images and using brute-force supersampling during evaluation, or training on multiscale data with supersampling during both training and evaluation [S21]. For every pixel, it casts multiple jittered rays sampled uniformly at random through the spatial footprint of the pixel, renders each ray with NeRF, and uses the mean of those rendered values as the predicted color in the loss function [S21]. This baseline relies on 'oracle' scale information that does not exist for most datasets [S21]. The 16x supersampling baseline performs worse than mip-NeRF and is significantly more expensive during both training and evaluation [S21].

### Two-stage training for proposers
A training schedule split into two equal-duration phases to stabilize learning of a trainable proposer network in NeRF-ID [S22, unofficial]. Phase 1 trains the learnt proposer to mimic the heuristic proposer by minimizing the L2 matching loss between their proposal sets [S22, unofficial]. Phase 2 trains all network components end-to-end, resetting the optimizer state and performing a linear warmup for 1,000 steps to reduce shock from switching regimes [S22, unofficial]. Training from scratch sometimes works but often fails; two-stage training is consistently effective [S22, unofficial].

### Surface-aware density formulation
A method to extract high-quality surfaces from NeRF by modeling geometry as a distance function or occupancy field and converting its output to density using a probability density function [S23]. It incorporates an eikonal term to encourage geometric consistency, enforcing that the gradient norm of the learned SDF is equal to one [S23]. The density distribution is modeled as $\Phi_S(x) = \frac{se^{-sx}}{1+e^{-sx}}$, where $s$ is a scaling parameter and $x$ is the SDF or occupancy value [S23].

### Incremental Registration
A training strategy that registers frames incrementally following temporal order, leveraging the intuition that camera motions in videos are smooth and adjacent poses are close [S24]. At the start of training, it jointly optimizes NeRF parameters $\Theta$ and poses $\{P_1, P_2\}$ from the first two frames $\{I_1, I_2\}$ [S24]. After every $k$ iterations, it adds a new frame $I_i$ and initializes its pose $P_i$ by the previous pose $P_{i-1}$ [S24]. It freezes the learning rate on poses $\{P_i\}$ and NeRF $\Theta$ until all frames are registered, applying a learning rate decay schedule after all $N$ images are added [S24]. Typical values for $k$ are 1k iterations for CO3D/HO3D datasets and 500 iterations for LLFF [S24]. Freeze duration is 100k iterations for HO3D and CO3D, and 30k for LLFF [S24].

### RapNeRF
A NeRF model tailored for view extrapolation, contrasting with standard NeRFs that excel at interpolation [S25]. Training begins with a baseline NeRF to recover a rough mesh [S25]. A Ray Atlas (RA) is created by extracting a rough 3D mesh from a pretrained NeRF and mapping training ray directions onto its vertices [S25]. Random Ray Casting (RRC) is employed: for a training ray hitting a surface point $v = o + tz d$, a backward ray is cast from $v$ toward a new origin $o'$ with uniformly sampled angular perturbations [S25]. RRC and RA augment training rays with a predetermined probability [S25].

### NELF
A neural volumetric rendering framework modeling scene appearance using light transport vectors, enabling realistic relighting and view synthesis of human portraits from only five input images [S25]. A UNet-style CNN extracts per-view features, and an MLP regresses volume density and transport vectors [S25]. Environment maps are estimated to disentangle lighting [S25]. It is trained on synthetic data and adapted to real images via a domain adaptation module [S25].

### Depth Completion Network NeRF
Extracts a sparse point cloud using COLMAP [S25]. Introduces a depth loss based on predicted depth and uncertainty in addition to the standard volumetric loss [S25]. Trains on RGB-D data from ScanNet and Matterport3D with Gaussian noise added to the depth [S25].

### JaxNeRF
A re-implementation of the original NeRF in JAX, used as a benchmark comparison for early works seeking to improve on training and rendering speed [S25]. It is slightly faster and more suited for distributed computing than the original TensorFlow implementation of NeRF [S25].

### SNeS
A method improving geometry by learning probable symmetries for partly symmetric and partly hidden in-scene objects through soft symmetry constraints on geometry and material properties [S25].

### NeRF with Per-Frame Latent Codes
A method using per-frame learned latent codes and adding facial expression as a 76-dimensional coefficient of a morphable model constructed from Face2Face [210] [S25].

### Online Incremental NeRF Optimization
Plans for the next best views, perceives new observations based on the current scene representation, and adds new observations into the training set incrementally to optimize the representation [S26, unofficial]. It differs from offline methods like DVGO by updating the representation online as new views are collected, rather than optimizing based on a fixed, pre-collected set of images [S26, unofficial].

### DreamFusion
A text-to-3D generation method that optimizes a NeRF to render images with high scores from a text-to-image diffusion model, requiring no 3D training data [S27]. Introduced in 2022 [S27].

## Operating characteristics

*   **Time Complexity:** $O(N_{iters} \cdot N_{rand} \cdot K \cdot C)$, where $N_{iters}$ is the number of iterations, $N_{rand}$ is the batch size (rays per step), $K$ is the number of samples per ray, and $C$ is the cost of the neural network forward pass [S5, S14, unofficial].
*   **Memory Complexity:** $O(N_{rand} \cdot K \cdot D)$ for the forward pass, where $D$ is the dimensionality of the network activations, plus $O(N_{iters})$ for optimizer states if not using memory-efficient schedulers [44-radiance-field-optimization.md]. Chunking reduces peak memory usage [S5, S14, unofficial].
*   **Measured Performance:** Training on 128x128 images typically takes 250 epochs or ~100,000 iterations [S4, S11, unofficial]. For example, training on 199 down-sampled images (200x200) with a batch size of 1024 results in ~7,800 iterations per epoch [S17, unofficial].
*   **Scaling:** Performance scales with batch size, but larger batches hinder data loading performance and prevent full GPU occupancy, while smaller batches cap iterations per second [S15, unofficial]. A batch size of $2^{10}$ (1024) is often chosen as a trade-off [S15, unofficial].
*   **Strengths:** Simple to implement, leverages standard deep learning frameworks, effective for high-quality novel view synthesis with sufficient views [S5, S14, unofficial].
*   **Limitations:** Slow convergence due to the non-convex nature of the problem, high memory usage for large scenes or high-resolution images, and sensitivity to hyperparameters like learning rate and sampling strategy [S5, S14, S17, unofficial].
*   **When to Use:** When high-quality novel view synthesis is required and computational resources are available for iterative optimization [S5, S14, unofficial].
*   **When Not to Use:** When real-time rendering is required without pre-training, or when input views are extremely sparse and require specialized initialization [S1][S17, unofficial].

## Background

The NeRF Training Loop was introduced by Mildenhall et al. in "NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis" (2020) [S5, S18, unofficial]. It builds upon the concept of coordinate-based neural networks for implicit representations [S3, S18, unofficial]. The method was initially implemented in TensorFlow [S5, unofficial] and later ported to PyTorch [S6, S14, unofficial]. It has become the standard baseline for neural radiance field optimization, with numerous variants and accelerations developed subsequently [S1][S15, unofficial][S25].

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
[S25] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S26] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
[S27] Differentiable Rendering - 可微渲染：从2D 学习3D, https://www.ultralytics.com/zh/glossary/differentiable-rendering, fetched 2026-10-30
