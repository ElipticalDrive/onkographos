# Training Optimization and Initialization
Also called: 3DGS Training and Initialization

## Overview
This family covers the initialization strategies and iterative optimization procedures required to reconstruct 3D scenes from 2D images using 3D Gaussian Splatting. It solves the problem of converting sparse or dense image collections into a set of differentiable volumetric primitives that can be rendered in real time. It differs from implicit neural field methods by optimizing explicit, static or dynamic ellipsoids via differentiable rasterization, achieving faster convergence and rendering speeds [3761-3d-gaussian-splatting-training.md].

## Shared principles
All methods in this family optimize a set of $N$ 3D Gaussian primitives. Each primitive $i$ is defined by:
*   Position $\mathbf{p}_i \in \mathbb{R}^3$
*   Rotation $q_i \in \mathbb{R}^4$ (unit quaternion)
*   Scale $\mathbf{s}_i \in \mathbb{R}^3$
*   Opacity $\alpha_i \in [0, 1]$
*   Appearance parameters (e.g., spherical harmonic coefficients $h_i$)

The optimization minimizes a loss function $\mathcal{L}$ between rendered images and ground truth views, typically using the Adam optimizer [3761-3d-gaussian-splatting-training.md]. The rendering process involves projecting 3D Gaussians to 2D screen space, sorting by depth, and alpha blending [3761-3d-gaussian-splatting-training.md].

## Subtopics

### [3DGS Initialization Policy](1398-3dgs-initialization-policy.md)
A deterministic strategy for placing $N$ Gaussians within a bounding volume. It sets apart from heuristic methods by using a fixed power-law scaling for initial opacity to ensure consistent integrated opacity regardless of $N$. Choose this for reproducible baselines in synthetic scenes.

### [3D Gaussian Splatting Training](3761-3d-gaussian-splatting-training.md)
The core iterative optimization loop for static scenes. It initializes Gaussians from Structure-from-Motion (SfM) points and uses adaptive density control (splitting/cloning/pruning) to refine geometry. Choose this for high-fidelity reconstruction of static scenes from image collections.

### [GauFRe](7994-gaufre.md)
A method for dynamic scenes that decouples static and dynamic components. It initializes points into static and dynamic subsets and optimizes them independently, using compact MLPs for deformation. Choose this when the scene contains significant static background and moving foreground elements.

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [3761-3d-gaussian-splatting-training.md]. The initialization policy using power-law opacity scaling was detailed in [1398-3dgs-initialization-policy.md]. GauFRe was proposed to address optimization instability in dynamic scenes by separating static and dynamic components [7994-gaufre.md].

## Sources
[S1] 3DGS Initialization Policy notes
[S2] 3D Gaussian Splatting Training notes
[S6] 3D Gaussian Splatting Training notes (unofficial)
