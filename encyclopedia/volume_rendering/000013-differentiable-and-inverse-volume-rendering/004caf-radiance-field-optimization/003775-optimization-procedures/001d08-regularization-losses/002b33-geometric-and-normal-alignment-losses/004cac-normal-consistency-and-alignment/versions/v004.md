# Normal Consistency and Alignment
## Overview
This family enforces geometric plausibility in differentiable volume rendering by constraining predicted surface normals against ray directions, ground truth data, or density gradients [4202-normal-ray-alignment-loss.md][4232-normal-ground-truth-alignment-loss.md][7713-predicted-normal-loss.md]. It solves the ambiguity of implicit surfaces in radiance fields, particularly in sparse-view or ill-posed reconstruction scenarios, by stabilizing optimization and improving surface quality [4202-normal-ray-alignment-loss.md][7713-predicted-normal-loss.md]. Unlike sibling methods like [Curvature Loss](2911-curvature-loss.md) and [Material Smoothness Loss](7299-material-smoothness-loss.md), these losses specifically target the orientation and consistency of the normal field.

## Shared principles

## Subtopics

### [Normal Ray Alignment Loss](4202-normal-ray-alignment-loss.md)
Normal Ray Alignment Loss is a geometric regularization term used in differentiable volume rendering to enforce consistency between predicted surface normals and camera ray directions [4202-normal-ray-alignment-loss.md]. It encourages normals to align (point away from the camera) or anti-align (point toward the camera) with the viewing ray [4202-normal-ray-alignment-loss.md]. Unlike [Predicted Normal Loss](7713-predicted-normal-loss.md), which typically enforces smoothness or consistency with ground-truth normals, this method specifically targets the geometric relationship between the normal field and the ray direction [4202-normal-ray-alignment-loss.md]. Choose when multi-view consistency is insufficient to resolve front-back ambiguity.

### [Normal Ground Truth Alignment Loss](4232-normal-ground-truth-alignment-loss.md)
The Normal Ground Truth Alignment Loss is a regularization term in differentiable inverse rendering that supervises predicted surface normals against available ground truth normals by minimizing their angular deviation [4232-normal-ground-truth-alignment-loss.md]. It differs from sibling methods like [Normal Ray Alignment Loss](4202-normal-ray-alignment-loss.md) and [Predicted Normal Loss](7713-predicted-normal-loss.md) by directly comparing network outputs to explicit geometric targets rather than enforcing internal consistency or ray alignment [4232-normal-ground-truth-alignment-loss.md]. Choose when ground truth normal annotations are available.

### [Predicted Normal Loss](7713-predicted-normal-loss.md)
Predicted Normal Loss is a regularization term used in differentiable volume rendering and inverse rendering to enforce geometric consistency between surface normals predicted by a neural network and normals rendered from the volume field [7713-predicted-normal-loss.md]. It addresses the problem of geometric inconsistency where explicit normal predictions may diverge from the implicit geometry defined by the density or signed distance function (SDF) field [7713-predicted-normal-loss.md]. It sets itself apart from [Normal Ray Alignment Loss](4202-normal-ray-alignment-loss.md) by comparing two neural predictions (predicted vs. rendered) rather than aligning with ray directions, and from [Normal Ground Truth Alignment Loss](4232-normal-ground-truth-alignment-loss.md) by not requiring external ground-truth normal annotations [7713-predicted-normal-loss.md]. Choose when no ground truth normals are available but implicit-explicit consistency is required.

### [Normal Alignment Regularization](14624-normal-alignment-regularization.md)
Normal Alignment Regularization is a geometric regularization technique used in differentiable volume rendering and 3D Gaussian Splatting to align the learned scene representation with the underlying physical surface [14624-normal-alignment-regularization.md]. It solves the problem of misaligned Gaussians or density fields by penalizing deviations between rendered normals and external geometric constraints, such as planar surfaces, depth-derived normals, or prior normal maps [14624-normal-alignment-regularization.md]. It sets itself apart from sibling methods by focusing on general surface alignment via external constraints rather than specific alignment to ray directions [4202-normal-ray-alignment-loss.md] or ground-truth normal labels [4232-normal-ground-truth-alignment-loss.md]. Choose when external geometric priors are available.

## Background
Originating from the need to stabilize differentiable rendering and 3D Gaussian Splatting optimization, these losses address the "floaters" and ambiguous geometry issues inherent in sparse-view reconstruction [G]. The specific formulations for ray alignment, ground truth alignment, and predicted normal consistency are detailed in the respective child articles.

## Sources
