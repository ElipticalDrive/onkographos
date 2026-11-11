# Model Architecture Modifications

Also called: NeRF Architecture Variants, Auxiliary Output Branches, Conditional NeRFs

## Overview
Model Architecture Modifications refer to structural changes in the neural network components of Neural Radiance Fields (NeRF) to address specific limitations such as overfitting in sparse views, lack of semantic understanding, dynamic scene representation, or computational efficiency. Unlike standard NeRF which maps 3D coordinates and view directions to density and color via a single Multi-Layer Perceptron (MLP), these modifications introduce auxiliary branches, conditional latent codes, or alternative geometric representations (e.g., SDFs, Voxels) to decouple or enhance specific scene properties [S1, S2, S3]. They are chosen when the standard volume rendering pipeline fails to capture complex phenomena like depth-of-field blur, semantic segmentation, or topological changes [S1, S2, S3].

## Shared principles
The core mechanism of model architecture modifications involves altering the input-output mapping of the radiance field function $F_\Theta$. In standard NeRF, the function is:
$$
(\sigma, \mathbf{c}) = F_\Theta(\gamma(\mathbf{x}), \gamma(\mathbf{d}))
$$
where $\mathbf{x}$ is the 3D position, $\mathbf{d}$ is the view direction, $\gamma$ is positional encoding, $\sigma$ is volume density, and $\mathbf{c}$ is color [S4, unofficial].

Modifications generally follow one of these structural patterns:
1.  **Auxiliary Branches:** The network outputs an additional vector $y_{aux}$ alongside $\sigma$ and $\mathbf{c}$ [S1, S2, S3]. This auxiliary output is rendered using the same transmittance weights $\alpha_i$ as the color channel:
    $$
    \hat{Y}(r) = \sum_{i=1}^{N} T_i \alpha_i y_{aux}(x_i)
    $$
    where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the accumulated transmittance [S1, S2, S3].
2.  **Conditional Inputs:** The network takes additional inputs such as per-image latent codes $\mathbf{z}_i$, time $t$, or semantic labels to condition the output, allowing the model to represent variations in appearance or geometry [S3].
3.  **Geometric Representation Swap:** Replacing the density-based representation with Signed Distance Functions (SDFs) or voxel grids to enforce explicit geometric constraints or improve memory efficiency [S3][S4, unofficial].

## Subtopics

### [Simple-NeRF](7371-simple-nerf.md)
Simple-NeRF is a regularization method for sparse-input NeRF that mitigates overfitting and floaters by training two simpler augmented NeRF models in tandem with the main model. It sets itself apart from siblings like DoF-NeRF by focusing on architectural simplification (reduced positional encoding frequencies and disabled view-dependent radiance) to enforce geometric consistency without external supervision.

### [DoF-NeRF](9432-dof-nerf.md)
DoF-NeRF extends NeRF to simulate depth-of-field (DoF) effects by incorporating the Circle of Confusion (CoC) into the rendering equation. It differs from standard variants by explicitly modeling aperture and focus distance, allowing the synthesis of views with arbitrary focus configurations and the recovery of sharp, all-in-focus 3D representations from shallow DoF inputs.

### [MIMO-NeRF-self-R2](11400-mimo-nerf-self-r2.md)
A variant of MIMO-NeRF using two reformulated MIMO MLPs with reduction factors $R_1=1$ and $R_2=1$. It is chosen for its balance between rendering speed and quality in multi-view synthesis tasks.

### [Voxel-based Radiance Fields](11470-voxel-based-radiance-fields.md)
A memory-efficient representation that replaces the MLP with voxel grids for volume density and spherical harmonic coefficients. It enables real-time volume rendering and efficient training, setting it apart from MLP-based methods by trading some view-dependent modeling fidelity for significant computational gains.

### [WaH-NeRF](11548-wah-nerf.md)
A framework for NeRF from Sparse inputs (NeRF-S) that addresses sample-position confusion and prediction confusion. It sets itself apart by explicitly modeling the "WHERE to Sample" and "HOW to Predict" aspects to mitigate rendering confusion in sparse settings.

### [Multi-Space Neural Radiance Field (MS-NeRF)](11707-multi-space-neural-radiance-field-ms-nerf.md)
MS-NeRF represents a scene using a group of feature fields in parallel sub-spaces instead of a single radiance field in Euclidean space. It is chosen for handling reflective and refractive objects where multi-view consistency is violated, setting it apart from single-field NeRFs.

### [NeRF-VPT](11770-nerf-vpt.md)
NeRF-VPT uses a cascading view prompt tuning paradigm, using RGB information from preceding rendering iterations as visual prompts for subsequent stages. It sets itself apart by progressively enhancing image quality through iterative prompting rather than static network weights.

### [Deceptive-NeRF/3DGS](11859-deceptive-nerf-3dgs.md)
A sparse-view reconstruction method that densifies input observations by generating photorealistic pseudo-observations via a diffusion model. It treats these pseudo-observations as real training views to overcome overfitting and artifacts in few-view settings, differing from architectural changes by augmenting the data distribution.

### DeblurNeRF
DeblurNeRF learns a deformable sparse kernel to warp ray origins to approximate spatially varying blur [S3]. It relies on an MLP to model local ray mixing, setting it apart by explicitly handling blur in the input images rather than assuming pinhole camera models [S3].

### [NeRF in the Wild (NeRF-W)](11995-nerf-in-the-wild-nerf-w.md)
NeRF-W addresses per-image appearance variations due to lighting and transient objects by conditioning the color MLP on per-image appearance embeddings and using a separate MLP for transient objects. It sets itself apart from standard NeRF by explicitly modeling appearance and transient artifacts.

### NRFF
NRFF uses multiscale tensor decomposition (MTD) and Anisotropic Spherical Gaussians (ASG) to encode view-dependent effects [S3]. It sets itself apart by improving the efficiency and accuracy of view-dependent modeling compared to standard MLPs [S3].

### [Semantic and Feature Branches](12095-semantic-and-feature-branches.md)
Semantic and Feature Branches extend the standard NeRF MLP to output auxiliary data—such as semantic labels, continuous feature vectors, or occlusion probabilities—alongside density and color. What sets this family apart from sibling architecture modifications (e.g., Simple-NeRF or DoF-NeRF) is the explicit decoupling of geometric/radiometric properties from high-level semantic or perceptual representations via dedicated network branches, allowing for differentiable supervision from 2D priors or sparse annotations.

### [Activation-Informed Coarse-to-Fine Rendering](12277-activation-informed-coarse-to-fine-rendering.md)
A rendering optimization that skips the expensive fine-pass evaluation of the NeRF MLP for rays that are determined to be empty or background, based on intermediate activation values from the coarse pass. It sets itself apart by optimizing the inference pipeline rather than the training architecture.

### MixRT
MixRT consists of a low-quality mesh for coarse geometry, a view-dependent displacement map, and a compressed NeRF model in Instant-NGP format [S3]. It sets itself apart by hybridizing mesh-based and neural rendering techniques [S3].

### [NeRF for Object Segmentation](12404-nerf-for-object-segmentation.md)
A NeRF-based component that estimates scene geometry and predicts semantic labels (occlusion masks) simultaneously using 'area encoding'. It leverages the correlation between geometry estimation and semantic label prediction, setting it apart by integrating segmentation directly into the volume rendering process.

### [MINE (Continuous Depth MPI)](12459-mine-continuous-depth-mpi.md)
MINE reconstructs a scene using a continuous depth Multi-Plane Image (MPI) representation, allowing rendering of any target view via homography warping after a single encoder pass. It sets itself apart from MLP-based NeRFs by using a plane-based representation for efficient novel view synthesis.

### [Activation Feature Density Approximation](12492-activation-feature-density-approximation.md)
A method to estimate volumetric density $\hat{d}$ directly from intermediate activation features $f_\ell$, allowing replacement of the expensive coarse MLP evaluation with a smaller MLP or direct computation. It sets itself apart by optimizing the density estimation step for real-world scenes.

### [Signed Distance Function (SDF) Representations](12495-signed-distance-function-sdf-representations.md)
SDF Representations use Signed Distance Functions to provide a more explicit and smooth formulation of surfaces, addressing geometric inaccuracy and surface holes typical of density-based NeRFs. They set themselves apart by enforcing explicit surface constraints.

### [SDF-based Neural Fields](12947-sdf-based-neural-fields.md)
SDF-based Neural Fields combine NeRF with SDFs to provide a more explicit and smooth formulation of surfaces. They set themselves apart by integrating SDF regularization into the neural field training.

### FlipNeRF
FlipNeRF is a regularization method for few-shot novel view synthesis that uses flipped reflection rays as additional training resources to improve surface normal estimation and reduce floating artifacts [S3]. It sets itself apart by leveraging geometric symmetries for regularization [S3].

### [Uncertainty-aware NeRF](12968-uncertainty-aware-nerf.md)
Uncertainty-aware NeRF separately predicts uncertainties for rendered color and depth images to reveal data reliability at the pixel level. It sets itself apart by providing confidence estimates alongside renderings, addressing artifacts and redundancy in synthetic data.

### [Volume-Augmented NeRF](12973-volume-augmented-nerf.md)
Volume-Augmented NeRF infers fluid dynamics by combining a volume-augmented NeRF renderer with a differentiable fluid simulator. It sets itself apart by integrating physical simulation constraints into the neural rendering architecture.

### [Neural-PIL Decomposition Architecture](12975-neural-pil-decomposition-architecture.md)
Neural-PIL Decomposition Architecture decomposes the neural radiance field into appearance, illumination, and geometry components. It sets itself apart by explicitly modeling the physical factors of image formation.

#### Rip-NeRF Optimization
- **Definition:** The optimization process for Rip-NeRF, involving the tiny MLP weights and the ripmaps R [S1].
- **Step:** Learning rate schedule: Reduced by 0.6x at steps 60K, 90K, 100K, and 108K, over a total of 120K iterations [S1]. Batch size adjustment: Dynamically adjusted to maintain approximately 256K spheres per batch [S1].
- **Parameter:** MLP weights initialization: Glorot and Bengio (2010) [S1]. Ripmap R initialization: Uniform distribution over [-0.01, 0.01] [S1]. Optimizer: AdamW [S1]. Learning rate scaling: Base learning rate for R is scaled by 10x due to its direct scene representation role [S1]. Base learning rate: $2 \times 10^{-3}$ [S1].

#### MIMO-NeRF-self-R3
- **Definition:** A variant employing $N_p-1$ reformulated MIMO MLPs with reduction factors $R_1=1, \dots, R^{N_p-1}=1$ [S2].
- **Formula:** The total number of MLPs running is calculated as $\sum_{m=1}^{N_p-1} \frac{N}{N_p} R^m = N \frac{N_p-1}{N_p} < N$ [S2].
- **Number:** On the Blender dataset with $N_p=4$, MIMO-NeRF-self-R3 achieves PSNR 30.89, SSIM 0.949, LPIPS 0.060, and T-time 5.05h [S2]. On the LLFF dataset with $N_p=4$, MIMO-NeRF-self-R3 achieves PSNR 27.27, SSIM 0.860, LPIPS 0.171, and T-time 3.78h [S2]. On the Blender dataset with $N_p=8$, MIMO-NeRF-self-R3 achieves PSNR 29.88, SSIM 0.937, LPIPS 0.080, and T-time 7.75h [S2]. On the LLFF dataset with $N_p=8$, MIMO-NeRF-self-R3 achieves PSNR 25.66, SSIM 0.797, LPIPS 0.243, and T-time 5.97h [S2].

#### GARF
- **Definition:** A method using Gaussian activations as an alternative to positional encoding in NeRF, combined with bundle adjustment for pose estimation from unknown camera poses [S3].
- **Step:** Uses Gaussian activations instead of positional encoding [S3]. Performs bundle adjustment for pose estimation [S3].
- **Number:** Successfully recovers scene representations from unknown camera poses in challenging scenes with low-textured regions [S3].

#### NeRF-VAE
- **Definition:** A generative NeRF model that generalizes to out-of-distribution scenes and removes the need to train on each scene from scratch [S3].
- **Step:** The NeRF renderer is conditioned on a latent code. This latent code is trained using Iterative Amortized Inference and a ResNet encoder [S3]. An attention-based scene function is introduced instead of the typical MLP [S3].
- **Comparison:** Outperforms baseline NeRF with a low number of scene views (5-20), but is outperformed by baseline NeRF when a large number of views are available (100+) due to lower scene expressivity [S3].

#### Fig-NeRF
- **Definition:** Scene composition model for object interpolation and amodal segmentation using two separate NeRF models for foreground and background [S3].
- **Step:** Use an appearance NeRF conditioned on latent codes for the background model [S3]. Use two photometric losses, one for the foreground and one for the background [S3].

#### HyperNeRF
- **Definition:** A method extending the canonical space to a higher dimension and using a slicing MLP to describe how to return to the 3D representation using ambient space coordinates [S3].
- **Step:** Use the canonical coordinate and ambient space coordinate to condition the usual density and color MLPs of baseline NeRF models [S3].
- **Comparison:** Achieved great results in synthesizing views in scenes with topological changes, such as a human opening their mouth or a banana being peeled [S3].

#### FastSR-NeRF
- **Definition:** A method introducing a simple super-resolution/upsampling CNN into the NeRF pipeline. It trains a small, fast, and efficient NeRF model to produce low-resolution 3D consistent features and uses a fast SR model to upscale these features [S3].
- **Step:** Introduce a novel augmentation technique called random patch sampling, which improves SR performance by increasing patch diversity [S3].
- **Comparison:** Unlike prior NeRF+SR methods that rely on complex training procedures, distillation, or high-resolution reference images, FastSR-NeRF requires no architectural changes or heavy computing [S3].

#### LISA
- **Definition:** A method targeting the modeling of hands by approximating human hands with a collection of rigid parts [S3].
- **Step:** Query points are input into MLPs, which are used to predict geometry (via SDFs) and color [S3].

#### Block-NeRF
- **Definition:** A method for city-scale NeRF-based reconstruction from street-level images that handles transient appearance and moving objects by dividing neighborhoods into overlapping blocks [S3].
- **Step:** Build each individual Block-NeRF on mip-NeRF using its IPE (Integrated Positional Encoding) and NeRF-W using its appearance latent code optimization [S3]. Use semantic segmentation to mask out transient objects such as pedestrians and cars during NeRF training [S3]. Train a visibility MLP in parallel, supervised using the transitivity function Eq. (3) and the density value generated by the NeRF MLP [S3]. Use the visibility MLP to discard low-visibility Block-NeRFs [S3]. Divide neighborhoods into blocks, train a Block-NeRF on each, and assign an overlap between blocks [S3]. Sample images from overlapping Block-NeRFs and composite them using inverse distance weighting after an appearance code matching optimization [S3].

#### PIE-NeRF
- **Definition:** A pipeline integrating physics-based elastodynamic simulation with a neural radiance field representation, using a meshless discretisation based on quadratic generalized moving least squares on top of the density field of a NeRF [S3].
- **Step:** Enables large deformations and nonlinear hyperelastic material simulation directly on the implicit representation and supports interactive manipulation of real-world captured scenes with complex geometries [S3].

#### DäRF Implementation Details
- **Parameter:** NeRF Backbone: K-planes [36] [S4, unofficial]. MDE Model: DPT-hybrid [38] [S4, unofficial]. Optimizer: Adam [21] [S4, unofficial]. NeRF Learning Rate: 1e-2 [S4, unofficial]. MDE Learning Rate: 1e-5 [S4, unofficial]. LR Schedule: Cosine warmup [S4, unofficial].

#### Few-shot NeRF
- **Definition:** NeRF represents a scene as a continuous function $F_\theta(\cdot)$ implemented by a neural network with parameters $\theta$, taking 3D coordinates $\mathbf{x}$ and viewing direction $\mathbf{d}$ with positional encoding $\gamma(\cdot)$ to output color $\mathbf{c}$ and density $\sigma$ [S4, unofficial].
- **Parameter:** Few-shot NeRF setting: number of input viewpoints $|S|$ is drastically reduced to a few viewpoints, specifically $|S| < 20$ [S4, unofficial].

#### Time-dependent NeRF
- **Definition:** A NeRF variant that represents scene geometry and appearance at each specific time step $t$, trained using neighboring frames around time $t$, to capture local scene geometry and motion [S5].
- **Step:** Represent scene appearance using mapping function $\mathbf{c} = \phi_c(\mathbf{x}, \mathbf{d})$ taking point $\mathbf{x} \in \mathbb{R}^3$ and viewing direction $\mathbf{d} \in \mathbb{R}^2$ to predict color $\mathbf{c} \in \mathbb{R}^3$ [S5]. Represent scene geometry via mapping function $s = \phi_g(\mathbf{x})$ which maps a 3D point $\mathbf{x}$ to a signed distance value $s \in \mathbb{R}$ [S5]. For a camera ray $r(h) = o + hd$, sample $K$ 3D points $\{x_i(r) = r(h_i) \mid i = 1, \dots, K, h_i < h_{i+1}\}$ [S5]. Use the time-dependent NeRF to compute SDF flows, which are used to constrain the learning of continuous camera motion via the linear relationship between SDF flow and scene flow [S5].
- **Formula:** Convert signed distance $s_i$ to density $\sigma_i$ using: $\sigma_i = \max((\Phi(s_i) - \Phi(s_{i+1}))/\Phi(s_i), 0)$, where $\Phi(s) = 1/(1+e^{1-\gamma s})$, $\gamma$ is a learnable parameter, and $i$ is the point index [S5]. Render pixel color $\hat{C}(r)$ via volume rendering: $\hat{C}(r) = \sum_{i=1}^{K} L_i \sigma_i c_i$, where $L_i = \prod_{j=1}^{i-1} (1 - \sigma_j)$ is the accumulated transmittance [S5].

#### NeRF Model Architecture
- **Definition:** The neural network architecture for NeRF, consisting of dense layers with ReLU activations, potentially using view directions as input [S6, unofficial].
- **Parameter:** `D`: The depth of the network (number of hidden layers). Default is 8 [S6, unofficial]. `input_ch_views`: The number of input channels for view directions. Default is 3 [S6, unofficial]. `output_ch`: The number of output channels. Default is 4 (RGB + density) [S6, unofficial]. `use_viewdirs`: Boolean flag to determine if view directions are used as input to the network [S6, unofficial].

#### NeuMan Network Architecture
- **Definition:** A NeRF architecture consisting of a coarse sub-model and a fine sub-model for the scene, and a single sub-model for the human NeRF model, leveraging SMPL geometry priors [S7].
- **Layout:** Architecture visualization (Fig. 10): Blue boxes are raw inputs, green boxes are positional encoded values, black boxes are hidden layers, pink boxes are final outputs. Numbers indicate feature dimensions. '+' indicates concatenation. ReLU is applied after each hidden layer [S7].
- **Comparison:** Unlike NeuralBody which overfits to training observations and produces poor rendering on the back of the subject, NeuMan generalizes better and faithfully renders the back [S7].

## Background
The original NeRF architecture [Mildenhall et al. 2020] used a simple MLP to map 3D coordinates and view directions to density and color [S4, S6, unofficial]. Subsequent research identified limitations in sparse view reconstruction, semantic understanding, and dynamic scene representation, leading to various architectural modifications. Simple-NeRF [S1, S2] and DoF-NeRF [S1] address sparse view and optical effects, respectively. Semantic and Feature Branches [S1, S2, S3] extend the output vector for auxiliary tasks. Generative models like NeRF-VAE [S3] and Fig-NeRF [S3] introduce latent codes. Large-scale methods like Block-NeRF [S3] and HyperNeRF [S3] modify the canonical space or scene decomposition. These modifications have become standard in modern NeRF implementations to handle real-world complexities.

## Sources
[S1] Rip-NeRF: Anti-aliasing Radiance Fields with Ripmap-Encoded Platonic Solids, https://arxiv.org/abs/2405.02386v1, fetched 2026-10-26
[S2] MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields, https://arxiv.org/abs/2310.01821v1, fetched 2026-10-26
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S4] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
[S5] Joint Optimization of Neural Radiance Fields and Continuous Camera Motion from a Monocular Video, https://doi.org/10.1109/cvpr52734.2025.01071, fetched 2026-10-26
[S6] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
[S7] NeuMan: Neural Human Radiance Field from a Single Video, https://doi.org/10.1007/978-3-031-19824-3_24, fetched 2026-10-29
