# Surface and Mesh Extraction

Also called: Gaussian Mesh Extraction, 3DGS Meshing

## Overview
This family provides methods to extract explicit triangular meshes or semantic segmentations from trained 3D Gaussian Splatting (3DGS) models, addressing the lack of native surface topology in the implicit volumetric representation [3692-gaussian-opacity-fields-gof-mesh-extraction.md], [7411-distance-based-gaussian-splatting.md]. Unlike standard 3DGS which renders via alpha-blended splats, these methods convert the Gaussian primitives into explicit geometric structures: either by accumulating density into a voxel grid for isosurface extraction, by aligning splats to a pre-existing mesh, or by reconstructing surfaces from rendered depth/normal cues [3692-gaussian-opacity-fields-gof-mesh-extraction.md], [7411-distance-based-gaussian-splatting.md], [12790-mesh-extraction-via-poisson-reconstruction.md].

## Shared principles
All methods operate on the output of a trained 3DGS model, retaining the core primitive representation: a set of $M$ anisotropic 3D Gaussian kernels $\mathcal{G} = \{G_i\}_{i=1}^M$ [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md), [1875-volumetric-gaussian-splatting-avis.md]. Each kernel is defined by:
*   $\mathbf{p}_i \in \mathbb{R}^3$: Position (mean) [S1].
*   $\mathbf{R}_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix [S1].
*   $\mathbf{S}_i \in \mathbb{R}^{3 \times 3}$: Scale matrix [S1].
*   The covariance matrix is $\mathbf{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^\top \mathbf{R}_i^\top$ [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md).

The rendering pipeline's geometric projection remains consistent for ray-based methods: 3D Gaussians are transformed into ray space, and the projected 2D covariance $\mathbf{\hat{\Sigma}}_i$ is computed using the Jacobian $J$ of the affine approximation of the projection and the rotation part $W$ of the extrinsic matrix:
$$ \mathbf{\hat{\Sigma}}_i = J W \mathbf{\Sigma}_i W^T J^T $$
[Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md)

However, the attribute representation and loss functions diverge significantly from the standard RGB opacity/color model. Standard 3DGS uses view-dependent color $\mathbf{c}_i(\mathbf{d})$ [G]. In radiative and domain-specific variants, this is replaced by view-independent scalar attributes or feature vectors [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md), [1875-volumetric-gaussian-splatting-avis.md].

## Subtopics

### [Gaussian Opacity Fields (GOF) Mesh Extraction](3692-gaussian-opacity-fields-gof-mesh-extraction.md)
Extracts a mesh by accumulating Gaussian opacity into a voxel grid and running marching cubes. Chosen when a training-free, explicit surface from the opacity field is required.

### [Distance-based Gaussian Splatting](7411-distance-based-gaussian-splatting.md)
Aligns Gaussians to a pre-existing mesh, classifying them as tightly-bound (surface-aligned) or loosely-bound (artifacts/view-dependent). Chosen when a known mesh topology exists and redundant splats need pruning.

### [2D-Lift-3D Scene Segmentation](11513-2d-lift-3d-scene-segmentation.md)
Segments Gaussians into semantic parts using multi-view 2D masks from SAM. Chosen for scene editing and semantic decomposition rather than surface geometry.

### [Mesh Extraction via Poisson Reconstruction](12790-mesh-extraction-via-poisson-reconstruction.md)
Extracts a mesh by rendering depth/normal maps from Gaussians, back-projecting to a point cloud, and running Poisson reconstruction. Chosen for high-quality surface reconstruction from multi-view consistency without voxelization.

## Background
3D Gaussian Splatting was introduced as a real-time explicit scene representation [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md). However, its volumetric nature lacks explicit mesh topology, limiting applications requiring CAD-like geometry or semantic segmentation. Subsequent works adapted 3DGS for surface extraction by leveraging opacity fields [3692-gaussian-opacity-fields-gof-mesh-extraction.md], distance metrics to existing meshes [7411-distance-based-gaussian-splatting.md], or rendered geometric cues [12790-mesh-extraction-via-poisson-reconstruction.md].

## Sources
