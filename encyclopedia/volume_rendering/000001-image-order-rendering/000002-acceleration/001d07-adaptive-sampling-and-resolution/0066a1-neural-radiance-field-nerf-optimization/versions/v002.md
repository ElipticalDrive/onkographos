# Neural Radiance Field (NeRF) Optimization


## Overview
NeRF Optimization trains a continuous volumetric scene representation, typically parameterized by a Multi-Layer Perceptron (MLP), to predict density and color from 3D coordinates and viewing directions [G]. It solves novel view synthesis by learning a differentiable volume rendering function that maps camera parameters to pixel colors via ray marching [G]. It sets itself apart from discrete mesh-based rendering by modeling geometry as a continuous density field, allowing for high-fidelity reconstruction of complex, unstructured scenes without explicit topology [G].

## Shared principles
Builds on [Adaptive Sampling and Resolution](1412-adaptive-sampling-and-resolution.md).

## Subtopics

### [Two-stage NeRF Rendering](925-two-stage-nerf-rendering.md)
Decouples density estimation from color evaluation using a coarse pass to guide importance sampling in a fine pass. Chosen when computational budget is limited but high-frequency detail is required.

### [Adaptive sampling and reconstruction](10469-adaptive-sampling-and-reconstruction.md)
Uses an importance network to predict sparse sample locations and a reconstruction network to upsample the result. Chosen for interactive applications where end-to-end differentiability and sparse querying are prioritized over raw ray-casting speed.

### [SteerNeRF](17152-steernerf.md)
Exploits temporal smoothness in continuous viewpoint trajectories by reprojecting features from previous frames. Chosen for real-time interactive navigation where camera motion is smooth and previous frames are available.

### [Score Distillation Sampling (SDS)](21179-score-distillation-sampling-sds.md)
Optimizes the NeRF using gradients from a frozen 2D diffusion model rather than ground-truth images. Chosen for 3D generation from text or image priors where explicit 3D training data is unavailable.

## Background
NeRF was introduced by Mildenhall et al. in 2020 [G]. The method represents a scene as a continuous 5D function mapping $(x, y, z, \theta, \phi)$ to volume density and view-dependent color [G]. Optimization involves minimizing the photometric loss between rendered images and ground-truth views using stochastic gradient descent [G]. The foundational approach uses uniform sampling along rays, which is computationally expensive [G]. Subsequent works have focused on acceleration through adaptive sampling, hierarchical representations, and alternative optimization objectives like Score Distillation Sampling [G].

## Sources
