# Dynamic Scene Representation
Also called: Dynamic Radiance Fields
## Overview
Dynamic Scene Representation methods extend static radiance fields to model temporal variations, enabling novel view synthesis of non-static scenes [G]. Unlike static methods, these approaches incorporate time or scene variables as inputs to the neural field or deformation operators to predict density and color for specific moments [7274-dynerf.md][12734-neural-rendering-of-variable-scenes.md]. They differ from sibling methods like [Deformation Field Parametrization](12731-deformation-field-parametrization.md) by explicitly handling the evolution of geometry and appearance over time, often through shared networks, explicit temporal interpolation, or deformation fields [G].

## Shared principles
This family builds on [Neural Radiance Fields](4972-neural-radiance-fields.md). All methods in this family rely on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps [44-radiance-field-optimization.md]. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses) [44-radiance-field-optimization.md].

## Subtopics

### [DyNeRF](7274-dynerf.md)
DyNeRF represents volumetric videos using a single shared Multi-Layer Perceptron (MLP) that takes spatial coordinates and a temporal embedding as input to predict the radiance field for a specific video frame. It sets itself apart from [Neural Rendering of Variable Scenes](12734-neural-rendering-of-variable-scenes.md) by using a monolithic shared network structure rather than decomposing the scene into static and dynamic components or using explicit voxel structures. While simple to implement, this single shared MLP structure tends to lose details in high-frequency regions of the image.

### [Neural Rendering of Variable Scenes](12734-neural-rendering-of-variable-scenes.md)
This method explicitly incorporates scene variation into the training phase by encoding predefined ranges of dynamic scene elements into a structured vector representation. Unlike methods that constrain learning to a single static manifold, these approaches learn a surface radiance field capable of handling all possible permutations of the specified scene variables. This distinguishes them from sibling methods like [DyNeRF](7274-dynerf.md), which typically model deformation or motion through explicit geometric transformations or temporal bases, whereas variable scene methods treat dynamic parameters as direct inputs to the neural field to capture higher-order interactions between moving objects and the static environment.

### [Explicit 4D Gaussian Splatting](14584-explicit-4d-gaussian-splatting.md)
Explicit 4D Gaussian Splatting (Ex4DGS) achieves memory efficiency and rendering speed by separating scene primitives into static and dynamic categories and explicitly interpolating their attributes over time. Unlike implicit deformation fields or continuous trajectory models, Ex4DGS stores sparse keyframe data for dynamic Gaussians and uses specific interpolators to reconstruct continuous motion, handling complex phenomena like self-occlusion and object disappearance without requiring auxiliary masks or supervision. It builds on [Dynamic Scene Representation](13340-dynamic-scene-representation.md) by introducing a fully explicit temporal domain where attributes are sampled at discrete timestamps and interpolated.

### [Gaussian Deformation Field (GauFRe)](15458-gaussian-deformation-field-gaufre.md)
GauFRe augments 3D Gaussian Splatting with a time-conditioned forward-warping deformation field to enable real-time novel view synthesis of dynamic scenes. Unlike NeRF-based methods that use backward warps to query canonical space, GauFRe stores Gaussians in a canonical space and uses a neural network to output deformed attributes (position, rotation, scale) for each time step $t$. It distinguishes itself from sibling methods by explicitly separating the scene into static and deformable Gaussian sets, preventing capacity waste and noise in static regions, and by modeling non-rigid deformations (scaling) in addition to rigid motion.

### [Light-weight 4D Gaussian Splatting (LGS)](15777-light-weight-4d-gaussian-splatting-lgs.md)
Light-weight 4D Gaussian Splatting (LGS) is a memory-efficient method for reconstructing and rendering dynamic scenes, specifically designed for surgical endoscopy. It addresses the high memory consumption and computational cost of standard 4D Gaussian Splatting by combining three compression techniques: Deformation-Aware Pruning (DAP), Gaussian-Attribute Pruning (GAP), and Feature Field Condensation (FFC). Unlike [Explicit 4D Gaussian Splatting](14584-explicit-4d-gaussian-splatting.md) which stores full attributes, or [Gaussian Deformation Field (GauFRe)](15458-gaussian-deformation-field-gaufre.md) which uses a deformation network, LGS achieves compression by pruning redundant Gaussians and condensing the feature field resolution, maintaining real-time rendering speeds with significantly smaller model sizes.

### Generalizable Dynamic Neural Radiance Field
A neural radiance field that takes transformed 3D location, view direction, and aggregated features as input to predict volume density and color [S2].

## Background
The concept of representing dynamic scenes using neural radiance fields emerged with the need to extend static NeRF capabilities to video and dynamic content [G]. The integration of the temporal dimension in a radiance field-based volumetric representation offers two benefits: it increases temporal homogeneity, reinforcing spatial information with temporal redundancy, and addresses temporary occluded areas [S1]. Deformation-based approaches achieve this by dividing the dynamic radiance field into a spatial radiance field and a dynamic deformation field [S1].

## Sources
[S1] ADVANCEMENTS IN RADIANCE FIELD TECHNIQUES FOR VOLUMETRIC VIDEO GENERATION: A TECHNICAL OVERVIEW, https://hal.science/hal-05049100v1, fetched 2026-10-26
[S2] HumanNeRF: Efficiently Generated Human Radiance Field from Sparse Inputs, https://doi.org/10.1109/cvpr52688.2022.00759, fetched 2026-10-26
