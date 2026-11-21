# Mesh-based Initialization

Also called: Mesh-to-Gaussian Initialization, Surface Sampling Initialization

## Overview
Mesh-based initialization converts a polygonal mesh into a set of 3D Gaussian splats to serve as the starting point for optimization or rendering. It solves the problem of establishing a geometric prior for Gaussian Splatting when a high-quality surface representation is already available, leveraging the explicit nature of Gaussians to support user-guided generation or scene-level inverse rendering. It differs from sibling methods like [Angle-pose Cuboid Uniform Initialization (ACUI)](4503-angle-pose-cuboid-uniform-initialization-acui.md) by deriving positions from continuous surface geometry rather than volumetric bounding boxes, and from [Learning-based and Generative Initialization](12356-learning-based-and-generative-initialization.md) by relying on deterministic geometric sampling rather than neural priors.

## Shared principles
The core mechanism involves sampling points from the surface of an input mesh and assigning Gaussian parameters to these points. The position of each Gaussian is determined by the sampled surface coordinates. The orientation and scale are typically aligned with the local surface geometry (e.g., normal-aligned, thin along the normal) to represent the surface as a collection of flat or slightly thickened primitives. This approach exploits the explicit nature of 3D Gaussian Splatting, allowing for direct integration with existing mesh processing pipelines.

## Subtopics

### User-Guided Generation Initialization
- **Definition**: A technique for initializing 3D Gaussian Splatting from user-provided 3D shapes to guide text-to-3D generation, leveraging the explicit nature of Gaussians [S1].
- **Step**: Generate initial points by uniformly sampling points on the surface of the provided user mesh [S1].
- **Parameter**: Set the position learning rate to a relatively small value to better preserve the user's desired geometric shape during optimization [S1].
- **Comparison**: Outperforms Latent-NeRF and Fantasia3D in both geometry and texture quality while mostly maintaining the user-provided geometric prior [S1].

### GeoSplatting with Existing Mesh Initialization
- **Definition**: An extension to GeoSplatting that initializes the optimization process with an existing triangle mesh (e.g., from NeuS or GOF) instead of building geometry from scratch, circumventing isosurface resolution limits and mask dependencies [S2].
- **Step**: Extract a triangle mesh from multi-view images using methods like NeuS or GOF [S2]. Convert the initial mesh into Gaussian points via MGadapter [S2]. Follow the remaining steps of the standard GeoSplatting pipeline [S2]. Refine the initial mesh through gradients derived from 3DGS rendering, leveraging the differentiable nature of MGadapter [S2].
- **Pitfall**: This two-stage extension is not applicable to specular objects because surface reconstruction methods lack effective PBR modeling, leading to noisy meshes that GeoSplatting cannot rectify [S2].
- **Comparison**: Unlike the standard pipeline which uses isosurfaces (limited by resolution and mask dependencies), this method extends applicability to scene-level inverse rendering and avoids those specific limitations [S2].

### Mesh-to-Splat Conversion Pipeline
- **Definition**: A pipeline that loads a polygonal mesh and converts it into a set of 3D Gaussian splats for rendering [S3, unofficial].
- **Step**: Invoke `mesh_to_splat(mesh, params.options, progress_callback)` to generate the Gaussian point cloud [S3, unofficial]. Iterate over output targets and save each generated splat using `saveSplat` with `RadExportMode::Stream` and SSOG options [S3, unofficial].
- **Pitfall**: If the input file is not a mesh, or if `mesh_to_splat` returns an error, log the failure and abort the pipeline [S3, unofficial].

## Background
Mesh-based initialization has emerged as a critical component in advanced Gaussian Splatting applications where geometric priors are valuable. It was notably adopted in works such as G SGEN for user-guided text-to-3D generation [S1] and GeoSplatting for physically-based inverse rendering [S2]. The approach allows for the integration of traditional computer graphics assets (meshes) with differentiable rendering frameworks, enabling hybrid workflows. Implementation details are also found in tools like LichtFeld-Studio [S3, unofficial], which provides a direct conversion pipeline.

## Sources
[S1] Text-to-3D using Gaussian Splatting, https://doi.org/10.1109/cvpr52733.2024.02022, fetched 2026-10-26
[S2] GeoSplatting: Towards Geometry Guided Gaussian Splatting for Physically-Based Inverse Rendering, https://doi.org/10.1109/ICCV51701.2025.02692, fetched 2026-10-30
[S3] MrNeRF/LichtFeld-Studio, https://github.com/MrNeRF/LichtFeld-Studio, fetched 2026-10-27 (unofficial)
