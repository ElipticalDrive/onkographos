# Monocular Depth Estimation Adaptation and Supervision

Also called: MDE Adaptation, Depth-Guided NeRF, Depth Regularization

## Overview
This family of methods integrates pre-trained Monocular Depth Estimation (MDE) networks into NeRF optimization to resolve the shape-radiance ambiguity inherent in implicit volume representations, particularly in sparse-view or few-shot settings [S2, unofficial]. Unlike standard NeRF which relies solely on photometric consistency, these methods use MDE outputs as geometric priors to constrain ray sampling, guide volume rendering, or directly supervise the NeRF's depth output [S3]. They differ from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by introducing external geometric constraints, and from [Regularization and Loss Functions](8586-regularization-and-loss-functions.md) by specifically leveraging the semantic geometric understanding of MDE models rather than generic smoothness penalties.

## Shared principles
All methods in this family share the core mechanism of aligning the relative, scale-ambiguous depth estimates from a pre-trained MDE network ($D^*$) with the absolute, metric depth rendered from the NeRF ($\bar{D}$). Since MDE networks output relative depth lacking absolute scale and shift, direct pixel-wise loss is insufficient and prone to bias from dominant objects [S1]. Therefore, a critical shared step is the computation of local scale ($w$) and shift ($q$) parameters.

The scale and shift are typically computed per-patch $P$ by minimizing the least squares error between the aligned MDE depth and the rendered NeRF depth:
$$ \min_{w, q} \sum_{p \in P} \| (w \cdot \text{sg}(D^*(p)) + q) - \bar{D}(p) \|^2 $$
where $\text{sg}(\cdot)$ denotes the stop-gradient operator, preventing gradients from flowing into the pre-trained MDE network during this alignment step [S2, unofficial]. This local fitting alleviates global bias caused by dominant objects [S1].

The total loss often combines a photometric term with a depth consistency term. The depth term may use L1 or L2 norms, though the source notation for the final loss is often generic $\| \cdot \|$ [S1]. In some variants, the MDE network itself is adapted (fine-tuned) jointly with the NeRF to enforce multi-view consistency and remove inherent MDE ambiguities (e.g., convexity/concavity confusion) [S2, unofficial].

## Subtopics

### [NerfingMVS](9518-nerfingmvs.md)
NerfingMVS is a method for indoor multi-view stereo that optimizes Neural Radiance Fields (NeRF) using depth priors to resolve the shape-radiance ambiguity inherent in implicit volume representations. It differs from standard NeRF by explicitly incorporating external depth estimates to guide volume sampling and constrain ray bounds, thereby improving geometric accuracy without relying on explicit pixel matching across views.

### [Depth-Guided NeRF Training via Earth Mover’s Distance](11911-depth-guided-nerf-training-via-earth-mover-s-distance.md)
This method improves the geometric understanding of Neural Radiance Fields (NeRFs) trained on sparse views by supervising the distribution of ray termination distances using Earth Mover’s Distance (EMD) against a monocular depth prior, rather than enforcing direct pixel-wise depth consistency. Unlike sibling methods such as [DäRF](12331-d-rf.md) which minimize the norm between rendered depth and prior, or [SCADE](9518-nerfingmvs.md) which relies on multiple ambiguous hypotheses, this approach uses a single depth prior from a diffusion model and weights the supervision by intrinsic uncertainty maps, allowing the NeRF to selectively incorporate prior information without propagating prior inaccuracies.

### [Monocular Depth Regularization on Seen Views](12330-monocular-depth-regularization-on-seen-views.md)
This method regularizes a Neural Radiance Field (NeRF) by aligning pseudo-depth maps from a pre-trained monocular depth estimation (MDE) network with the absolute depth rendered from the NeRF for seen input views. It solves the problem that MDE networks output relative depth lacking absolute scale and shift, which makes direct pixel-wise loss insufficient and prone to bias from dominant objects. It sets itself apart from global statistic fitting by computing scale and shift parameters locally on image patches, thereby preserving local depth statistics and reducing the computational cost of full-image rendering.

### [DäRF](12331-d-rf.md)
DäRF is a framework for robust few-shot NeRF reconstruction that combines Neural Radiance Fields (NeRF) with Monocular Depth Estimation (MDE) through online complementary training. It solves the problem of poor generalization in sparse-view NeRF by injecting MDE's geometric prior into the NeRF representation at both seen and unseen viewpoints, while simultaneously adapting the MDE network to produce multiview-consistent geometry aligned with the NeRF. Unlike methods that only distill depth from pretrained MDE on seen views, DäRF jointly optimizes both networks to remove MDE ambiguity via patch-wise scale-shift fitting and confidence modeling.

### [NeRF-guided MDE Adaptation](12333-nerf-guided-mde-adaptation.md)
NeRF-guided MDE Adaptation is a method that aligns a pre-trained Monocular Depth Estimation (MDE) network with the absolute geometry of a specific scene by using a NeRF as a geometric prior. It solves the ambiguity inherent in monocular depth estimates, such as the inability to distinguish surface convexity/concavity or flat surface orientation, by providing scene-specific absolute depth priors and multi-view consistency. Unlike sibling methods that primarily use depth for loss supervision or sampling guidance, this method explicitly adapts the MDE weights to the target scene, often jointly optimizing the NeRF and the MDE to mutually reinforce geometric accuracy.

### [Model Variants](13153-model-variants.md)
This family defines three specific configurations for the Forward Warping Depth (FWD) framework, distinguished by their source of depth supervision and the training status of the Monocular Depth Estimation (MDE) module. They solve the problem of how to integrate depth priors into NeRF optimization when the availability of ground-truth depth or pre-trained MDE networks varies. FWD-U represents a fully unsupervised baseline where the depth network is trained from scratch, while FWD leverages a pre-trained MVS module that is fine-tuned end-to-end. FWD-D bypasses the MVS module entirely, using sensor depths as direct inputs.

### [ShadowNeuS](14132-shadowneus.md)
ShadowNeuS reconstructs a neural Signed Distance Field (SDF) of a scene from single-view images under multiple lighting conditions by supervising shadow rays (rays between the light source and the scene) rather than just camera rays. It solves the problem that standard NeRF does not model rays from the scene to the light source, missing shadow information which contains geometry cues via light visibility. Unlike NeRF which uses multi-view cameras as ray sensors, this method uses a single-view camera and shadow rays to provide supervision, making it applicable to real capture setups with limited views.

### [NeRF-Supervision](14729-nerf-supervision.md)
NeRF-Supervision is a method for generating dense correspondence datasets to train object-centric visual descriptors using only RGB images and Neural Radiance Fields (NeRFs). It solves the problem of acquiring ground-truth dense matches for thin, reflective, or ambiguous objects that fail standard RGB-D or Multi-View Stereo (MVS) pipelines. Unlike MVS, which relies on patch-based photometric consistency, NeRF-Supervision uses the NeRF’s density field to probabilistically sample correspondences, handling ambiguity via a "distribution-of-depths" formulation rather than a single depth map.

#### Depth Completion Network NeRF
- [S1] Extract sparse point cloud using COLMAP.
- [S1] Introduce a depth loss based on predicted depth and uncertainty in addition to the standard volumetric loss.
- [S1] Train on RGB-D data from ScanNet and Matterport3D with Gaussian noise added to the depth.

#### Monocular Depth Regularization on Unseen Views
- [S2, unofficial] A loss function that supervises NeRF geometry at unseen/novel viewpoints by aligning NeRF-rendered depth with monocular depth estimated from the rendered novel view image.
- [S2, unofficial] For each unseen view $l$ in the set of unseen views $U$: 1. Sample a patch of the novel viewpoint. 2. Render color $\bar{I}_l$ and depth $\bar{D}_l$ from NeRF. 3. Extract monocular depth $\bar{D}_l^* = G_\phi(\bar{I}_l)$ from the rendered color image. 4. Compute scale $w_l$ and shift $q_l$ via least squares between $\bar{D}_l^*$ and $\bar{D}_l$. 5. Calculate loss $L_{unseen}$.
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
The integration of monocular depth priors into NeRF optimization emerged as a solution to the limitations of sparse-view reconstruction, where photometric loss alone leads to shape-radiance ambiguity and poor geometry [S2, unofficial]. Early works like NerfingMVS (September 2021) utilized COLMAP sparse point clouds and pre-trained monocular depth networks to guide volume sampling [S3]. Subsequent developments, such as DäRF (2023), introduced joint optimization of the NeRF and the MDE network to enforce multi-view consistency and adapt the MDE to the specific scene geometry [S2, unofficial]. More recent approaches leverage diffusion-based depth estimators and Earth Mover’s Distance to supervise the distribution of ray termination distances, further refining geometric accuracy in few-shot settings [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
[S3] Reference-guided Controllable Inpainting of Neural Radiance Fields, https://doi.org/10.1109/ICCV51070.2023.01633, fetched 2026-10-29
