# Monocular Depth Estimation Adaptation and Supervision

Also called: MDE Adaptation, Depth-Guided NeRF, Monocular Depth Regularization

## Overview
This family of methods integrates pre-trained Monocular Depth Estimation (MDE) networks into Neural Radiance Field (NeRF) training to resolve the shape-radiance ambiguity and scale uncertainty inherent in sparse-view or single-view reconstruction [S2, unofficial]. By supervising the NeRF's rendered depth with pseudo-depth maps from MDE, these methods enforce geometric consistency that photometric loss alone cannot provide [S2, unofficial]. They differ from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by introducing external geometric priors, and from [Regularization and Loss Functions](8586-regularization-and-loss-functions.md) by specifically targeting depth distribution alignment rather than just density smoothness.

## Shared principles
All methods in this family rely on the fundamental volume rendering equation to compute rendered depth $\bar{D}$ from a NeRF model $\mathcal{N}$ [Parent]. For a ray $r$ with samples at distances $y_i$, densities $\sigma_i$, and colors $c_i$, the transmittance $T_i$ and weight $w_i$ are computed as:
$$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
$$ w_i = T_i (1 - \exp(-\sigma_i \delta_i)) $$
The rendered depth is the expected distance:
$$ \bar{D}(r) = \sum_i w_i y_i $$
where $\delta_i$ is the distance between samples [Parent].

To align the relative, scale-ambiguous MDE output $D^*$ with the absolute NeRF depth $\bar{D}$, a local scale $w$ and shift $q$ are computed per patch $P$ via least squares minimization:
$$ \min_{w, q} \sum_{p \in P} \| (w \cdot D^*(p) + q) - \bar{D}(p) \|^2 $$
This patch-wise approach prevents dominant objects from skewing the global alignment [S1]. The stop-gradient operator $\text{sg}(\cdot)$ is applied to the MDE output (or NeRF output, depending on the direction of adaptation) to control gradient flow [S1].

## Subtopics

### [NerfingMVS](9518-nerfingmvs.md)
NerfingMVS resolves shape-radiance ambiguity in indoor scenes by guiding volume sampling with sparse SfM data and a fine-tuned MDE network. Unlike [DäRF](12331-d-rf.md), which jointly optimizes MDE and NeRF, NerfingMVS uses the MDE primarily to clamp ray bounds and weight samples, producing dense depth without pixel matching. Choose this method when sparse geometric priors are available and strict volumetric sampling guidance is required.

### [Depth-Guided NeRF Training via Earth Mover’s Distance](11911-depth-guided-nerf-training-via-earth-mover-s-distance.md)
This method improves geometric accuracy in sparse-view settings by supervising the distribution of ray termination distances using Earth Mover’s Distance (EMD) rather than direct depth matching. Unlike [DäRF](12331-d-rf.md) which minimizes norm differences, or [NerfingMVS](9518-nerfingmvs.md) which assumes perfect pseudo-ground truths, this approach uses a diffusion-based depth prior (DiffDP) and its uncertainty map to selectively guide the NeRF. It prevents prior inaccuracies from propagating by weighting the EMD loss against uncertainty. Choose this method when using diffusion-based priors and needing robust handling of prior uncertainty.

### [Monocular Depth Regularization on Seen Views](12330-monocular-depth-regularization-on-seen-views.md)
This method applies a consistency loss between pseudo-depth maps from a pre-trained MDE network and NeRF-rendered depth specifically for input training views. It solves scale ambiguity by computing local scale and shift parameters per image patch via least squares, preventing dominant objects from skewing alignment. Unlike global fitting methods, this patch-wise approach also mitigates the computational bottleneck of full-image rendering. Choose this method for efficient regularization on seen views without adapting the MDE network.

### [DäRF](12331-d-rf.md)
DäRF is a framework for robust few-shot NeRF reconstruction that combines NeRF with MDE through online complementary training. It addresses monocular depth ambiguity and lack of multiview consistency by jointly optimizing the NeRF and the MDE network, adapting MDE output to NeRF's absolute scaling while using MDE geometry to regularize NeRF. Unlike methods that only apply pretrained MDE to seen views, DäRF distills the monocular depth prior to NeRF at both seen and unseen viewpoints. Choose this method when joint adaptation of geometry and appearance is required for few-shot scenarios.

### [NeRF-guided MDE Adaptation](12333-nerf-guided-mde-adaptation.md)
This method adapts a pre-trained MDE network to the specific absolute geometry of a scene by using a NeRF model as a geometric prior. It solves the ambiguity inherent in monocular depth estimation (e.g., convexity/concavity confusion) by leveraging NeRF's multi-view consistency. Unlike sibling methods that primarily use depth to regularize NeRF training, this method explicitly fine-tunes the MDE network itself to align with the NeRF's learned geometry. Choose this method when the goal is to improve the MDE network's performance on the specific scene.

### [Model Variants](13153-model-variants.md)
This family defines three specific architectural configurations for the Forward Warping Depth (FWD) framework, distinguished by their depth supervision source and the training dynamics of the depth estimation module. While the parent article establishes the general mechanism of aligning rendered NeRF depth with pseudo-depth from an MDE network, these variants determine whether that MDE network is pre-trained, randomly initialized, or bypassed entirely in favor of sensor data.

### [ShadowNeuS](14132-shadowneus.md)
ShadowNeuS reconstructs a neural Signed Distance Field (SDF) from single-view images under multiple lighting conditions by supervising shadow rays rather than relying solely on camera rays. It solves the problem that standard NeRF does not model rays from the scene to the light source, missing shadow information which contains critical geometry cues. Unlike NeRF which uses multi-view cameras, this method uses a single-view camera and shadow rays, making it applicable to real capture setups with limited views. Choose this method for single-view reconstruction with known lighting conditions.

### [NeRF-Supervision](14729-nerf-supervision.md)
NeRF-Supervision generates dense correspondence datasets to train object-centric visual descriptors using only RGB images and NeRFs. It solves the problem of obtaining ground-truth dense correspondences for thin, reflective, or ambiguous objects where traditional RGB-D sensors and MVS fail. Unlike MVS, which relies on patch-based photometric consistency, NeRF-Supervision uses the NeRF's density field to sample correspondences probabilistically, handling ambiguity via a "distribution-of-depths" formulation. Choose this method for training descriptors on challenging objects.

#### Depth Completion Network NeRF
- [S1] Extract sparse point cloud using COLMAP.
- [S1] Introduce a depth loss based on predicted depth and uncertainty in addition to the standard volumetric loss.
- [S1] Train on RGB-D data from ScanNet and Matterport3D with Gaussian noise added to the depth.

#### Monocular Depth Regularization on Unseen Views
- [S2, unofficial] A loss function that supervises NeRF geometry at unseen/novel viewpoints by aligning NeRF-rendered depth with monocular depth estimated from the rendered novel view image.
- [S2, unofficial] For each unseen view $l$ in the set of unseen views $U$: 1. Sample a patch of the novel viewpoint. 2. Render color $\bar{I}_l$ and depth $\bar{D}_l$ from NeRF. 3. Extract monocular depth $\bar{D}^*_l = G_\phi(\bar{I}_l)$ from the rendered color image. 4. Compute scale $w_l$ and shift $q_l$ via least squares between $\bar{D}^*_l$ and $\bar{D}_l$. 5. Calculate loss $L_{unseen}$.
- [S2, unofficial] $U$: Set of unseen view images.
- [S2, unofficial] $P$: Randomly sampled patch of pixels.
- [S2, unofficial] Monocular depth from noisy NeRF rendering may be affected by fine-grained artifacts. The MDE model's strong geometric prior is robust against these artifacts, filtering them out.
- [S2, unofficial] Unlike previous methods [14, 41, 61, 50] that only impose priors on input viewpoints (limited by self-occlusion/sparsity), this method applies priors to arbitrary viewpoints, reaching every location.

#### MDE Adaptation Loss
- [S2, unofficial] A composite loss function for adapting a monocular depth estimator to predict scene-specific absolute geometry by enforcing multi-view consistency and local patch-wise scale-shift fitting.
- [S2, unofficial] Compute the first addend term (scale-shift loss) to force the MDE to adapt towards multiview consistency, reducing its ill-posed nature and aligning its initial global depth prediction with the absolute geometry captured by NeRF.
- [S2, unofficial] Compute the second addend term (L1 loss with patch-wise scale-shift fitting) to aid the modeling of fine, detailed, local geometry that the model struggles to capture without local fitting.
- [S2, unofficial] Optimize the MDE network by minimizing the sum of both terms simultaneously; using only one term performs worse on all metrics than using both in conjunction.
- [S2, unofficial] Scale-shift loss weight: Balances global alignment with NeRF geometry; value not specified in this section but required for optimization.
- [S2, unofficial] L1 loss weight: Balances local geometric detail preservation via patch-wise fitting; value not specified in this section but required for optimization.

#### Weighted Disparity Alignment
- [S3] Predict uncalibrated disparity of the source view using a monocular depth estimator: $D_e = D(I_r)$.
- [S3] Define the aligned predicted inverse depth $D_r$ using four parameters $a_0, a_1, a_2, a_3$ and matrices $H, V$: $D_r = a_0 D_e + a_1 \mathbf{1}_{HW} + a_2 H + a_3 V$.
- [S3] Define $H$ and $V$ as $H \times W$ matrices where for pixel $p=(p_x, p_y)$, $H(p)=p_x$ and $V(p)=p_y$.
- [S3] Set weight $w(p)$ to the inverse of the distance between pixel $p$ and the mask centre-of-mass.
- [S3] Perform an additional optimization step to correct $D_r$ to encourage smoothness around the mask, yielding $D_r^{smooth}$.
- [S3] $N_{depth}$: Number of iterations for supervised training on unmasked pixels before applying depth loss.

## Background
The integration of monocular depth priors into NeRF optimization emerged as a solution to the ill-posed nature of single-view and sparse-view reconstruction, where photometric loss alone often leads to shape-radiance ambiguity [S2, unofficial]. Early works like NerfingMVS utilized sparse SfM points to guide sampling [S3]. Subsequent methods like DäRF introduced joint optimization of MDE and NeRF to enforce multiview consistency [S2, unofficial]. More recent approaches leverage diffusion-based depth estimators and Earth Mover’s Distance to handle prior uncertainty more robustly [S1]. These techniques are widely used in few-shot NeRF, inpainting, and single-view reconstruction tasks [S1][S2, unofficial][S3].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
[S3] Reference-guided Controllable Inpainting of Neural Radiance Fields, https://doi.org/10.1109/ICCV51070.2023.01633, fetched 2026-10-29
