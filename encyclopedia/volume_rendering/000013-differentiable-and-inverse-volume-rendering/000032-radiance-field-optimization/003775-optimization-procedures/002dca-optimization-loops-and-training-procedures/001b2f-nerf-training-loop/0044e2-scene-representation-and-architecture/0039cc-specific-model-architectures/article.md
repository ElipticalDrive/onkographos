# Specific Model Architectures

Also called: NeRF Architectures, Neural Radiance Field Variants

## Overview
Specific Model Architectures refer to modifications of the canonical Neural Radiance Field (NeRF) MLP structure, sampling strategies, and scene representations to address limitations in training speed, sparse-view generalization, dynamic scene modeling, and generative control. While the parent article [Scene Representation and Architecture](11368-scene-representation-and-architecture.md) defines the standard implicit volumetric representation, these variants alter the underlying data structures (e.g., hash grids, voxels, tensors), the neural network topology (e.g., MIMO MLPs, Transformers), or the optimization objectives (e.g., geometric priors, diffusion losses) to enable real-time rendering, few-shot learning, or explicit 3D manipulation. They differ from sibling methods like [Scene Representation and Partitioning](9444-scene-representation-and-partitioning.md) by focusing on the specific architectural components (network layers, encoding schemes) rather than just spatial decomposition, and from [Specific Model Architectures and Extensions](10278-specific-model-architectures-and-extensions.md) by covering the broader family of distinct architectural paradigms.

## How it works
The implementation of specific NeRF architectures varies significantly by variant. Below are the detailed implementations for the primary methods cited in the sources.

### VaxNeRF (Voxel-Accelerated NeRF)
VaxNeRF accelerates training and inference by restricting ray sampling to a pre-computed visual hull $V$, reducing the number of MLP queries [S1].

**Inputs:** Multi-view images $\{I_i\}$, camera models $\{P_i\}$, coarse foreground-background segmentation masks $\{M_i\}$.
**Outputs:** A single MLP $\mathcal{F}_\theta$ and the visual hull voxel grid $V$.

**Data Structures:**
*   **Visual Hull Grid:** A 3D voxel grid of size $D \times D \times D$. Default $D=400$. Stores integer counts.
*   **Auxiliary Grids:** Two grids $G_1, G_2$ of size $D \times D \times D$, initialized to 0.
*   **MLP:** Single Multi-Layer Perceptron taking position $\mathbf{x}$ and direction $\mathbf{d}$, outputting density $\sigma$ and color $\mathbf{c}$.

**Steps:**
1.  **Visual Hull Construction:**
    1.  Initialize $G_1$ and $G_2$ to zero.
    2.  For each view $I_i$ with mask $M_i$ and camera $P_i$:
        *   Cast rays from foreground pixels in $M_i$. Mark intersected voxels in $G_1$ (increment by 1).
        *   Cast rays from all pixels in $I_i$. Mark intersected voxels in $G_2$ (increment by 1).
    3.  Define $V$ as the set of voxel positions where $G_1 \ge G_2$ and $G_1 > 0$.
    *   *Note:* A moderate dilation is applied to $M_i$ or $V$ to prevent boundary leakage [S1].
2.  **Ray Setup:**
    *   For a novel view, compute external matrix $M_{pose} = R_\theta R_\phi T_r$ and internal matrix $K$.
    *   Compute ray direction in camera frame: $\mathbf{d}_{cam} = ((x - w/2)/f, (y - h/2)/f, -1)^\top$.
    *   Transform to world frame: $\mathbf{d} = M_{pose, 3\times3} \mathbf{d}_{cam}$. Origin $\mathbf{o}$ is the camera center.
3.  **Sampling:**
    *   Sample $N$ points uniformly along the ray segment $[z_{near}, z_{far}]$.
    *   Filter points: Keep only $\mathbf{x}_i$ if $\mathbf{x}_i \in V$.
    *   Default $N \in \{600, 800\}$ [S1].
4.  **Rendering:**
    *   Query MLP for $\sigma_i, \mathbf{c}_i$.
    *   Compute opacity $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$.
    *   Composite color: $C(\mathbf{r}) = \sum_{i: \mathbf{x}_i \in V} T_i \alpha_i \mathbf{c}_i$, where $T_i = \prod_{j < i} (1 - \alpha_j)$ [S1].

**Parameters:**
*   Grid Resolution $D$: 400 [S1].
*   Samples per ray $N$: 600 or 800 [S1].
*   Batch size: 1024 rays [S1].
*   Memory: $\sim 70$ MB for grid vs $\sim 7$ MB for MLP [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_visual_hull(views, masks, cameras, D=400):
    G1 = zeros((D, D, D))
    G2 = zeros((D, D, D))
    for view, mask, camera in zip(views, masks, cameras):
        # Cast rays from foreground pixels
        rays_fg = cast_rays_from_pixels(view, mask, camera)
        for ray in rays_fg:
            voxels = discretize_ray_to_voxels(ray, D)
            G1[voxels] += 1
        
        # Cast rays from all pixels
        rays_all = cast_rays_from_pixels(view, full_mask(view), camera)
        for ray in rays_all:
            voxels = discretize_ray_to_voxels(ray, D)
            G2[voxels] += 1
            
    # Visual hull V: voxels where G1 >= G2 and G1 > 0
    V = (G1 >= G2) & (G1 > 0)
    return V

def render_vaxnerf(ray_origin, ray_dir, visual_hull, mlp, N=600):
    # Uniform sampling
    t = linspace(z_near, z_far, N)
    points = ray_origin + t[:, None] * ray_dir
    
    # Filter by visual hull
    # Note: In practice, check voxel occupancy for speed
    valid_indices = [i for i, p in enumerate(points) if is_inside_voxel(p, visual_hull)]
    
    if not valid_indices:
        return background_color
        
    sampled_points = points[valid_indices]
    sigmas, colors = mlp(sampled_points, ray_dir)
    
    # Volume rendering
    alphas = 1 - exp(-sigmas * (z_far - z_near) / N)
    Ts = cumprod(1 - alphas[:-1])
    Ts = insert(Ts, 0, 1)
    weights = Ts * alphas[:-1]
    
    color = sum(weights * colors[:-1])
    return color
```

### SimpleNeRF (Regularized Sparse Input NeRF)
SimpleNeRF mitigates overfitting and floaters in sparse-view settings by training two augmented NeRF models alongside the main model to provide depth supervision [S4].

**Inputs:** Sparse set of RGB images.
**Outputs:** Main NeRF model (inference only); Augmented models are discarded after training.

**Data Structures:**
*   **Main Model:** Standard NeRF with Coarse MLP ($F_1$) and Fine MLP ($F_2$).
*   **Points Augmentation Model:** Reduces positional encoding frequency ($l_p$) and concatenates positional encoding to the input of $F_2$ (view-dependent).
*   **Views Augmentation Model:** Computes density and color based on position alone (view-independent).

**Steps:**
1.  **Training Augmented Models:**
    *   **Points Augmentation:** Compute density $\tilde{\sigma}_i$ and color $\tilde{\mathbf{c}}_i$ using reduced $l_p$. Loss $L^{ap}$ supervises depth $z_c$ (main coarse) against $z^{ap}$ (augmented coarse).
    *   **Views Augmentation:** Compute $\tilde{\sigma}_i, \tilde{\mathbf{c}}_i = F_1(\mathbf{p}_i, 0, l_p)$. Loss $L^{av}$ supervises $z_c$ against $z^{av}$.
2.  **Reliability Masking:**
    *   For pixel $q$, reproject a $k \times k$ patch using depths $z_c$ (main) and $z_a$ (augmented) to the nearest training view.
    *   Compute MSE intensities $e_c$ and $e_a$.
    *   Define threshold $e_\tau$.
    *   Compute mask $m_a$:
        *   $1$ if $e_a < e_c$ and $e_a < e_\tau$ (Augmented is better).
        *   $-1$ if $e_c < e_a$ and $e_c < e_\tau$ (Main is better).
        *   $0$ otherwise (Unreliable/Specular).
3.  **Loss Calculation:**
    *   $L^{ap} = \mathbb{1}_{m^{ap}=1} \|z_c - \tau(z^{ap})\|^2 + \mathbb{1}_{m^{ap}=-1} \|z^{ap} - \tau(z_c)\|^2$.
    *   $L^{av} = \mathbb{1}_{m^{av}=1} \|z_c - \tau(z^{av})\|^2 + \mathbb{1}_{m^{av}=-1} \|z^{av} - \tau(z_c)\|^2$.
    *   Add consistency loss $L_{cfc}$ between coarse and fine depths of the main model.
4.  **Inference:** Use only the Main Model.

**Parameters:**
*   Baseline: DS-NeRF [S4].
*   Threshold $e_\tau$: Not explicitly given [S4].

### Neural Confidence Field (ICON)
Adds a confidence prediction head to NeRF to calibrate loss weights during optimization [S3].

**Data Structures:**
*   **Confidence Head:** One fully-connected layer + sigmoid on top of NeRF features.

**Steps:**
1.  **Confidence Prediction:** Predict $\zeta(\mathbf{x}, \mathbf{d})$ for each sample.
2.  **Ray Confidence Aggregation:**
    $$ \zeta_r = \frac{\int_{z_{near}}^{z_{far}} P(z) \zeta(\mathbf{r}(z), \mathbf{d}) dz}{\int_{z_{near}}^{z_{far}} P(z) dz} + \left(1 - \int_{z_{near}}^{z_{far}} P(z) dz\right) \frac{\int_{z_{near}}^{z_{far}} \zeta(\mathbf{r}(z), \mathbf{d}) dz}{\int_{z_{near}}^{z_{far}} P(z) dz} $$
    where $P(z) = T(z)\sigma(\mathbf{r}(z))$. The first term dominates opaque pixels; the second transparent pixels [S3].
3.  **Pose Confidence:** Update pose confidence $\zeta_{P_i}$ using momentum $\beta=0.9$ based on ray confidence from pose $P_i$.
4.  **Loss Calibration:** Minimize $L_{conf} = \| e^{-E/\tau} - \zeta_r \|^2$, where $E$ is photometric error and $\tau$ is temperature. Gradients stopped before NeRF parameters [S3].

### LiDAR-NeRF
Uses a hybrid 3D grid with multi-resolution hash encoding (torch-ngp) for LiDAR data [S2].

**Parameters:**
*   Optimizer: Adam, lr $1e-2$.
*   Coarse samples: 768; Fine samples: 64.
*   Hash finest resolution: 32768.
*   Patch size: $2 \times 8$.
*   Steps: 30k.

### MERF (Memory-Efficient Radiance Fields)
Combines sparse feature grids and 2D feature planes for real-time browser rendering [S10].

**Steps:**
1.  **Optimization:** Use NGP hash grid for differentiable sparsification.
2.  **Baking:** After optimization, query NGP to bake out MERF representation (binary occupancy grid + triplanes) for fast rendering.
3.  **Rendering:** Remove matrix multiplication overhead compared to TensoRF.

## Subtopics

### AdaNeRF
AdaNeRF employs fixed sample positions and incrementally introduces sparsity during training to achieve real-time rendering [S1]. It uses a lightweight Sampling Network to predict importance weights for all points on a line, allowing a heavier Shading Network to process only significant samples [S2][S1]. It differs from ENeRF (multi-view geometry) and NSVF (sparse voxels) [S2].

### SinNeRF
SinNeRF addresses single-image NeRF training by incorporating a depth map prior and adversarial/perceptual losses to constrain unobserved views, preventing degenerate solutions [S1]. It distinguishes itself from AdaNeRF (adaptive sampling) and ReconFusion (diffusion priors) [S1].

### [NeRF-RPN](8663-nerf-rpn.md)
NeRF-RPN performs 3D object detection and instance segmentation directly on the NeRF representation without rendered images. It uses a multi-scale Object-NeRF architecture separating global scene encoding from object encoding, regressing 3D AABBs and class labels from the volumetric feature space, unlike methods relying on 2D features or mesh extraction.

### AligNeRF
AligNeRF addresses rendering artifacts from misalignment (moving objects/calibration errors) and improves high-frequency detail recovery [S1, S2]. It combines a convolution-augmented architecture with a staged, alignment-aware fine-tuning procedure using patch-based sampling and optical flow estimation [S1, S2].

### MultiPlaneNeRF
MultiPlaneNeRF replaces trainable object-specific weights with a non-trainable 2D image-based representation [S1]. It projects 3D points onto fixed 2D training images to create aggregate feature vectors processed by a small shared decoder, allowing generalization across objects by swapping input images, unlike vanilla NeRF which requires per-object training [S1].

### [Pool proposer architecture](8953-pool-proposer-architecture.md)
The Pool proposer architecture generates fine-grained sampling positions by processing coarse features independently before aggregation. It is simpler and more stable than the Transformer proposer, though it has higher variance than the MLPMix variant.

### [Transformer-based Architectures](8998-transformer-based-architectures.md)
These models replace or augment MLPs with Transformers to model spatial dependencies and aggregate multi-view information via pairwise attention. They are distinguished by their ability to generalize across scenes or model large-scale structures in super-resolution, unlike local convolution or hash grid methods.

### [NeRDi](9112-nerdi.md)
NeRDi enables single-view synthesis using a pre-trained 2D diffusion model for general image priors. It minimizes diffusion loss of rendered views under input view constraints, supplemented by semantic/geometric guidance and a visual language model, distinguishing it from standard multi-view NeRF.

### [Neural Light Field Representations](9273-neural-light-field-representations.md)
These methods map camera rays directly to pixel colors, bypassing volumetric ray-marching. They use distillation from a pre-trained NeRF teacher to a deep student network, enabling inference speeds orders of magnitude faster than iterative sampling.

### [RING-NeRF](9323-ring-nerf.md)
RING-NeRF introduces a continuous multi-scale representation and decoder invariance over spatial/scale domains. It provides intrinsic continuous LOD without explicit supervision, encoding LOD in the grid hierarchy and using distance-aware forward mapping to avoid aliasing, unlike AdaNeRF or Zip-NeRF.

### [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md)
This family addresses computational bottlenecks by decomposing the scene into localized, sparse, or projected components. It focuses on structural sparsity and spatial partitioning, differing from AdaNeRF (adaptive capacity) and SinNeRF (sinusoidal activations).

### [Geometric Priors and Depth Integration](9395-geometric-priors-and-depth-integration.md)
These methods integrate explicit geometric constraints (SfM, epipolar geometry, monocular depth) into the optimization loop. They address shape-radiance ambiguity under sparse views by enforcing 3D geometric consistency, unlike AdaNeRF or Latent-NeRF which rely on appearance regularization or latent compression.

### [Generative and Style-based Models](9572-generative-and-style-based-models.md)
These integrate GANs or diffusion models with NeRF to synthesize novel 3D-consistent images from noise or text. They learn a distribution of 3D shapes rather than fitting a static scene, generating radiance field parameters directly from a generative prior, unlike AdaNeRF which adapts a pre-trained NeRF.

### IRON
IRON reconstructs high-quality 3D meshes and material textures from photometric images using a collocated flashlight [S1]. It uses a hybrid optimization for neural SDFs: volumetric rendering for topology, then edge-aware physics-based surface rendering for geometry and material disentanglement, avoiding local minima of direct surface optimization [S1].

### [pi-GAN](10298-pi-gan.md)
pi-GAN is a generative adversarial approach for unsupervised 3D representation learning from 2D images. It outputs a 5D spatio-angular radiance field rendered via volume rendering, conditioned on noise via a StyleGAN-inspired mapping network with FiLM and progressive growing, differing from 2D GANs and other implicit GANs like GRAF.

### [Instance-NeRF](10611-instance-nerf.md)
Instance-NeRF extends pre-trained NeRF with an instance field for 3D instance segmentation without 3D ground truth. It projects coarse 3D masks back to 2D to match multi-view consistent segmentation maps, addressing multi-view inconsistent instance IDs, unlike Semantic-NeRF which uses 2D semantic priors.

### [MIMO-NeRF](10928-mimo-nerf.md)
MIMO-NeRF accelerates inference by replacing the SISO MLP with a Multi-Input Multi-Output (MIMO) MLP. It groups neighboring samples on a ray to process them in parallel, reducing the number of MLP forward passes per pixel, unlike AdaNeRF (view-dependent features) or Sparse Representations (data structure changes).

### [Appearance and Lighting Decomposition](11061-appearance-and-lighting-decomposition.md)
These methods extend $f_\theta(\mathbf{x}, \mathbf{d})$ to explicitly separate geometry, material, and lighting. They predict physical parameters (normals, roughness, diffuse/specular) rather than raw color, enabling relighting and glossy reflections, unlike sibling methods that modify MLP structure for efficiency without physical decomposition.

### [Quadri-Plane Radiance Fields](11128-quadri-plane-radiance-fields.md)
Quadri-Plane Radiance Fields represent a scene using four multi-channel 2D feature planes and a lightweight decoder MLP. This reduces memory usage compared to 3D grids or coordinate-based MLPs, enabling efficient volumetric super-resolution.

### [Neural RGB-D Surface Reconstruction](11352-neural-rgb-d-surface-reconstruction.md)
Neural RGB-D reconstructs room-scale scenes from RGB-D sequences using a hybrid representation: TSDF for geometry and volumetric radiance field for appearance. It uses an implicit surface representation (TSDF) for hard boundaries, unlike standard NeRF's density-based volume which causes semitransparent artifacts.

### DINER
DINER synthesizes novel views from sparse RGB inputs by leveraging predicted dense depth maps to guide geometry and sampling [S1]. It uses off-the-shelf depth estimators for dense geometric priors, enabling high-quality reconstruction under large camera disparities, unlike methods relying on sparse landmarks [S1].

### [GNeuVox](11357-gneuvox.md)
GNeuVox is a framework for fast, differentiable human body synthesis from monocular video. It replaces the implicit MLP with explicit generalizable neural voxels for geometry and individual neural voxels for texture, allowing training in ~5 minutes with quality comparable to previous methods.

### [HexPlane](11358-hexplane.md)
HexPlane represents dynamic 3D scenes by decomposing a 4D spacetime grid into six feature planes spanning coordinate axis pairs (e.g., XY, ZT). It projects points to planes, aggregates features, and passes them to a tiny MLP, avoiding large MLPs and deformation fields, achieving >100x acceleration for dynamic novel view synthesis.

### STensoRRF
STensoRRF represents a 3D scene using a single 4D tensor decomposed into low-rank components for high-quality novel view synthesis and relighting [S1]. It explicitly decodes BRDF values via a dedicated decoder, differing from AdaNeRF or SinNeRF by using a unified tensor rather than multiple grids/positional embeddings [S1].

### GIRAFFE
GIRAFFE represents scenes as a composition of $N$ distinct entities (objects/background), each with its own neural feature field and learnable affine transformation [S1]. This enables explicit control over pose, shape, and appearance of individual entities, decoupling scene composition from a monolithic field [S1].

### [Ced-NeRF](11364-ced-nerf.md)
Ced-NeRF reconstructs dynamic scenes using a hybrid representation of a high-resolution hash grid and a canonical space deformation network. It prioritizes a single high-resolution grid over sparse voxels or separated models to maximize speed and efficiency, avoiding overfitting artifacts common in hybrid dynamic representations.

### [Dynamic and Deformable Fields](11366-dynamic-and-deformable-fields.md)
This family of methods handles time-varying scenes by introducing deformation fields or 4D representations, extending static NeRF to dynamic environments.

### NeRF in the Wild
Adds an Appearance embedding to express different lighting conditions and a Transient embedding to model changing geometry [S5, unofficial].

### CLIP-NeRF
Adds CLIP (text-image pre-trained multimodal model) to provide greater freedom in shape operations and global deformation, using a disentangled conditional NeRF architecture [S6]. It applies a learned deformation network for shape modulation and embeds appearance codes into volume rendering [S6].

### JacobiNeRF
Synthesizes a NeRF model based on second-order information (Jacobian matrix) to maximize mutual information between highly correlated entities under random scene perturbations [S6]. It uses the standard rendering equation $\hat{C}(r, z_s, z_c) = \sum_{i=1}^{N_c-1} c_i \alpha_i \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ [S6].

### RefSR-NeRF
Reference-guided super-resolution neural radiation field framework combining NeRF and RefSR-CNN [S7]. Stage 1 modifies NeRF sampling to patch ray sampling on high-resolution input for low-resolution image. RefSR-CNN yields feature maps from detail-dominant and degradation-dominant modules, which are fused to reconstruct high-resolution images [S7]. On NeRF-LLFF, PSNR is nearly one point higher than the best paper [S7].

### RelightingNeRF
Proposed for unstructured photographs and free-viewpoint reconstruction, changing architecture to output SDF values and latent features [S7]. Input position into first MLP outputs SDF and latent features. Normals, shadow, and specular cues are obtained from SDF. Second MLP takes position, lighting, viewing direction, and latent features to produce final color [S7].

### DiffusioNeRF
Utilizes a denoising diffusion model to learn priors for scene geometry and color, normalizing geometry and color for better quality under sparse views [S7]. During NeRF training, it renders random RGBD patches and backpropagates estimated gradients of the log-likelihood to the color and density fields [S7]. On LLFF, PSNR is 1-2 points higher for 3, 6, and 9 views [S7].

### MS-NeRF
Addresses rendering distortion from reflection/refraction (e.g., mirrors) where standard NeRF fails multi-view consistency [S7]. It represents the scene by a set of characteristic fields in parallel subspaces. NeRF output is modified to output density and features, generating multiple sets. Volumetric rendering generates multiple feature maps. A Decoder MLP in each subspace decodes RGB. A Gate MLP obtains pixel-level weights. Features are integrated by summation [S7]. On synthetic, real, and RFFR datasets, PSNR is 2-4 points higher [S7].

### Edit-NeRF
The first method to execute user-controlled deformation models on NeRF, proposing a modular network's conditional radiance field [S6]. It trains on multiple instances of the same category, learns underlying part features unsupervised, and allows user guidance in 2D shape/texture to propagate to 3D space [S6]. It has limited capability (adding/deleting local parts) and slow iterative optimization [S6].

### DT-NeRF
Both Diffusion and Transformer models are trained jointly within a unified optimization framework, sharing the same loss function, and are not pretrained but optimized simultaneously during training through backpropagation [S8].

### NeRF-A
A NeRF variant using appearance embeddings (NeRF-A) without transient embeddings for handling varying camera conditions [S9]. It trains on the left half of images and tests on the right half, interpolating between appearance embeddings for controllable rendering [S9]. On classroom image: PSNR 23.52, SSIM 0.907, LPIPS 0.247 [S9].

### MERF
A volumetric scene representation achieving real-time rendering of large-scale scenes in a browser by combining a sparse feature grid and high-resolution 2D feature planes, with a lossless baking procedure [S10]. It uses a unified radiance field representation with two parameterizations: NGP hash grid for optimization and baked binary occupancy grid/triplanes for rendering. During optimization, NGP hash grid compresses parameterization. After optimization, NGP is queried to bake out MERF. Compared to TensoRF, MERF speeds up query time by removing matrix multiplication and halves memory bandwidth consumption [S10]. MERF achieves better image quality for the same compression as SNeRG++, matching a SNeRG++ model 17x larger [S10].

## Operating characteristics
*   **VaxNeRF:** Trains 2-8x faster than original NeRF with no quality degradation [S1]. Requires $\sim 70$ MB memory for the $400^3$ voxel grid, compared to $\sim 7$ MB for the NeRF MLP [S1]. Speed gain depends on visual hull size relative to scene volume (e.g., factor of 4 for large objects, almost 8 for fine objects) [S1].
*   **SimpleNeRF:** Outperforms RegNeRF and FreeNeRF on LLFF dataset with 2 and 4 views (e.g., SSIM 0.6501, PSNR 19.57 with 2 views) [S4]. Designed for few-shot/sparse input NeRF training [S4].
*   **MERF:** Achieves real-time rendering in a browser [S10]. Speeds up query time compared to TensoRF by removing matrix multiplication [S10].
*   **HexPlane:** Over 100x acceleration for dynamic novel view synthesis [S1].
*   **GNeuVox:** Trains in approximately 5 minutes for a novel human body [S1].

## Background
NeRF (Neural Radiance Fields) was introduced by Mildenhall et al. in 2020 [S1]. VaxNeRF was proposed by Kondo et al. in 2021 to address the time-consuming optimization process by restricting sampling to the visual hull [S1]. SimpleNeRF was developed to regularize sparse input NeRFs using augmented models [S4]. LiDAR-NeRF was implemented based on torch-ngp for LiDAR view synthesis [S2]. ICON was developed for incremental confidence in joint pose and radiance field optimization [S3]. MERF was developed for memory-efficient radiance fields for real-time view synthesis [S10]. Other variants like SinNeRF, NeRDi, and AdaNeRF emerged to address single-view synthesis, diffusion priors, and adaptive sampling respectively [S1, S2][S5, unofficial][S6, S7, S8, S9].

## Sources
[S1] Accelerating NeRF with the Visual Hull, https://doi.org/10.5201/ipol.2024.553, fetched 2026-10-26
[S2] LiDAR-NeRF: Novel LiDAR View Synthesis via Neural Radiance Fields, https://arxiv.org/abs/2304.10406v2, fetched 2026-10-26
[S3] ICON: Incremental CONfidence for Joint Pose and Radiance Field Optimization, https://doi.org/10.1109/cvpr52733.2024.00517, fetched 2026-10-26
[S4] SimpleNeRF: Regularizing Sparse Input Neural Radiance Fields with Simpler Solutions, https://doi.org/10.1145/3610548.3618188, fetched 2026-10-26
[S5] NeRF的基础及后续扩展, https://pengsida.net/files/NeRF_Tutorial.pdf, fetched 2026-10-30 (unofficial)
[S6] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
[S7] Methods and Strategies for Improving the Novel View Synthesis Quality of Neural Radiation Field, https://doi.org/10.1109/access.2024.3382997, fetched 2026-10-30
[S8] DT-NeRF: A Diffusion and Transformer-Based Optimization Approach for Neural Radiance Fields in 3D Reconstruction, https://doi.org/10.62762/tis.2025.874668, fetched 2026-10-26
[S9] HDR-Plenoxels: Self-Calibrating High Dynamic Range Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_23, fetched 2026-10-26
[S10] MERF: Memory-Efficient Radiance Fields for Real-time View Synthesis in Unbounded Scenes, https://doi.org/10.1145/3592426, fetched 2026-10-26
