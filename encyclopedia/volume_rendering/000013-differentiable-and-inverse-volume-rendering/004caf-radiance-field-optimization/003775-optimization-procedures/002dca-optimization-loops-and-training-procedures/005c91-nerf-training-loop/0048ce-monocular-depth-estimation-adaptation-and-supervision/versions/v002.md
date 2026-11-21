# Monocular Depth Estimation Adaptation and Supervision

Also called: MDE Adaptation, Depth-Guided NeRF, Monocular Depth Regularization

## Overview
This family of methods integrates Monocular Depth Estimation (MDE) priors into Neural Radiance Field (NeRF) training to resolve the shape-radiance ambiguity and improve geometric accuracy, particularly in sparse-view or few-shot settings [S1][S2, unofficial]. Unlike standard NeRF which relies solely on photometric consistency, these methods use MDE to guide volume sampling bounds, regularize depth distributions via Earth Mover’s Distance, or adapt the MDE network itself to the scene's absolute geometry through bidirectional knowledge transfer [S1][S2, unofficial]. They differ from sibling methods like [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by explicitly injecting external geometric constraints derived from 2D depth priors into the 3D optimization loop [S1][S2, unofficial].

## Shared principles
All methods in this family rely on the extraction of a pseudo-depth map $D^*$ from a pre-trained MDE network $G_\phi$ given an input image $I$ [S1][S2, unofficial]. This prior is used to constrain or supervise the NeRF's rendered depth $\bar{D}$, which is computed as the expected distance of ray termination:
$$ \bar{D}(r) = \sum_i w_i y_i $$
where $w_i$ is the weight of sample $i$ and $y_i$ is its distance from the camera [S1][S2, unofficial].

To align the relative scale and shift of the monocular prior with the NeRF's coordinate system, a linear alignment is typically performed. For a set of pixels $P$, scale $w$ and shift $q$ are computed via least squares:
$$ \min_{w, q} \sum_{p \in P} \| (w \cdot D^*(p) + q) - \bar{D}(p) \|^2 $$
The stop-gradient operator $\text{sg}(\cdot)$ is applied to the MDE output or NeRF output depending on the direction of supervision to prevent gradient flow into the frozen or separately updated network [S1][S2, unofficial].

Methods often employ patch-wise fitting rather than global fitting to avoid skewing by dominant objects and to handle local geometric details [S1][S2, unofficial]. Confidence modeling is also a shared principle in advanced variants, where reliability masks $M(p)$ are used to weight losses, mitigating errors from noisy priors or NeRF artifacts [S1][S2, unofficial].

## Subtopics

### [NerfingMVS](9518-nerfingmvs.md)
NerfingMVS focuses on indoor multi-view stereo by using depth priors to guide volume sampling bounds. It differs from standard NeRF by explicitly incorporating sparse depth estimates from Structure-from-Motion (SfM) and dense depth estimates from a fine-tuned monocular network to restrict the sampling interval to relevant depths, rather than relying on uniform or stratified sampling. It is chosen when sparse 3D point clouds are available and the goal is to resolve shape-radiance ambiguity in indoor scenes.

### [Depth-Guided NeRF Training via Earth Mover’s Distance](11911-depth-guided-nerf-training-via-earth-mover-s-distance.md)
This method improves geometric understanding in sparse-view settings by supervising the distribution of ray termination distances using Earth Mover’s Distance (EMD) against monocular depth priors, rather than enforcing direct depth matching. It differs from siblings like [Monocular Depth Regularization on Seen Views](12330-monocular-depth-regularization-on-seen-views.md) by utilizing a diffusion-based depth prior (DiffDP) to generate uncertainty maps, which dynamically weight the EMD loss to prevent the NeRF from learning incorrect geometry from noisy priors. Unlike [DäRF](12331-d-rf.md) which minimizes the norm between rendered and prior depth, this approach guides the probability distribution of ray terminations, avoiding the collapse of multimodal distributions and reducing interference with photometric learning.

### [Monocular Depth Regularization on Seen Views](12330-monocular-depth-regularization-on-seen-views.md)
This method applies a consistency loss between NeRF-rendered depth and pseudo-depth from a pre-trained monocular depth estimator (MDE) specifically for training views (seen views). It differs from global scale/shift fitting by computing alignment parameters per image patch, which prevents dominant objects from skewing the depth alignment and improves local geometric accuracy. It builds on the shared principles of depth rendering and MDE extraction [Parent].

### [DäRF](12331-d-rf.md)
DäRF is a framework for robust few-shot NeRF reconstruction that combines Neural Radiance Fields with Monocular Depth Estimation (MDE) through online complementary training. It addresses the ambiguity problems inherent in monocular depth by jointly optimizing the NeRF and MDE networks at a specific scene, adapting MDE to NeRF's absolute scaling and multiview consistency while using MDE's geometry prior to regularize NeRF. Unlike methods that only apply pretrained MDE to seen views, DäRF distills the monocular depth prior to NeRF at both seen and unseen viewpoints.

### [NeRF-guided MDE Adaptation](12333-nerf-guided-mde-adaptation.md)
NeRF-guided MDE Adaptation is a method that aligns a pre-trained Monocular Depth Estimation (MDE) network with the absolute geometry of a specific scene by using a NeRF model as a geometric prior. It solves the ambiguity inherent in monocular depth estimation, such as the inability to distinguish surface convexity/concavity or determine the orientation of flat surfaces, by transferring the NeRF's scene-specific absolute depth and multi-view consistency knowledge to the MDE. Unlike sibling methods that only use MDE to regularize NeRF training, this method explicitly adapts the MDE parameters to the target scene, creating a bidirectional knowledge transfer where the MDE provides geometric priors to the NeRF and the NeRF provides absolute scale to the MDE.

### Depth Completion Network NeRF
- [S1] Extract sparse point cloud using COLMAP.
- [S1] Introduce a depth loss based on predicted depth and uncertainty in addition to the standard volumetric loss.
- [S1] Train on RGB-D data from ScanNet and Matterport3D with Gaussian noise added to the depth.

### Monocular Depth Regularization on Unseen Views
- [S2, unofficial] A loss function that supervises NeRF geometry at unseen/novel viewpoints by aligning NeRF-rendered depth with monocular depth estimated from the rendered novel view image.
- [S2, unofficial] For each unseen view $l$ in the set of unseen views $U$: 1. Sample a patch of the novel viewpoint. 2. Render color $\bar{I}_l$ and depth $\bar{D}_l$ from NeRF. 3. Extract monocular depth $\bar{D}^*_l = G_\phi(\bar{I}_l)$ from the rendered color image. 4. Compute scale $w_l$ and shift $q_l$ via least squares between $\bar{D}^*_l$ and $\bar{D}_l$. 5. Calculate loss $L_{unseen}$.
- [S2, unofficial] $U$: Set of unseen view images.
- [S2, unofficial] $P$: Randomly sampled patch of pixels.
- [S2, unofficial] Monocular depth from noisy NeRF rendering may be affected by fine-grained artifacts. The MDE model's strong geometric prior is robust against these artifacts, filtering them out.
- [S2, unofficial] Unlike previous methods [14, 41, 61, 50] that only impose priors on input viewpoints (limited by self-occlusion/sparsity), this method applies priors to arbitrary viewpoints, reaching every location.

### MDE Adaptation Loss
- [S2, unofficial] A composite loss function for adapting a monocular depth estimator to predict scene-specific absolute geometry by enforcing multi-view consistency and local patch-wise scale-shift fitting.
- [S2, unofficial] Compute the first addend term (scale-shift loss) to force the MDE to adapt towards multiview consistency, reducing its ill-posed nature and aligning its initial global depth prediction with the absolute geometry captured by NeRF.
- [S2, unofficial] Compute the second addend term (L1 loss with patch-wise scale-shift fitting) to aid the modeling of fine, detailed, local geometry that the model struggles to capture without local fitting.
- [S2, unofficial] Optimize the MDE network by minimizing the sum of both terms simultaneously; using only one term performs worse on all metrics than using both in conjunction.
- [S2, unofficial] Scale-shift loss weight: Balances global alignment with NeRF geometry; value not specified in this section but required for optimization.
- [S2, unofficial] L1 loss weight: Balances local geometric detail preservation via patch-wise fitting; value not specified in this section but required for optimization.

## Background
The integration of monocular depth priors into NeRF optimization emerged as a solution to the ill-posed nature of few-shot NeRF reconstruction, where photometric loss alone often leads to shape-radiance ambiguity [S1][S2, unofficial]. Early works like NerfingMVS (September 2021) utilized COLMAP sparse point clouds and fine-tuned monocular networks to guide sampling bounds [S2, unofficial]. Concurrently, Roessle et al. (April 2021) used COLMAP point clouds processed by Depth Completion Networks to introduce depth and uncertainty losses [S1]. More recent frameworks like DäRF (2023) and Earth Mover’s Distance-guided methods have advanced this by enabling bidirectional adaptation between NeRF and MDE networks, using diffusion-based priors and distributional matching to handle uncertainty and multiview consistency [S1][S2, unofficial]. These methods are primarily used in sparse-view reconstruction, indoor multi-view stereo, and few-shot 3D scene modeling [S1][S2, unofficial].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
