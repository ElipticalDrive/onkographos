# Neural and Learning-based Extraction

Also called: Differentiable Isosurface Extraction, Neural Meshing

## Overview
Neural and Learning-based Extraction methods generate explicit triangle meshes from implicit representations (such as Signed Distance Fields or occupancy fields) by integrating the extraction process into a differentiable computational graph. Unlike static algorithms like Marching Cubes, these methods allow the mesh geometry to be optimized via gradient descent to fit target scalar values or visual objectives [S2, unofficial]. They set themselves apart from sibling methods by treating grid deformations and topological weights as learnable parameters, enabling the resolution of topological ambiguities and the production of high-fidelity surfaces directly from neural network outputs [977-mise-meshing-isosurfaces-by-extracting.md][3669-flexicubes.md].

## Shared principles
All methods in this family operate on a discretized domain, typically a regular grid (Cartesian, octree, or tetrahedral) where each cell contains scalar values or occupancy states at its vertices or centers. The core mechanism involves traversing the grid cells and evaluating the data to determine which parts of the domain constitute the surface [G].

1.  **Domain Traversal**: The algorithm iterates over the volume data structure [G]. For regular grids, this often involves checking each cell or voxel [G].
2.  **Surface Determination**: A criterion is applied to identify surface locations. In isosurface methods, this involves comparing scalar values against a threshold (isovalue) to detect sign changes or equality [30-isosurface-extraction.md]. In blocky meshing, this involves checking occupancy states or material IDs to identify exposed boundaries [31-blocky-meshing.md].
3.  **Primitive Generation**: Geometric primitives (vertices, edges, faces) are generated based on the determined surface locations [G].
4.  **Mesh Coherence**: Adjacent primitives are merged or connected to form a coherent mesh, often requiring vertex merging to avoid duplicates and ensure watertightness [30-isosurface-extraction.md].

## Subtopics

### [MISE (Meshing Isosurfaces by Extracting)](977-mise-meshing-isosurfaces-by-extracting.md)
MISE generates a 3D mesh from a trained Differentiable Volumetric Rendering (DVR) model by querying an implicit occupancy field using an adaptive resolution algorithm. It sets itself apart from uniform grid-based extraction methods by iteratively refining the sampling grid only where the occupancy field is ambiguous, thereby achieving high-fidelity meshes with significantly fewer model evaluations than static high-resolution marching cubes. It is chosen when the implicit function is expensive to evaluate and the surface geometry is sparse or complex.

### [FlexiCubes](3669-flexicubes.md)
FlexiCubes is a differentiable isosurface extraction method that converts a Signed Distance Field (SDF) defined on a voxel grid into a triangle mesh, specifically designed for gradient-based optimization of geometric, visual, or physical objectives. Unlike static extraction methods, it introduces learnable per-cube weights (beta, alpha, gamma) and vertex deformations to resolve topological ambiguities and improve mesh quality during training. It builds on [Neural and Learning-based Extraction](6371-neural-and-learning-based-extraction.md) by providing a fully differentiable pipeline where the mesh geometry itself becomes an optimization variable. It is chosen when optimizing a mesh to fit a target SDF or when training a neural network to output geometry directly.

### [Ground Truth Generation for Neural Marching Cubes](3820-ground-truth-generation-for-neural-marching-cubes.md)
This method generates multi-resolution ground truth data for training Neural Marching Cubes (NMC) by computing per-grid-cell labels (case IDs, vertex positions, and plane parameters) from high-resolution voxel and surface intersection data. It solves the problem of creating accurate training targets for a Convolutional Neural Network (CNN) that predicts Marching Cubes configurations, enabling the network to learn the mapping from input fields to mesh primitives. It differs from gradient-based methods by explicitly using computed surface intersection points and their normals for least-squares fitting to determine plane parameters, rather than relying solely on voxel occupancy or field gradients. It is chosen when training a CNN to replace the extraction step for inference speed.

### [NeRF Density Field Mesh Extraction](4104-nerf-density-field-mesh-extraction.md)
This method extracts a triangle mesh from a trained Neural Radiance Field (NeRF) by querying the learned density field $\sigma$ on a 3D grid and applying the marching cubes algorithm at a specific threshold. It solves the problem of converting a volumetric, implicit representation into an explicit, watertight geometric mesh suitable for standard rendering pipelines or downstream geometry processing. It differs from its siblings, MISE and FlexiCubes, by relying on a fixed-resolution regular grid and a static isosurface threshold rather than adaptive refinement or learned cube templates [977-mise-meshing-isosurfaces-by-extracting.md][3669-flexicubes.md]. It is chosen for standard NeRF-to-mesh conversion where differentiability is not required.

### [Differentiable Marching Cubes](9774-differentiable-marching-cubes.md)
Differentiable Marching Cubes (DiffMC) is a mesh extraction method that generates explicit, watertight manifold meshes from scalar fields (typically Signed Distance Fields) by optimizing learnable grid deformations and per-cube weights. It builds on [Neural and Learning-based Extraction](6371-neural-and-learning-based-extraction.md) by introducing a deformable grid and differentiable FlexiCubes operator, allowing the mesh geometry to be optimized via gradient descent to fit target scalar values. Unlike static extraction methods, DiffMC dynamically adjusts surface representation through gradient-based optimization, producing significantly smoother surfaces than tet-based alternatives like DMTet when extracting from density fields. It is chosen when high-quality SDF-to-mesh conversion is needed within a differentiable optimization loop.

## Background
The family originates from the need to bridge the gap between implicit neural representations (like NeRFs and SDFs) and explicit polygonal meshes for downstream applications. Early work focused on static extraction from NeRF density fields [4104-nerf-density-field-mesh-extraction.md]. The introduction of differentiable extraction, such as MISE [977-mise-meshing-isosurfaces-by-extracting.md], allowed for adaptive sampling. Subsequent developments like FlexiCubes [3669-flexicubes.md] and Differentiable Marching Cubes [9774-differentiable-marching-cubes.md] enabled end-to-end training of geometry by making the extraction process differentiable with respect to grid parameters. Ground truth generation methods [3820-ground-truth-generation-for-neural-marching-cubes.md] were developed to train CNNs for fast inference, replacing the differentiable optimization step with a learned approximation.

## Sources
[S1] MISE/FlexiCubes/NeRF Extraction source code and documentation, URL, date fetched
[S2] FlexiCubes/DiffMC technical reports and documentation, URL, date fetched
