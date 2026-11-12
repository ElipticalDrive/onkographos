# Monocular Depth Estimation Adaptation and Supervision

Also called: MDE-NeRF, Depth-Guided NeRF, Monocular Depth Regularization

## Overview
This family of methods integrates pre-trained Monocular Depth Estimation (MDE) networks into the NeRF training loop to resolve the shape-radiance ambiguity and scale uncertainty inherent in sparse-view or single-view reconstruction [S2, unofficial]. Unlike standard NeRF which relies solely on photometric consistency, these methods use MDE outputs as geometric priors to guide volume sampling, regularize density distributions, or directly adapt the MDE network to the scene's absolute geometry [S2, unofficial]. They are distinguished from sibling methods like [DäRF](12331-d-rf.md) by their specific strategies for handling the relative nature of monocular depth, ranging from sparse prior fine-tuning [S2, unofficial] to distribution-based Earth Mover’s Distance supervision [11911-depth-guided-nerf-training-via-earth-mover-s-distance.md].

## Shared principles
All methods in this family share the following mechanism:
1.  **Pseudo-Depth Generation:** A pre-trained MDE network $G_\phi$ (e.g., MiDaS, DiffDP) is applied to input images $I$ to generate relative depth maps $D^*$ [S2, unofficial].
2.  **Scale and Shift Alignment:** Since $D^*$ is relative and scale-ambiguous, it must be aligned with the absolute depth $\bar{D}$ rendered by the NeRF. This is typically done via a linear transformation $w D^* + q$, where $w$ (scale) and $q$ (shift) are computed per-patch or globally using least squares to minimize $\| (w D^* + q) - \bar{D} \|^2$ [12330-monocular-depth-regularization-on-seen-views.md].
3.  **Stop-Gradient:** To prevent gradients from flowing into the frozen MDE network during NeRF optimization, the stop-gradient operator $\text{sg}(\cdot)$ is applied to the MDE output or the NeRF depth depending on the direction of supervision [12330-monocular-depth-regularization-on-seen-views.md].
4.  **Patch-wise Processing:** To avoid global scale bias from dominant objects, alignment and loss computation are often performed on local image patches $P$ rather than the full image [12330-monocular-depth-regularization-on-seen-views.md].

## Subtopics

### [NerfingMVS](9518-nerfingmvs.md)
Optimizes NeRF using sparse SfM priors to fine-tune an MDE network, then uses the resulting dense priors to clamp ray sampling bounds. Chosen when sparse geometric structure is available and volume sampling efficiency is critical.

### [Depth-Guided NeRF Training via Earth Mover’s Distance](11911-depth-guided-nerf-training-via-earth-mover-s-distance.md)
Supervises the distribution of ray termination distances using EMD weighted by diffusion-based uncertainty. Chosen for sparse-view settings where distributional consistency is preferred over point-wise matching.

### [Monocular Depth Regularization on Seen Views](12330-monocular-depth-regularization-on-seen-views.md)
Applies a consistency loss between rendered depth and MDE pseudo-depth only on input views, using patch-wise alignment. Chosen for basic geometric regularization without modifying the sampling strategy.

### [DäRF](12331-d-rf.md)
Jointly optimizes NeRF and MDE to adapt the MDE network to the scene's absolute geometry. Chosen when multiview consistency of the MDE output itself is required, not just the NeRF geometry.

### [NeRF-guided MDE Adaptation](12333-nerf-guided-mde-adaptation.md)
Adapts a pre-trained MDE network to the specific absolute geometry of a scene by using NeRF as a prior. Chosen when the goal is to improve the MDE network's performance on the specific scene domain.

### [Model Variants](13153-model-variants.md)
Defines three configurations for the Forward Warping Depth (FWD) framework, differing in MDE initialization (pre-trained, random, or sensor-based). Chosen based on the availability of initial depth priors and sensor data.

## Background
The integration of monocular depth priors into NeRF optimization emerged to address the limitations of sparse-view reconstruction, where photometric loss alone leads to ambiguous geometry [S2, unofficial]. Early works like NerfingMVS (2021) utilized sparse SfM points to fine-tune MDE networks for guided sampling [S1]. Subsequent methods like DäRF (2023) introduced joint optimization to adapt MDE networks to specific scenes, improving multiview consistency [S2, unofficial]. More recent approaches leverage diffusion-based depth estimators (e.g., DiffDP) and distributional losses like Earth Mover’s Distance to handle uncertainty and avoid direct pixel-wise matching errors [11911-depth-guided-nerf-training-via-earth-mover-s-distance.md].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
