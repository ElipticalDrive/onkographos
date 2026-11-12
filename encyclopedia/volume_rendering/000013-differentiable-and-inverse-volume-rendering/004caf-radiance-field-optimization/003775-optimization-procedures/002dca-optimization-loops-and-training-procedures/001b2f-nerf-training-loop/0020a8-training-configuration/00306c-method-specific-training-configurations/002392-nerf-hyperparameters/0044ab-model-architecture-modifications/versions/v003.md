# Model Architecture Modifications

Also called: NeRF Architecture Variants, Radiance Field Network Modifications

## Overview
This family encompasses modifications to the core neural network architecture of Neural Radiance Fields (NeRF) to address specific limitations such as overfitting in sparse views, computational bottlenecks, lack of semantic understanding, or physical inaccuracies like depth-of-field blur and reflections. Unlike standard NeRF, which uses a single MLP to map $(\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$, these methods alter the input space (e.g., adding fluid volume or appearance embeddings), the output space (e.g., adding semantic branches or SDFs), or the internal representation (e.g., replacing MLPs with voxel grids or hash tables).

## Shared principles
The standard NeRF architecture [S6, unofficial] consists of two Multi-Layer Perceptrons (MLPs): a coarse network and a fine network. The coarse network takes positional encoded position $\gamma(\mathbf{x})$ and outputs a latent feature vector $\mathbf{h}$ and volume density $\sigma$. The fine network takes $\mathbf{h}$, positional encoded position $\gamma(\mathbf{x})$, and positional encoded view direction $\gamma(\mathbf{d})$ to output color $\mathbf{c}$.

$$
\gamma(\mathbf{x}, d_1, d_2) = [\mathbf{x}, \sin(2^{d_1}\pi\mathbf{x}), \cos(2^{d_1}\pi\mathbf{x}), \dots, \sin(2^{d_2-1}\pi\mathbf{x}), \cos(2^{d_2-1}\pi\mathbf{x})]
$$

Modifications generally fall into three categories:
1.  **Input Expansion:** Adding new conditioning vectors (e.g., appearance embeddings $\mathbf{z}_i$, fluid volume $v$, or previous stage RGB) to the input concatenation.
2.  **Output Expansion:** Adding auxiliary heads to predict values other than $\sigma$ and $\mathbf{c}$, such as semantic labels $y_{aux}$, object codes $o$, or area encoding probabilities $p$. These are rendered using the same transmittance weights $T_i$ and opacity $\alpha_i$ as color:
    $$
    \hat{Y}(r) = \sum_{i=1}^{N} T_i \alpha_i y_{aux}(x_i)
    $$
3.  **Representation Substitution:** Replacing the continuous MLP function with discrete structures (voxel grids, hash tables) or different implicit surfaces (SDFs).

## Subtopics

### [Simple-NeRF](7371-simple-nerf.md)
A regularization method for sparse-input NeRF that trains two "augmented" models (Smoothing and Lambertian) in tandem with the main model to mitigate overfitting and floaters. It differs from siblings like [DoF-NeRF](9432-dof-nerf.md) by using simplified architectures (reduced PE frequencies, no view dependence) rather than identical twins.

### [DoF-NeRF](9432-dof-nerf.md)
Extends NeRF to simulate depth-of-field (DoF) by learning aperture $K$ and focus distance $F$ parameters per image. It models the Circle of Confusion (CoC) diameter $\delta(h_t) = K \times |1/F - 1/h_t|$ to scatter radiance, unlike standard NeRF which assumes a pinhole camera.

### [MIMO-NeRF-self-R2](11400-mimo-nerf-self-r2.md)
Optimizes training speed by grouping rays and sharing MLP instances. With reduction factors $R^1=1, R^2=1$, it reduces the number of MLP forward passes per step compared to standard NeRF, balancing speed and quality better than variants like [MIMO-NeRF-distill].

### [Voxel-based Radiance Fields](11470-voxel-based-radiance-fields.md)
Replaces the MLP with structured 3D voxel grids for density ($G_\sigma$) and spherical harmonic coefficients for color ($G_{sh}$). This enables real-time rendering via trilinear interpolation, trading the high-capacity continuous approximation of MLPs for memory efficiency.

### [WaH-NeRF](11548-wah-nerf.md)
Addresses sample-position and prediction confusion in sparse views by predicting deformable sampling offsets $O_{offset}$ to adaptively shift conical frustums. It differs from [Simple-NeRF](7371-simple-nerf.md) by modifying the sampling geometry before rendering rather than regularizing the output.

### [Multi-Space Neural Radiance Field (MS-NeRF)](11707-multi-space-neural-radiance-field-ms-nerf.md)
Handles reflective/refractive objects by outputting $K$ parallel feature fields (sub-spaces) with gating weights $w_k$. Virtual images are placed in specific sub-spaces, unlike standard NeRF which blurs them as surface textures.

### [NeRF-VPT](11770-nerf-vpt.md)
Uses a cascading view prompt tuning paradigm where the RGB output of stage $i-1$ is concatenated with the direction encoding as input to stage $i$. This progressively enhances quality without external priors, unlike methods requiring pre-trained networks.

### [Deceptive-NeRF/3DGS](11859-deceptive-nerf-3dgs.md)
Densifies training data by generating pseudo-observations (RGB-D) from coarse renders using a diffusion model, incorporating them directly into the training set. This avoids the per-step cost of Score Distillation Sampling used in other regularizers.

### [DeblurNeRF](11964-deblurnerf.md)
Models blur as pixel-wise independent ray transformations using a Deformable Sparse Kernel (DSK). It predicts offsets to ray origins and directions, approximating spatially varying blur more efficiently than rigid camera motion models.

### [NeRF in the Wild (NeRF-W)](11995-nerf-in-the-wild-nerf-w.md)
Decouples static geometry from dynamic appearance using per-image appearance embeddings $\mathbf{z}_i$ and transient MLPs $M_\tau$ for transient objects. This handles lighting changes and moving objects that violate multi-view consistency in standard NeRF.

### [NRFF](12072-nrff.md)
Replaces positional encoding with Multiscale Tensor Decomposition (MTD) and predicts Anisotropic Spherical Gaussians (ASG) features. It integrates specular contributions via Rendering Equation Encoding (REE) in the latent space, avoiding explicit environment maps.

### [Semantic and Feature Branches](12095-semantic-and-feature-branches.md)
Adds an auxiliary output branch predicting a high-dimensional vector $y_{aux}$ (semantic/feature) alongside $\sigma$ and $\mathbf{c}$. This enables 3D-aware tasks like segmentation by rendering features using standard volume integration.

### [Activation-Informed Coarse-to-Fine Rendering](12277-activation-informed-coarse-to-fine-rendering.md)
Accelerates inference by skipping the fine-pass MLP evaluation for rays with low intermediate activation values $v_\ell$ from the coarse pass. It uses a threshold $\tau = \mu(V_\ell)$ to identify empty/background rays.

### [MixRT](12401-mixrt.md)
A hybrid representation for edge devices combining a triangle mesh (geometry), a displacement map (detail), and an Instant-NGP hash table (density/color). It distributes workload across rasterizers and SIMD units for real-time performance.

### [NeRF for Object Segmentation](12404-nerf-for-object-segmentation.md)
Introduces an "area encoding" auxiliary branch predicting occlusion probability $p$. This allows the model to distinguish visible from occluded regions, enabling segmentation alongside rendering.

### [MINE (Continuous Depth MPI)](12459-mine-continuous-depth-mpi.md)
Reconstructs scenes using a continuous depth Multi-Plane Image (MPI). Complexity is constant with respect to the number of target views, unlike pixelNeRF, enabling dense supervision during training.

### [Activation Feature Density Approximation](12492-activation-feature-density-approximation.md)
Estimates density $\hat{\vec{d}}$ directly from intermediate MLP activations $f_\ell$ using handcrafted functions (e.g., $f_1(f_\ell) = \text{ReLU}((\mu(f_\ell) - \sigma(f_\ell)) - f_\ell)$), replacing expensive coarse network evaluations.

### [Signed Distance Function (SDF) Representations](12495-signed-distance-function-sdf-representations.md)
Empty section.

### [SDF-based Neural Fields](12947-sdf-based-neural-fields.md)
Replaces density $\sigma$ with a Signed Distance Function $s(\mathbf{x})$, defining geometry via the zero-level set. Density is derived via softplus/sigmoid: $\sigma(\mathbf{x}) = \text{softplus}(s(\mathbf{x}))$. This provides sharper boundaries than density-based NeRFs.

### [FlipNeRF](12967-flipnerf.md)
Generates auxiliary "flipped reflection rays" based on estimated surface normals to improve rendering quality and normal accuracy in sparse views. It targets floating artifacts through uncertainty-aware emptiness regularization.

### [Uncertainty-aware NeRF](12968-uncertainty-aware-nerf.md)
Predicts uncertainties for color and depth as Gaussian distributions (mean and variance). This reveals data reliability at the pixel level, allowing filtering of poorly rendered views during training.

### [Volume-Augmented NeRF](12973-volume-augmented-nerf.md)
Binds explicit fluid volume values $v$ to sample points as a third input channel alongside position and view direction. This enables modeling of dynamic fluid density and color changes.

### [Neural-PIL Decomposition Architecture](12975-neural-pil-decomposition-architecture.md)
Decouples scene representation into a Coarse Network (appearance) and a Decomposition Network (geometry/BRDF). It uses pre-trained auto-encoders for illumination and BRDF manifolds to enable physically plausible rendering under varying lighting.

### [Feature Fields](12995-feature-fields.md)
Generalizes NeRF to predict arbitrary per-point feature signals $\mathbf{f}$ (semantic, physical, etc.) in addition to color and density. Supervision is achieved by reprojecting 3D features to 2D image planes.

### [Neural Multi-View Image Reconstruction with Hash Encoding](13013-neural-multi-view-image-reconstruction-with-hash-encoding.md)
Combines hash-encoded implicit SDF representation with a neural radiance field. It uses a hierarchical hash grid for geometry, offering faster training and lower memory than dense voxel grids while preserving fine details.

### [G-NeLF](13480-g-nelf.md)
A grid-based Neural Light Field method that predicts ray color directly without point-wise density modeling. It uses a hash-based multi-resolution tri-plane and an LSTM decoder to process ray features sequentially.

### [DM-NeRF](13525-dm-nerf.md)
Introduces an "object field" predicting a one-hot object code $o$ for every 3D point. This enables instance-level decomposition and manipulation (translation, rotation) without explicit mesh extraction.

### [Last Layer Re-training](13532-last-layer-re-training.md)
Propagates color edits from a single view to the entire NeRF by retraining only the last layer of the color MLP. This preserves view-dependent information in deeper layers while enabling near real-time editing.

### Rip-NeRF Optimization
- [S1] The optimization process for Rip-NeRF, involving the tiny MLP weights and the ripmaps R.
- [S1] Learning rate schedule: Reduced by 0.6x at steps 60K, 90K, 100K, and 108K, over a total of 120K iterations.
- [S1] Batch size adjustment: Dynamically adjusted to maintain approximately 256K spheres per batch.
- [S1] MLP weights initialization: Glorot and Bengio (2010).
- [S1] Ripmap R initialization: Uniform distribution over [-0.01, 0.01].
- [S1] Optimizer: AdamW.
- [S1] Learning rate scaling: Base learning rate for R is scaled by 10x due to its direct scene representation role.
- [S1] Base learning rate: 2 × 10^-3.

### MIMO-NeRF-self-R3
- [S2] A variant employing Np-1 reformulated MIMO MLPs with reduction factors R1=1, ..., R^{Np-1}=1.
- [S2] The total number of MLPs running is calculated as $\sum_{m=1}^{N_p-1} \frac{N}{N_p} R^m = N \frac{N_p-1}{N_p} < N$.
- [S2] On the Blender dataset with Np=4, MIMO-NeRF-self-R3 achieves PSNR 30.89, SSIM 0.949, LPIPS 0.060, and T-time 5.05h.
- [S2] On the LLFF dataset with Np=4, MIMO-NeRF-self-R3 achieves PSNR 27.27, SSIM 0.860, LPIPS 0.171, and T-time 3.78h.
- [S2] On the Blender dataset with Np=8, MIMO-NeRF-self-R3 achieves PSNR 29.88, SSIM 0.937, LPIPS 0.080, and T-time 7.75h.
- [S2] On the LLFF dataset with Np=8, MIMO-NeRF-self-R3 achieves PSNR 25.66, SSIM 0.797, LPIPS 0.243, and T-time 5.97h.

### GARF
- [S3] A method using Gaussian activations as an alternative to positional encoding in NeRF, combined with bundle adjustment for pose estimation from unknown camera poses.
- [S3] Uses Gaussian activations instead of positional encoding.
- [S3] Performs bundle adjustment for pose estimation.
- [S3] Use Gaussian activations as an effective alternative to positional encoding.
- [S3] Use bundle adjustment for pose estimation.
- [S3] Successfully recovers scene representations from unknown camera poses in challenging scenes with low-textured regions.

### NeRF-VAE
- [S3] A generative NeRF model that generalizes to out-of-distribution scenes and removes the need to train on each scene from scratch.
- [S3] The NeRF renderer is conditioned on a latent code. This latent code is trained using Iterative Amortized Inference and a ResNet encoder.
- [S3] An attention-based scene function is introduced instead of the typical MLP.
- [S3] Outperforms baseline NeRF with a low number of scene views (5-20), but is outperformed by baseline NeRF when a large number of views are available (100+) due to lower scene expressivity.

### Fig-NeRF
- [S3] Scene composition model for object interpolation and amodal segmentation using two separate NeRF models for foreground and background.
- [S3] Use an appearance NeRF conditioned on latent codes for the background model.
- [S3] Use two photometric losses, one for the foreground and one for the background.

### HyperNeRF
- [S3] A method extending the canonical space to a higher dimension and using a slicing MLP to describe how to return to the 3D representation using ambient space coordinates.
- [S3] Use the canonical coordinate and ambient space coordinate to condition the usual density and color MLPs of baseline NeRF models.
- [S3] Achieved great results in synthesizing views in scenes with topological changes, such as a human opening their mouth or a banana being peeled.

### FastSR-NeRF
- [S3] A method introducing a simple super-resolution/upsampling CNN into the NeRF pipeline. It trains a small, fast, and efficient NeRF model to produce low-resolution 3D consistent features and uses a fast SR model to upscale these features.
- [S3] Introduce a novel augmentation technique called random patch sampling, which improves SR performance by increasing patch diversity.
- [S3] Unlike prior NeRF+SR methods that rely on complex training procedures, distillation, or high-resolution reference images, FastSR-NeRF requires no architectural changes or heavy computing.

### LISA
- [S3] A method targeting the modeling of hands by approximating human hands with a collection of rigid parts.
- [S3] Query points are input into MLPs, which are used to predict geometry (via SDFs) and color.

### PIE-NeRF
- [S3] A pipeline integrating physics-based elastodynamic simulation with a neural radiance field representation, using a meshless discretisation based on quadratic generalized moving least squares on top of the density field of a NeRF.
- [S3] Enables large deformations and nonlinear hyperelastic material simulation directly on the implicit representation and supports interactive manipulation of real-world captured scenes with complex geometries.

### DäRF Implementation Details
- [S4, unofficial] NeRF Backbone: K-planes [36].
- [S4, unofficial] MDE Model: DPT-hybrid [38].
- [S4, unofficial] Optimizer: Adam [21].
- [S4, unofficial] NeRF Learning Rate: 1e-2.
- [S4, unofficial] MDE Learning Rate: 1e-5.
- [S4, unofficial] LR Schedule: Cosine warmup.

### Few-shot NeRF
- [S4, unofficial] NeRF represents a scene as a continuous function Fθ(·) implemented by a neural network with parameters θ, taking 3D coordinates x and viewing direction d with positional encoding γ(·) to output color c and density σ.
- [S4, unofficial] Few-shot NeRF setting: number of input viewpoints |S| is drastically reduced to a few viewpoints, specifically |S| < 20.

### Time-dependent NeRF
- [S5] A NeRF variant that represents scene geometry and appearance at each specific time step t, trained using neighboring frames around time t, to capture local scene geometry and motion.
- [S5] Represent scene appearance using mapping function c = ϕc(x, d) taking point x ∈ R3 and viewing direction d ∈ R2 to predict color c ∈ R3.
- [S5] Represent scene geometry via mapping function s = ϕg(x) which maps a 3D point x to a signed distance value s ∈ R.
- [S5] For a camera ray r(h) = o + hd, sample K 3D points {xi(r) = r(hi) | i = 1, ..., K, hi < hi+1}.
- [S5] Use the time-dependent NeRF to compute SDF flows, which are used to constrain the learning of continuous camera motion via the linear relationship between SDF flow and scene flow.
- [S5] Convert signed distance si to density σi using: σi = max((Φ(si) - Φ(si+1))/Φ(si), 0), where Φ(s) = 1/(1+e^(1-γs)), γ is a learnable parameter, and i is the point index.
- [S5] Render pixel color Ĉ(r) via volume rendering: Ĉ(r) = Σ(i=1 to K) Li σi ci, where Li = Π(j=1 to i-1) (1 - σj) is the accumulated transmittance.

### NeRF Model Architecture
- [S6, unofficial] The neural network architecture for NeRF, consisting of dense layers with ReLU activations, potentially using view directions as input.
- [S6, unofficial] `D`: The depth of the network (number of hidden layers). Default is 8.
- [S6, unofficial] `input_ch_views`: The number of input channels for view directions. Default is 3.
- [S6, unofficial] `output_ch`: The number of output channels. Default is 4 (RGB + density).
- [S6, unofficial] `use_viewdirs`: Boolean flag to determine if view directions are used as input to the network.

### NeuMan Network Architecture
- [S7] A NeRF architecture consisting of a coarse sub-model and a fine sub-model for the scene, and a single sub-model for the human NeRF model, leveraging SMPL geometry priors.
- [S7] Architecture visualization (Fig. 10): Blue boxes are raw inputs, green boxes are positional encoded values, black boxes are hidden layers, pink boxes are final outputs. Numbers indicate feature dimensions. '+' indicates concatenation. ReLU is applied after each hidden layer.
- [S7] Unlike NeuralBody which overfits to training observations and produces poor rendering on the back of the subject, NeuMan generalizes better and faithfully renders the back.

### Object Field Representation
- [S8, unofficial] A function f(p) that maps a 3D point p to an object code vector o, representing object ownership independent of viewing angles.
- [S8, unofficial] H: Predefined maximum number of solid objects the network predicts. A relatively large value is chosen in practice.
- [S8, unofficial] f: Parameterized by a series of MLPs.
- [S8, unofficial] Variable number of objects and large non-occupied volumes. Solved by fixing dimension to H+1, reserving the last dimension for empty space to handle occlusion/collision.

## Background
The original NeRF architecture [S6, unofficial] introduced a compact MLP-based representation for novel view synthesis. Subsequent research rapidly expanded the architecture to handle specific challenges: sparse views [S4, unofficial], dynamic scenes [S5], physical effects like blur and DoF [9432-dof-nerf.md, 11964-deblurnerf.md], and semantic understanding [12095-semantic-and-feature-branches.md]. The field has shifted from pure MLPs to hybrid representations (voxels, hash grids) [11470-voxel-based-radiance-fields.md, 13013-neural-multi-view-image-reconstruction-with-hash-encoding.md] and generative models [S3] to improve efficiency and generalization.

## Sources
[S1] Rip-NeRF: Anti-aliasing Radiance Fields with Ripmap-Encoded Platonic Solids, https://arxiv.org/abs/2405.02386v1, fetched 2026-10-26
[S2] MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields, https://arxiv.org/abs/2310.01821v1, fetched 2026-10-26
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S4] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
[S5] Joint Optimization of Neural Radiance Fields and Continuous Camera Motion from a Monocular Video, https://doi.org/10.1109/cvpr52734.2025.01071, fetched 2026-10-26
[S6] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
[S7] NeuMan: Neural Human Radiance Field from a Single Video, https://doi.org/10.1007/978-3-031-19824-3_24, fetched 2026-10-29
[S8] DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images, https://arxiv.org/abs/2208.07227v2, fetched 2026-10-26 (unofficial)
