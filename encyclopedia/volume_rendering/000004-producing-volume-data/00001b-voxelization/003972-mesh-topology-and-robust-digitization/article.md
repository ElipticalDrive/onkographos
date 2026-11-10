# Mesh Topology and Robust Digitization


## Overview
This family of methods converts polygon meshes into discrete volumetric representations (voxel grids) while prioritizing topological validity and robustness against geometric defects. Unlike standard rasterization which may produce gaps or artifacts on non-watertight or low-quality meshes, these techniques enforce structural integrity, ensuring the resulting volume accurately reflects the mesh's interior or surface connectivity [S1, unofficial].

## Shared principles
All methods in this family map a continuous polygon mesh into a discrete `UniformGrid` or `RectilinearGrid` suitable for rendering or simulation [G]. The core mechanism involves:
1.  **Input Validation/Preprocessing**: Handling non-watertight, non-manifold, or low-quality input meshes, often through triangulation, mesh improvement, or tolerance-based insertion [S1, unofficial].
2.  **Discretization**: Mapping the mesh geometry to a 3D grid, either by intersecting primitives with voxel targets, enclosing points within surfaces, or generating intermediate tetrahedral meshes [S1, unofficial].
3.  **Topological Enforcement**: Ensuring the output voxels maintain specific connectivity (e.g., 26-connected) or label consistency across multiple mesh blocks, preventing gaps or mislabeled regions [S1, unofficial].

## Subtopics

### [Relaxed Connectivity Voxelization](2481-relaxed-connectivity-voxelization.md)
Produces a binary `UniformGrid` from effectively 1D primitives (curves, surfaces) where occupied voxels form a 26-connected path. It distinguishes itself by using a target of three bisecting quadrilaterals per voxel to guarantee connectivity without gaps, unlike standard methods that may break thin structures. Choose this for flow simulation or fill operations on thin structures where topological continuity is critical.

### [Polygon Mesh Voxelization via Union of Sets](4605-polygon-mesh-voxelization-via-union-of-sets.md)
Transforms a polygon mesh into a `UniformGrid` using high-level library abstractions (e.g., PyVista). It enforces topological validity through built-in watertightness checks. Distinguished from explicit geometric decomposition methods, it relies on library functions like `pyvista.voxelize`. Choose this for rapid voxelization of valid, watertight meshes where ease of implementation is preferred over low-level control.

### [Labeled Volume Creation via MultiBlock Iteration](7389-labeled-volume-creation-via-multiblock-iteration.md)
Produces a single `RectilinearGrid` from a `MultiBlock` mesh input, labeling each voxel with the integer ID of the specific sub-mesh block it intersects. It differs from binary voxelization by preserving the topological identity of each mesh component through iterative enclosed-point testing or data sampling. Choose this when distinguishing between multiple overlapping or adjacent mesh components in a single volume is required.

### [Improved Tetrahedral Mesh Generation for Voxelization](7476-improved-tetrahedral-mesh-generation-for-voxelization.md)
A robust framework that generates a high-quality tetrahedral mesh within a user-defined error bound to produce solid binary voxel data. It specifically addresses the failure of existing methods on low-quality, non-watertight, or topologically open meshes. Unlike centroid-connection methods, it combines background grid insertion with mesh improvement to maintain quality and speed, achieving near 100% success rates on difficult datasets. Choose this for robust voxelization of internet-sourced 3D models that are often non-watertight or low-quality.

#### Displacement Voxelization
**Definition**: An extension to the base framework that uses PN-triangles and displacement mapping to voxelize low-resolution meshes with rugged surfaces at high resolution, achieving visual quality comparable to conventional voxelization of high-resolution meshes [S1, unofficial].
**Steps**: Perform triangle subdivision using the hardware tessellator as in the base method. Use PN-triangles to reconstruct mesh details. Utilize the positions and normals of the PN-triangles to apply displacement mapping during the voxelization process [S1, unofficial].

## Background
The need for robust mesh voxelization arises from the prevalence of low-quality, non-watertight, or topologically open meshes in internet 3D models, which cause standard voxelization methods to fail [S1, unofficial]. Traditional approaches often connect every triangle facet to the model centroid, producing faulty meshes for high-face models, or suffer from boundary quality issues and high costs for sparse models [S1, unofficial]. The development of improved tetrahedral mesh generation and relaxed connectivity methods aims to provide near 100% availability and topological correctness where other methods achieve only around 50% [S1, unofficial]. Displacement voxelization was developed to maintain visual quality while achieving higher performance by voxelizing low-resolution versions of triangle meshes [S1, unofficial].

## Sources
[S1] Point-Tessellated Voxelization, http://www.cs.columbia.edu/cg/raymond/gi12/paper100.pdf, fetched 2026-10-26 (unofficial)
