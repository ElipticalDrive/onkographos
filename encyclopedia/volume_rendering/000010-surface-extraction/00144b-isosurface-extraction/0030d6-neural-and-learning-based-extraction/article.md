# Neural and Learning-based Extraction

Also called: Differentiable Meshing, Neural Isosurface Extraction

## Overview
Neural and Learning-based Extraction methods generate 3D meshes from implicit representations learned by neural networks, such as Signed Distance Fields (SDFs) or occupancy fields, by making the extraction process differentiable or adaptive. Unlike static isosurface methods like Marching Cubes, which operate on fixed grids, these methods integrate mesh generation into the training pipeline or use learned parameters to resolve topological ambiguities, enabling end-to-end optimization of geometry from visual or physical objectives [977-mise-meshing-isosurfaces-by-extracting.md][3669-flexicubes.md].

## Shared principles
All methods in this family operate on a discretized domain, typically a regular grid (Cartesian, octree, or tetrahedral) where each cell contains scalar values or occupancy states at its vertices or centers. The core mechanism involves traversing the grid cells and evaluating the data to determine which parts of the domain constitute the surface [G].

1.  **Domain Traversal**: The algorithm iterates over the volume data structure [G]. For regular grids, this often involves checking each cell or voxel [G].
2.  **Surface Determination**: A criterion is applied to identify surface locations. In isosurface methods, this involves comparing scalar values against a threshold (isovalue) to detect sign changes or equality [30-isosurface-extraction.md]. In blocky meshing, this involves checking occupancy states or material IDs to identify exposed boundaries [31-blocky-meshing.md].
3.  **Primitive Generation**: Geometric primitives (vertices, edges, faces) are generated based on the determined surface locations [G].
4.  **Mesh Coherence**: Adjacent primitives are merged or connected to form a coherent mesh, often requiring vertex merging to avoid duplicates and ensure watertightness [30-isosurface-extraction.md].

## Subtopics

### [MISE (Meshing Isosurfaces by Extracting)](977-mise-meshing-isosurfaces-by-extracting.md)
MISE generates a 3D mesh from a trained Differentiable Volumetric Rendering (DVR) model by querying an implicit occupancy field using an adaptive resolution algorithm [977-mise-meshing-isosurfaces-by-extracting.md]. It sets itself apart from uniform grid-based extraction methods by iteratively refining the sampling grid only where the occupancy field is ambiguous, thereby achieving high-fidelity meshes with significantly fewer model evaluations than static high-resolution marching cubes [977-mise-meshing-isosurfaces-by-extracting.md]. Choose this method when computational budget for model inference is limited and the implicit field is smooth but requires high-frequency detail [977-mise-meshing-isosurfaces-by-extracting.md]. [977-mise-meshing-isosurfaces-by-extracting.md]

### [FlexiCubes](3669-flexicubes.md)
FlexiCubes is a differentiable isosurface extraction method that converts a Signed Distance Field (SDF) defined on a voxel grid into a triangle mesh, specifically designed for gradient-based optimization of geometric, visual, or physical objectives [3669-flexicubes.md]. Unlike static extraction methods, it introduces learnable per-cube weights (beta, alpha, gamma) and vertex deformations to resolve topological ambiguities and improve mesh quality during training [3669-flexicubes.md]. It builds on Neural and Learning-based Extraction by providing a fully differentiable pipeline where the mesh geometry itself becomes an optimization variable [3669-flexicubes.md]. Choose this method when optimizing geometry from differentiable rendering or physical simulations [3669-flexicubes.md]. [3669-flexicubes.md]

### [Ground Truth Generation for Neural Marching Cubes](3820-ground-truth-generation-for-neural-marching-cubes.md)
This method generates multi-resolution ground truth data for training Neural Marching Cubes (NMC) by computing per-grid-cell labels (case IDs, vertex positions, and plane parameters) from high-resolution voxel and surface intersection data [3820-ground-truth-generation-for-neural-marching-cubes.md]. It solves the problem of creating accurate training targets for a Convolutional Neural Network (CNN) that predicts Marching Cubes configurations, enabling the network to learn the mapping from input fields to mesh primitives [3820-ground-truth-generation-for-neural-marching-cubes.md]. It differs from gradient-based methods by explicitly using computed surface intersection points and their normals for least-squares fitting to determine plane parameters, rather than relying solely on voxel occupancy or field gradients [3820-ground-truth-generation-for-neural-marching-cubes.md]. Choose this method when training a CNN-based extractor for fast, static inference [3820-ground-truth-generation-for-neural-marching-cubes.md]. [3820-ground-truth-generation-for-neural-marching-cubes.md]

### [NeRF Density Field Mesh Extraction](4104-nerf-density-field-mesh-extraction.md)
This method extracts a triangle mesh from a trained Neural Radiance Field (NeRF) by querying the learned density field $\sigma$ on a 3D grid and applying the marching cubes algorithm at a specific threshold [4104-nerf-density-field-mesh-extraction.md]. It solves the problem of converting a volumetric, implicit representation into an explicit, watertight geometric mesh suitable for standard rendering pipelines or downstream geometry processing. It differs from its siblings, MISE and FlexiCubes, by relying on a fixed-resolution regular grid and a static isosurface threshold rather than adaptive refinement or learned cube templates [4104-nerf-density-field-mesh-extraction.md]. Choose this method for simple, static mesh extraction from pre-trained NeRFs where differentiability is not required [4104-nerf-density-field-mesh-extraction.md]. [4104-nerf-density-field-mesh-extraction.md]

### Differentiable Marching Cubes
**Definition**: A differentiable meshing algorithm that divides 3D space into a deformable grid and takes a scalar field (often SDF) defined on its vertices as input to generate an explicit watertight manifold mesh [S1].
**Steps**: Divide the 3D space into a deformable grid; take a scalar field (often SDF) defined on the grid vertices as input; turn the scalar field into an explicit mesh using a differentiable marching cubes algorithm [S1].
**Parameters**: Grid resolution: The resolution of the deformable grid used for DiffMC. Typical values are 256 for general results, 128 when comparing with nvdiffrec, 375 for LLFF dataset, and 340/56 for foreground/background in Mip-NeRF 360 [S1].
**Pitfalls**: DMTet tends to introduce more artifacts (zigzag lines) when extracting meshes from density fields due to varying space divisions and non-linearity, whereas DiffMC generates smoother surfaces [S1].
**Numbers**: Rendering quality (PSNR) improves with DiffMC resolution, plateauing around 400. At resolution 256, PSNR is 31.19; at 400, PSNR is 31.54 [S1].
**Comparison**: DiffMC produces significantly smoother surfaces and mitigates artifacts from density field non-linearity compared to DMTet, which produces zigzag lines [S1]. [S1]

## Background
The family of neural extraction methods emerged with the rise of Neural Radiance Fields (NeRF) and differentiable rendering, which require explicit geometry for downstream tasks like physics simulation or texturing. Early approaches adapted static algorithms like Marching Cubes to query neural fields [4104-nerf-density-field-mesh-extraction.md]. Subsequent work introduced adaptivity to reduce computational cost, such as MISE, which refines grids based on occupancy uncertainty [977-mise-meshing-isosurfaces-by-extracting.md]. The field then shifted towards fully differentiable extraction to allow joint optimization of geometry and appearance, leading to methods like FlexiCubes and DiffMC, which resolve topological ambiguities through learnable parameters or deformable grids [3669-flexicubes.md][S1].

## Sources
[S1] NeuManifold: Neural Watertight Manifold Reconstruction with Efficient and High-Quality Rendering Support, https://doi.org/10.1109/wacv61041.2025.00081, fetched 2026-10-29
