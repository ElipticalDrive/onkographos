# Dynamic and Deformation Modeling

Also called: Dynamic NeRF, Time-varying NeRF, Deformable NeRF

## Overview
Dynamic and Deformation Modeling extends static Neural Radiance Fields to reconstruct time-varying scenes from multi-view video or sparse image collections, solving the problem of novel view synthesis for non-static content [G]. It sets itself apart from static NeRF variants by explicitly incorporating temporal information, either through direct time-conditioned radiance fields or by warping spatial coordinates to a canonical space, allowing for the coherent reconstruction of moving objects that would otherwise appear inconsistent in a static model [G].

## Shared principles
Builds on [Neural Radiance Fields](4972-neural-radiance-fields.md). All methods in this family rely on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps . The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses) .

## Subtopics

### [Deformation Field Parametrization](12731-deformation-field-parametrization.md)
Deformation Field Parametrization methods map 3D spatial coordinates from a dynamic, time-varying state to a static canonical space (or vice versa) to enable the rendering of dynamic scenes using static neural radiance fields. This approach solves the problem of modeling non-rigid, time-varying geometry without requiring a separate radiance field for every time step. It sets itself apart from sibling methods by the specific mathematical representation of the deformation field: while some use cubic B-splines for sparse data efficiency, others use MLPs guided by parametric models like 3DMMs or inverse Linear Blend Skinning with neural residuals. Choose this when explicit geometric warping is preferred over direct temporal conditioning.

### [Dynamic Scene Representation](13340-dynamic-scene-representation.md)
Dynamic Scene Representation methods extend static radiance fields to model temporal variations, enabling novel view synthesis of non-static scenes. Unlike static methods, these approaches incorporate time or scene variables as inputs to the neural field or deformation operators to predict density and color for specific moments. They differ from sibling methods like Deformation Field Parametrization by explicitly handling the evolution of geometry and appearance over time, often through shared networks, explicit temporal interpolation, or deformation fields. Choose this when direct temporal conditioning or shared network architectures are sufficient for the scene's dynamics.

### [Deformable Neural Radiance Fields (Nerfies)](13428-deformable-neural-radiance-fields-nerfies.md)
Deformable Neural Radiance Fields (Nerfies) reconstructs non-rigidly deforming scenes by decomposing the scene into a static canonical template volume (a standard NeRF) and a per-observation continuous deformation field that maps observation-space coordinates to canonical-space coordinates. Unlike methods that condition deformation on time or define it on discrete meshes/voxels, Nerfies conditions the deformation on a learned per-frame latent code and models the deformation as a continuous function using an MLP. It distinguishes itself from siblings by introducing elastic regularization to resolve ambiguities between motion and scaling, and a coarse-to-fine optimization strategy to avoid local minima. Choose this for high-quality reconstruction of non-rigid human deformation or elastic objects.

### [JointRF Representation](13660-jointrf-representation.md)
JointRF is a compact residual feature grid representation for dynamic Neural Radiance Fields (NeRFs) that handles large motions and long sequences by decomposing the radiance field into a long-reference basis feature grid and a coefficient feature grid. Unlike canonical space methods that warp features back to a static canonical space—making them fragile to large motions and topological changes—JointRF uses residual grids to capture temporal variations, thereby diminishing temporal redundancy and improving compression efficiency. Choose this when dealing with long sequences or large motions where canonical warping fails.

### [D-TensoRF](13781-d-tensorf.md)
D-TensoRF is a dynamic radiance field model that maps 3D location, viewing direction, and time instant to emitted color and volume density using factorized 4D/5D tensor grids. It extends TensoRF by incorporating a time dimension and applying Matrix-Matrix decomposition to reduce memory complexity. Unlike Deformable Neural Radiance Fields (Nerfies), which rely on explicit deformation fields to warp static coordinates, D-TensoRF models temporal dynamics directly within the tensor structure, augmented by smoothing regularization to enforce temporal dependency. Choose this when memory efficiency and direct temporal modeling in tensor grids are prioritized over explicit deformation fields.

### [SE(3) Deformation Field](13996-se-3-deformation-field.md)
The SE(3) Deformation Field encodes scene deformation as a rigid transformation (rotation and translation) per point using a screw axis representation. It replaces per-point translation fields to simplify optimization by requiring a single rotation parameter across space, rather than distinct translation vectors for each point. This method is primarily used in dynamic scene reconstruction to handle rotational motion more efficiently than pure translation-based deformation fields. Choose this when scenes exhibit significant rigid rotational motion and a simpler optimization landscape is desired.

## Background
The extension of NeRF to dynamic scenes originated with the recognition that static radiance fields cannot represent temporal changes [G]. Early approaches focused on direct time-conditioning, where the temporal variable is concatenated with spatial coordinates to predict density and color [G]. Subsequent work introduced deformation fields to handle non-rigid motion, mapping points from observation space to a canonical space to leverage static NeRF priors [G]. These methods are primarily used in novel view synthesis for video, 4D reconstruction, and dynamic scene analysis [G].

## Sources
[S1] Radiance Field Optimization, URL, date fetched
