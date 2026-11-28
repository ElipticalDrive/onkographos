# Geometry and Surface Priors

Also called: Geometric constraints, Surface alignment, Depth priors

## Overview
This family of methods addresses the fundamental limitation of vanilla 3D Gaussian Splatting (3DGS): the production of "floating" volumetric ellipsoids that lack an explicit, watertight geometric boundary [G]. By aligning the discrete Gaussian representation with continuous geometric priors (such as SDFs, depth maps, or explicit meshes), these methods enforce positional and normal consistency to enable accurate mesh extraction, physically-based rendering, and robust initialization [G]. Unlike sibling methods that may rely solely on photometric loss or implicit constraints, this family explicitly integrates geometric supervision into the training or post-processing pipeline to ensure the Gaussians conform to physical surfaces [G].

## Shared principles
All methods in this family share the goal of aligning the discrete Gaussian representation with a continuous geometric prior [G]. The core mechanism involves adding geometric loss terms to the standard photometric rendering loss $\mathcal{L}_{rgb}$ [G]. These losses typically enforce two properties:
1. **Positional Consistency:** Gaussian centers $\mathbf{c}_i$ are encouraged to lie on the zero-level set of a distance function (SDF or UDF) or align with metric depth estimates [G].
2. **Normal Consistency:** The orientation of the Gaussians (derived from their covariance matrix or rotation quaternion) is aligned with the surface normal estimated from the geometric prior [G].

The total loss function generally takes the form:
$$ \mathcal{L}_{total} = \mathcal{L}_{rgb} + \lambda_{geo} \mathcal{L}_{geo} + \lambda_{norm} \mathcal{L}_{norm} $$
where $\mathcal{L}_{geo}$ is the geometric consistency loss and $\mathcal{L}_{norm}$ is the normal consistency loss [S1]. The weights $\lambda_{geo}$ and $\lambda_{norm}$ are hyperparameters that balance photometric fidelity against geometric accuracy [G].

## Subtopics

### [NeuSG](9172-neusg.md)
Integrates the NeuS implicit surface reconstruction method into 3DGS to transfer explicit surface properties to discrete Gaussian primitives. Sets itself apart by directly coupling a NeuS-style Signed Distance Function (SDF) with a Gaussian rasterizer, unlike siblings that use monocular depth or explicit mesh extraction. Choose when watertight surface extraction and explicit SDF guidance are required over photometric-only training [G].

### [Monocular Depth Scale Alignment](9183-monocular-depth-scale-alignment.md)
A post-optimization step that resolves the up-to-scale ambiguity in multi-view reconstruction by aligning optimized 3D Gaussians with metric-scale monocular depth estimates. Sets itself apart by operating after standard optimization, fixing Gaussian parameters and optimizing only a global scale factor, unlike siblings that embed depth priors during training [G]. Choose when metric scale recovery is needed without retraining [G].

### [Sparse-view 3D Gaussian Splatting with Depth Priors](9185-sparse-view-3d-gaussian-splatting-with-depth-priors.md)
Addresses 3D reconstruction from sparse-view inputs where standard 3DGS suffers from geometric ambiguity due to insufficient multi-view constraints [G]. Sets itself apart by explicitly integrating depth priors from pre-trained 2D monocular depth estimation networks to supervise Gaussian distribution, rather than relying solely on implicit neural fields. Choose when multi-view cues are insufficient and external depth estimates are available [G].

### [GSDF](9205-gsdf.md)
Aligns the discrete 3D Gaussian representation with a continuous SDF prior to enhance geometry reconstruction quality. Sets itself apart by explicitly transferring surface properties from a neural SDF to the Gaussians, encouraging centers on the zero-level set and normals to align with the SDF gradient, unlike siblings like NeuSG. Choose when explicit SDF alignment is preferred over implicit NeuS coupling [G].

### [GaussianPro](9167-gaussianpro.md)
Mitigates reconstruction artifacts and improves geometric fidelity by introducing a progressive propagation strategy. Sets itself apart by explicitly enforcing normal consistency between neighboring views and introducing planar constraints, ensuring discrete Gaussians align with continuous surface geometry. Choose when "popping" artifacts and planar surface alignment are critical [G].

### 3DGSR
Encourages consistency between SDF and 3DGS to enhance geometry reconstruction quality [S1]. Sets itself apart by directly optimizing SDF-Gaussian alignment without auxiliary networks [G]. Choose when lightweight SDF consistency is needed [G].

### Gaussian Splatting Surface Refinement
Uses pretrained volume rendering priors (UDF) to push Gaussians toward true surfaces by optimizing along increasing rendering weights, avoiding off-surface drift and discreteness [S2]. Sets itself apart as a post-training refinement strategy effective for open and thin structures, outperforming closed-surface-tailored methods [S2]. Choose when refining existing reconstructions for thin/open structures without retraining [G].

## Background
The problem of "floating" Gaussians in standard 3DGS has been a primary driver for geometric priors [G]. Early works like NeuSG adapted implicit surface representations (NeuS) to guide Gaussian placement [S1]. Subsequent works such as GSDF and 3DGSR encouraged consistency between SDFs and Gaussian attributes [S1]. More recent approaches leverage monocular depth estimators for scale alignment or sparse-view reconstruction [S1]. Volume rendering priors, such as those in VRP-UDF, have been used to refine surfaces, particularly for thin structures where standard methods fail [S2].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] VRP-UDF: Toward Unbiased Learning of Unsigned Distance Functions From Multi-View Images With Volume Rendering Priors., https://doi.org/10.1109/tpami.2026.3653901, fetched 2026-10-27
