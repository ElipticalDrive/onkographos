# Explicit Voxel/Grid Representations


## Overview
Explicit Voxel/Grid Representations optimize scene parameters (density, color, features) stored in discrete 3D grids or hierarchical structures, avoiding the implicit parameterization of multi-layer perceptrons (MLPs) used in [Neural Radiance Fields](4972-neural-radiance-fields.md) [G]. This family distinguishes itself by enabling direct optimization of grid values, which significantly reduces training time and memory overhead compared to implicit methods, while offering more structured scene representations than point-based methods like [3D Gaussian Splatting](1795-3d-gaussian-splatting.md) [G].

## Shared principles
Builds on [Radiance-field optimization](44-radiance-field-optimization.md). All methods in this family represent the scene using discrete 3D voxel grids or hierarchical grid structures to store parameters such as volume density, color, and features [G]. These parameters are optimized directly from input images, bypassing the need for training large implicit neural networks [G]. The forward rendering model evaluates the volume rendering equation along camera rays, integrating density and color samples to produce pixel colors [G]. Gradients are propagated back through the differentiable rendering operator to update the voxel grid values [G].

## Subtopics

### [Explicit Radiance Fields](806-explicit-radiance-fields.md)
Explicit Radiance Fields (specifically the PERF method) reconstruct 3D scenes by formulating volumetric reconstruction as a non-linear least-squares problem solved explicitly using a Gauss-Newton solver, avoiding neural networks. It represents the scene as a discrete voxel grid for the foreground and a hierarchy of environment maps for the background. Unlike implicit radiance fields that use MLPs and Adam optimization, this method achieves significantly faster convergence (2-3 orders of magnitude) by leveraging analytical derivatives and iterative linear solvers, while maintaining quality comparable to state-of-the-art neural methods.

### [Direct Voxel Grid Optimization](4694-direct-voxel-grid-optimization.md)
Direct Voxel Grid Optimization (DVGO) reconstructs per-scene radiance fields from calibrated images by directly optimizing a dense 3D voxel grid for volume density and a feature voxel grid for color, achieving convergence in less than 15 minutes on a single GPU. It sets itself apart from implicit methods like [Neural Radiance Fields](4972-neural-radiance-fields.md) by eliminating the need for lengthy MLP training, and from earlier explicit voxel methods by training the voxel grid directly from scratch without requiring a conversion step from a pre-trained implicit model. Unlike [Plenoxels](806-explicit-radiance-fields.md) which optimizes colors directly in the grid, DVGO uses a shallow MLP to handle view-dependent appearance, balancing speed and quality.

### [Canonical Radiance Field with Voxel Grid](8920-canonical-radiance-field-with-voxel-grid.md)
This method replaces the continuous, coordinate-based Multi-Layer Perceptron (MLP) of NeRF with a discrete, learnable voxel grid to represent a static scene in a canonical time frame. It solves the problem of querying infinite 3D points in canonical space for forward warping, which is impractical with heavy MLPs. It differs from [Explicit Radiance Fields](806-explicit-radiance-fields.md) by explicitly separating density and color features into distinct grids optimized for canonical static representation, and from [Direct Voxel Grid Optimization](4694-direct-voxel-grid-optimization.md) by serving as a foundational static representation for dynamic scene warping rather than direct view synthesis optimization.

### [Forward Flow for Novel View Synthesis of Dynamic Scenes](13903-forward-flow-for-novel-view-synthesis-of-dynamic-scenes.md)
Forward Flow is a dynamic NeRF method that replaces backward flow deformation with forward flow to warp a canonical radiance field to target time steps, ensuring smooth and continuous motion modeling. Unlike backward flow methods that map points from current time to canonical space (causing discontinuity for fixed 3D positions), forward flow maps canonical space to current time, yielding smooth flow norms for the same 3D position along the timeline.

## Background
Explicit voxel representations emerged as a response to the computational bottlenecks of implicit neural radiance fields [G]. Early works like [Explicit Radiance Fields](806-explicit-radiance-fields.md) demonstrated that direct optimization of voxel grids could achieve fast convergence for static scenes. [Direct Voxel Grid Optimization](4694-direct-voxel-grid-optimization.md) later popularized the approach by showing that a dense voxel grid combined with a shallow MLP could match the quality of NeRF while training orders of magnitude faster. These methods are widely used in real-time rendering, dynamic scene reconstruction, and scenarios requiring rapid per-scene optimization [G].

## Sources
