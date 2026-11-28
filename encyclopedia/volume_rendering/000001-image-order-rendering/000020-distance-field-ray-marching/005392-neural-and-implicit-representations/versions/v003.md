# Neural and Implicit Representations


## Overview
Neural and Implicit Representations render scenes by evaluating continuous scalar fields—parameterized as neural networks, Gaussian mixtures, or radial basis functions—along camera rays to compute geometry and appearance [1036-ray-marching-on-3d-gaussians.md][2409-ray-marching-with-neural-sdf.md][9641-rbf-particle-volume-path-tracing.md][15259-metanlr.md]. This family replaces analytical primitives or explicit meshes with learned or weighted fields, enabling high-fidelity rendering of complex structures through ray marching or path tracing.

## Shared principles

## Subtopics

### [Ray-marching on 3D Gaussians](1036-ray-marching-on-3d-gaussians.md)
Evaluates a continuous Gaussian mixture field along camera rays to compute accurate volumetric integrals, explicitly handling primitive overlap and self-attenuation [1036-ray-marching-on-3d-gaussians.md]. It serves as a ground-truth baseline to analyze approximations in standard rasterization-based 3D Gaussian Splatting, such as ignoring spatial overlap and using approximate projected shapes [1036-ray-marching-on-3d-gaussians.md]. Unlike rasterization, it performs adaptive sampling along rays to resolve the exact order and contribution of overlapping Gaussians, ensuring precision governed by sample count rather than rasterization artifacts [1036-ray-marching-on-3d-gaussians.md]. Choose when exact volumetric integration of Gaussian primitives is required.

### [Ray Marching with Neural SDF](2409-ray-marching-with-neural-sdf.md)
Approximates the Signed Distance Field using a deep neural network and renders images by marching rays along the estimated distance field [2409-ray-marching-with-neural-sdf.md]. It differs from traditional SDF rendering by replacing the analytical distance function with a learned implicit representation, allowing for the reconstruction of complex geometries from data rather than explicit modeling [2409-ray-marching-with-neural-sdf.md]. Choose when data-driven surface reconstruction from a neural network is preferred over analytical primitives.

### [RBF Particle Volume Path Tracing](9641-rbf-particle-volume-path-tracing.md)
Renders volumetric data represented as particles by modeling the scene as a continuous scalar field composed of additive weighted radial basis functions (RBFs), specifically Gaussians [9641-rbf-particle-volume-path-tracing.md]. It solves the visual clutter and pathological performance issues associated with BVH-based ray tracing of overlapping splatted particles by treating the volume as a single unified field rather than discrete geometric proxies [9641-rbf-particle-volume-path-tracing.md]. Unlike splatting, it allows for adjustable smoothing via kernel parameters and supports full path tracing through Delta Tracking [9641-rbf-particle-volume-path-tracing.md]. Choose when unbiased path tracing of particle-based volumes is required.

### [MetaNLR++](15259-metanlr.md)
Combines a neural implicit shape representation (SDF) with 2D CNN-based image feature extraction, aggregation, and re-projection for novel view synthesis [15259-metanlr.md]. It solves slow training and rendering times of previous neural lumigraph methods by leveraging meta-learning to specialize initializations, enabling high-quality convergence in minutes and real-time rendering [15259-metanlr.md]. Unlike volumetric methods, it renders explicit surfaces via sphere tracing, and uses deep feature blending to inpaint geometry errors and avoid artifacts from imperfect shape reconstruction [15259-metanlr.md]. Choose when fast, high-quality novel view synthesis with explicit surface rendering is needed.

### NeuMesh Rendering Pipeline
A neural mesh-based implicit field method that disentangles geometry and texture codes for editing [S1].
* Step: Pre-compute near-far bounds for each ray by counting the minimum and maximum distances of ray-to-mesh intersections. Sample 64 coarse points along the ray. Adopt a progressive up-sampling strategy to guide the sampling of 64 fine points, yielding 128 samples for each ray [S1].
* Formula: Distillation loss: Supervised with output from the teacher model (NeuS) for queried color and SDF value. Formula referenced as Eq.(4) in Sec 3.2. Background supervision: Binary cross-entropy loss using foreground masks for unbounded backgrounds [S1].
* Parameter: Batch size: 512 rays. Optimizer: Adam with initial learning rate 0.0005 and cosine annealing scheduler with 5000 warm-up steps. Hardware: Single Nvidia RTX3090-24G GPU. Training time: Approximately 16 hours for each model [S1].
* Layout: Model Architecture: Geometry Code $l_g(x)$ (dim 32), Texture Code $l_t(x)$ (dim 32), Sign Distance $h(x)$ (dim 1). Interpolated Items include $l_t(x)$ (dim 32), gradient of SDF $|\nabla_x s|$ (dim 256), $s$ (dim 256), $\gamma$ (dim 256), $h(x)$ (dim 1), $d$ (dim 160), $\gamma$ (dim 32). Geometry Decoder outputs (256, 256, 256). Radiance Decoder outputs (3) [S1].

### [Implicit Function Rendering Pipeline](15197-implicit-function-rendering-pipeline.md)
A rendering architecture that combines an implicit function (scene representation), a sampling scheme (raymarching), and a renderer to produce images from ray bundles [15197-implicit-function-rendering-pipeline.md]. Choose when a modular pipeline separating implicit representation, sampling, and rendering is desired.

## Background
The concept of implicit surfaces dates back to constructive solid geometry, but neural implicit representations emerged with the advent of deep learning [G]. NeuMesh builds on distillation from the NeuS teacher model to optimize disentangled geometry and texture codes [S1]. MetaNLR++ leverages meta-learning with the Reptile algorithm to accelerate convergence on new scenes [15259-metanlr.md]. RBF-based methods integrate radial basis functions into unbiased path tracing frameworks for particle volumes [9641-rbf-particle-volume-path-tracing.md].

## Sources
[S1] NeuMesh: Learning Disentangled Neural Mesh-Based Implicit Field for Geometry and Texture Editing, https://doi.org/10.1007/978-3-031-19787-1_34, fetched 2026-10-26
