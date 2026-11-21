# Geometric Priors and Depth Integration

Also called: Geometric Regularization, Depth-Guided NeRF, SDF-based NeRF

## Overview
Geometric Priors and Depth Integration methods enhance standard Neural Radiance Fields (NeRF) by incorporating explicit geometric constraints—such as depth maps, surface normals, or Structure-from-Motion (SfM) features—into the optimization loop . Unlike standard NeRF, which relies solely on implicit photometric consistency and often suffers from ambiguous geometry in sparse-view or textureless scenarios, these methods enforce hard boundaries, metric scale, or structural consistency to improve reconstruction fidelity . They distinguish themselves from sibling methods like [AdaNeRF](6195-adanerf.md) or [SinNeRF](6199-sinnerf.md) by explicitly modeling the scene's 3D structure (via SDFs, cost volumes, or depth priors) rather than just refining the radiance field's appearance or frequency representation.

## Shared principles
All methods in this family build on [Specific Model Architectures](8585-specific-model-architectures.md) but diverge in how the geometric information is integrated into the optimization loop. The shared mechanism involves augmenting the standard photometric loss with geometric terms or modifying the ray sampling process to respect geometric boundaries.

**Geometric Loss Functions:**
Many methods introduce explicit loss terms that penalize deviations from geometric priors. Common forms include:
1.  **Depth Consistency Loss:** Minimizes the difference between the rendered depth (computed as the expected depth along the ray) and a prior depth map $D_{prior}$:
    $$ L_{depth} = \| D_{rendered} - D_{prior} \|_1 $$
2.  **Normal Smoothness:** Encourages surface normals derived from the density field to be smooth or aligned with estimated normals .
3.  **Epipolar Constraints:** Enforces that 3D points reconstructed from matched features in different views lie on the corresponding epipolar lines .

**Depth-Guided Sampling:**
Instead of uniform or hierarchical sampling along the entire ray, these methods use depth priors to concentrate samples near the estimated surface. This reduces computational cost and mitigates the "transparent" artifacts common in standard NeRF when geometry is ambiguous. For example, samples are clustered around the depth value predicted by a monocular depth estimator .

**Signed Distance Fields (SDF):**
Some variants replace the volumetric density $\sigma$ with a Signed Distance Field (SDF). The rendering integral is modified to integrate weights based on the SDF value relative to a truncation distance $tr$, effectively creating a thin shell around the surface rather than a volumetric blob. The weight for a sample at distance $d$ is often computed as:
$$ w = \sigma\left(\frac{d}{tr}\right) \cdot \sigma\left(-\frac{d}{tr}\right) $$
This allows for direct mesh extraction via Marching Cubes .

## Subtopics

### [SfMNeRF](8664-sfmnerf.md)
Integrates explicit SfM constraints (matched features, epipolar geometry) into the NeRF loss to improve geometric accuracy under sparse inputs. It uses sub-pixel rendering and a pyramidal optimization strategy to avoid local minima.

### [DASNeRF](8668-dasnerf.md)
Uses monocular depth priors (DepthAnythingV2) to guide learning via Depth Ordering Fidelity Loss and Depth Structural Consistency Regularization. It employs dynamic optimal sampling to handle varying depth levels.

### [GeoNeRF](8710-geonerf.md)
Addresses sparse-view ambiguity by modeling geometry through a cascaded 3D cost volume and Transformer-based attention. It extracts view-independent and view-dependent tokens to predict density and color, enabling robust generalization.

### [Neural RGB-D Surface Reconstruction](11352-neural-rgb-d-surface-reconstruction.md)
Replaces density with a Truncated Signed Distance Field (TSDF) for geometry and a radiance field for appearance. It enforces hard boundaries, allowing direct mesh extraction and solving the issue of semi-transparent artifacts.

### [DINER](11354-diner.md)
Solves high-disparity synthesis artifacts by using dense depth priors to guide feature fusion and sampling. It leverages predicted depth maps to align features from sparse input views, improving correspondence search.

### [Volume Rendering Prior (VRPrior)](14662-volume-rendering-prior-vrprior.md)
Replaces handcrafted UDF-to-density mappings with a learnable neural network $f_w$ that considers 3D context from neighboring points. It learns a robust mapping from local geometric structures to rendering weights, reducing bias and outlier sensitivity.

### [Geometric Structure Constraint](16347-geometric-structure-constraint.md)
Integrates explicit geometric priors (normal smoothness, bilateral edge-aware constraints) directly into the optimization loop. It unifies physical imaging and geometric constraints for accurate 3D reconstruction under complex imaging conditions.

## Background
The integration of geometric priors into NeRF-based reconstruction emerged to address the inherent ambiguity of implicit volumetric representations, particularly in sparse-view settings or scenes with low texture. Early NeRF implementations  relied on photometric consistency alone, which often resulted in "floaters" or incorrect geometry in unobserved regions. Subsequent works introduced explicit geometric signals such as SfM features , monocular depth estimators , and SDFs  to regularize the optimization landscape. This family represents the convergence of traditional multi-view stereo (MVS) techniques with differentiable rendering, allowing for metric-scale reconstruction and direct mesh extraction.

## Sources
