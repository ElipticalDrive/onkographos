# Progressive and Multi-stage Training

Also called: Multi-stage Optimization, Coarse-to-Fine Training, Progressive Densification

## Overview
Progressive and Multi-stage Training refers to methodologies that refine 3D Gaussian Splatting representations through distinct phases or iterative stages rather than a single monolithic optimization. It solves the problem of poor reconstruction quality and insufficient geometric detail that arises when optimizing from sparse initializations or when balancing global structure with local high-frequency details [S1, unofficial]. Unlike single-stage training, this approach yields significantly higher rendering quality by balancing global structure capture with local detail refinement [S1, unofficial].

## Shared principles
The following mechanisms are common to all progressive and multi-stage variants:

1.  **Staged Initialization:** The training process begins with a coarse representation (e.g., sparse SfM points, feed-forward predictions, or downsampled meshes) rather than a fully dense set of Gaussians [S1, unofficial]. This initial state provides global structure but lacks fine detail.
2.  **Iterative Densification:** New primitives are introduced periodically or in specific stages to increase density in regions with high reconstruction error or high gradient magnitude. This is distinct from static initialization methods [S1, unofficial].
3.  **Progressive Refinement:** The optimization landscape is managed by gradually increasing the complexity of the representation. Early stages focus on global geometry and color, while later stages focus on fine details and texture [S1, unofficial].
4.  **Pruning and Management:** As density increases, redundant or low-opacity primitives are pruned to maintain efficiency and prevent overfitting [S1, unofficial]. This is often coupled with densification to maintain a target density or memory budget.

## Subtopics

### [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md)
The Hybrid Coarse-to-Fine Strategy employs multiple stages of refinement for mesh-based or point-based representations, addressing insufficient geometric detail in sparse initial point clouds. It differs from standard 3DGS by scoring primitives based on reconstruction error and spatial extent, then inserting new primitives to reduce error in high-impact regions. It is chosen when mesh-based representations or Delaunay triangulations are used as the underlying structure for densification.

### [Training Objective and Densification](7448-training-objective-and-densification.md)
This method defines specific loss functions and densification strategies for **MeshGS** (Adaptive Mesh-Aligned Gaussian Splatting), integrating explicit mesh geometry into the training loop. It solves the problem of Gaussian primitives drifting away from or failing to align with underlying surface geometry. It differs from standard 3DGS by introducing geometry-aware losses (normal consistency, projection, and scale regularization) for "tightly-bound" splats and by initializing new primitives via mesh surface sampling rather than random perturbation.

### [Fine-tuning](9138-fine-tuning.md)
Fine-tuning is the optimization phase that begins after an initial representation is established, aiming to refine geometry and color attributes to achieve photorealistic reconstruction. It solves the problem of initializing a scene representation by allowing the model to converge from a coarse or pretrained state to a high-fidelity solution. It sets itself apart from pure initialization methods by actively optimizing all primitive attributes and managing population density through growing and pruning, and from pure training methods by often leveraging pretrained priors to accelerate convergence and reduce memory consumption.

### Progressive Propagation Strategy
- **Definition:** Introduces a progressive propagation strategy to expand the Gaussian cloud, ensuring smooth expansion and better coverage [S1, unofficial]. It expands the Gaussian cloud in a fronto-parallel manner, ensuring smooth expansion and better coverage [S1, unofficial].

### Interleaved Densification
- **Step:** Alternate between spatial and temporal densification every 2 epochs to avoid Gaussians failing to reconstruct either motion or appearance detail [S2].

## Background
The concept of multi-stage training in 3D Gaussian Splatting emerged as a response to the limitations of single-stage optimization, particularly regarding initialization quality and the trade-off between global structure and local detail. The Hybrid Coarse-to-Fine Strategy and Progressive Propagation Strategy are discussed in the context of improving reconstruction quality in visually salient areas and ensuring smooth expansion [S1, S1, unofficial]. Interleaved densification for spatio-temporal optimization was introduced to handle scenes with oscillatory motion, where static densification fails to capture either motion or appearance detail [S2]. These methods build on the foundational densification strategies of 3DGS [Parent].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
