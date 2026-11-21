# Voxel-Specific Shading Pipelines

Also called: Voxel Lighting, Voxel Shading

## Overview
Voxel-Specific Shading Pipelines are a family of rendering techniques designed to compute illumination for data stored in discrete 3D grids (voxels). Unlike continuous surface shading which operates on smooth Signed Distance Functions (SDFs) or triangle meshes, these methods operate on the discrete topology of voxel cells, handling both surface-like boundaries and volumetric density . The family encompasses approaches ranging from per-voxel geometric shading with hard shadows to CPU-based palette index manipulation and dedicated volumetric ray-marching shaders for fluid simulations.

## Shared principles
All methods in this family operate on a voxel grid structure where data is indexed by integer coordinates $(x, y, z)$. The core distinction from continuous shading is the handling of geometry and lighting:
1.  **Discrete Coordinates:** Positions are often rounded or mapped to voxel cells. For surface shading, the "surface" is defined by the boundary between occupied and empty voxels, or by an isosurface threshold within the grid .
2.  **Neighbor Dependence:** Lighting calculations frequently depend on the occupancy or value of immediate neighboring voxels to determine occlusion, shading factors, or gradients, rather than casting long-distance rays for every light sample (except in explicit ray-marching variants) .
3.  **Data Types:** Inputs include block types, palette indices, or continuous scalar/vector fields (e.g., temperature, velocity) stored in the voxel cells .

## Subtopics

### [Voxel Shading Pipeline](3898-voxel-shading-pipeline.md)
A per-voxel lighting model that combines Lambertian diffuse, rim lighting, and GGX-like specular terms with hard shadows. It differs by operating on discrete voxel coordinates, rounding hit positions, and using secondary ray occlusion checks for shadows. Choose this for high-quality, physically-inspired shading of voxel-based geometry.

### [Fast Shading for Voxel Engines](5606-fast-shading-for-voxel-engines.md)
A CPU-based algorithm that simulates directional sunlight by modifying voxel palette indices based on neighbor occupancy. It differs by using simple integer arithmetic and clamping to tone groups, avoiding complex geometric calculations. Choose this for performance-critical voxel engines where visual fidelity is secondary to speed.

### [Voxel Shader Volumetric Ray Marching](16930-voxel-shader-volumetric-ray-marching.md)
A dedicated pipeline for volumetric data (fire, smoke) that reads scalar/vector fields from grids like Phoenix FD or V-Ray Volume Grids. It differs by providing UI-driven controls for emission, scattering, and displacement, handling volumetric opacity accumulation rather than surface reflection. Choose this for simulating and rendering volumetric effects.

## Background
The concept of shading voxel data emerged with the rise of voxel-based games and simulation tools. Early implementations, such as those in voxel engines, relied on simple neighbor checks to simulate lighting via palette manipulation to save CPU cycles . As GPU capabilities increased, more complex pipelines were developed to apply standard BRDFs (like GGX) to voxel surfaces, treating the voxel grid as a discrete mesh . Concurrently, 3D simulation software (e.g., Maya with Phoenix FD, V-Ray) developed specialized shaders to handle the volumetric nature of fluid and fire data, which standard surface materials could not render .

## Sources
[S1] Voxel Shading Pipeline notes, URL, date fetched
[S2] Fast Shading for Voxel Engines notes, URL, date fetched
