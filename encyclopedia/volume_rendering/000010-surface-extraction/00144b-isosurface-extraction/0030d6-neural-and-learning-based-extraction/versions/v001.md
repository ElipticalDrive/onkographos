# Neural and Learning-based Extraction

Also called: Differentiable Isosurface Extraction, Gradient-based Mesh Optimization

## Overview
Neural and Learning-based Extraction generates isosurfaces from implicit fields learned by neural networks (e.g., NeRF, DVR, SDF networks) by integrating the extraction process into the optimization loop or using adaptive sampling strategies. Unlike static methods like [Marching Cubes](30-isosurface-extraction.md) or [Surface Nets](921-surface-nets.md) that operate on a fixed grid, these methods allow the mesh topology and geometry to be refined via backpropagation, enabling direct optimization of visual, geometric, or physical objectives [S2, unofficial]. It sets itself apart from siblings by treating the isosurface extraction as a differentiable operation or an adaptive query process, rather than a post-processing step on a discretized volume.

## Shared principles
All methods in this family operate on an implicit function $f: \mathbb{R}^3 \rightarrow \mathbb{R}$ (such as occupancy, SDF, or density) parameterized by a neural network or learned voxel grid. The core mechanism involves evaluating this function at specific 3D points to determine surface presence or distance. Key shared characteristics include:
1.  **Implicit Representation**: The surface is defined as the zero-level set (or threshold crossing) of the learned field, not as an explicit mesh during training.
2.  **Differentiability or Adaptive Querying**: Either the extraction function is differentiable with respect to the field parameters (allowing gradient-based optimization of the mesh) or the query points are adaptively selected to refine the boundary estimate iteratively.
3.  **Thresholding**: A specific threshold value (e.g., 0.5 for occupancy, 0.0 for SDF, or a density value $\sigma$) is used to define the isosurface.
4.  **Grid or Point-based Evaluation**: The field is evaluated either on a structured voxel grid (for methods like FlexiCubes) or at unstructured query points (for methods like MISE or NeRF extraction).

## Subtopics

### [MISE (Meshing Isosurfaces by Extracting)](977-mise-meshing-isosurfaces-by-extracting.md)
MISE generates a 3D mesh from a trained Differentiable Volumetric Rendering (DVR) model by querying an implicit occupancy field using an adaptive resolution strategy. It sets itself apart from static grid-based siblings like [Surface Nets](921-surface-nets.md) or [Marching Cubes](30-isosurface-extraction.md) by iteratively refining the mesh geometry only where necessary, rather than evaluating a uniform grid at a fixed resolution. It is chosen when high-fidelity mesh extraction from occupancy networks is required without the memory cost of a dense global grid.

### [FlexiCubes](3669-flexicubes.md)
FlexiCubes is a high-quality isosurface extraction method designed specifically for gradient-based mesh optimization with respect to geometric, visual, or physical objectives. It extracts a triangle mesh from a Signed Distance Field (SDF) defined on a voxel grid by using learnable cube weights (beta, alpha, gamma) and optional vertex deformations. Unlike static extraction methods, FlexiCubes allows the topology and geometry to be refined via backpropagation, making it suitable for differentiable rendering pipelines. It sets itself apart from siblings like [MISE](977-mise-meshing-isosurfaces-by-extracting.md) by providing a differentiable, weight-optimized representation that can be directly integrated into optimization loops for generative 3D tasks.

#### NeRF Density Field Mesh Extraction
**Definition**: Extracts a triangle mesh from the learned density field by querying $\sigma$ on a 3D grid and running marching cubes at a threshold [S1, unofficial].
**Steps**:
1. Create 1D coordinate array: `lin = torch.linspace(-scene_scale, scene_scale, resolution, device=device)` [S1, unofficial].
2. Create 3D grid: `xx, yy, zz = torch.meshgrid(lin, lin, lin, indexing="ij")` [S1, unofficial].
3. Stack and reshape to points: `pts = torch.stack([xx, zz, yy], dim=-1).reshape(-1, 3)` (swaps Y/Z for NeRF convention) [S1, unofficial].
4. Query density $\sigma$ in chunks: `_, sigma = coarse_model(pts[i:i+chunk], dummy_dirs[i:i+chunk])` [S1, unofficial].
5. Concatenate and reshape to `[resolution, resolution, resolution]` [S1, unofficial].
6. Run marching cubes: `verts, faces, normals, _ = measure.marching_cubes(sigma_grid, level=sigma_threshold)` [S1, unofficial].
7. Rescale vertices from `[0, resolution]` to world coordinates: `verts = verts / (resolution - 1) * (2 * scene_scale) - scene_scale` [S1, unofficial].
**Parameters**:
- `resolution`: voxel grid resolution (default 128) [S1, unofficial].
- `scene_scale`: half-extent of the query box (default 1.5) [S1, unofficial].
- `sigma_threshold`: iso-value for the surface (default 50.0) [S1, unofficial].
- `chunk`: number of points queried per batch (default 32768) [S1, unofficial].
**Pitfall**: Y/Z axes are swapped during grid creation to match NeRF's coordinate convention [S1, unofficial].

## Background
Originating from the rise of Neural Radiance Fields (NeRF) [S1, unofficial], this family addresses the limitation of traditional isosurface extraction which requires dense, memory-intensive volumetric grids. Early NeRF implementations used Marching Cubes on sampled density fields [S1, unofficial]. Subsequent developments like MISE introduced adaptive querying to reduce memory usage, while FlexiCubes introduced differentiable extraction to enable end-to-end optimization of 3D shapes from 2D images or other objectives [S2, unofficial]. These methods are primarily used in 3D reconstruction, generative 3D modeling, and differentiable rendering pipelines.

## Sources
[S1] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
