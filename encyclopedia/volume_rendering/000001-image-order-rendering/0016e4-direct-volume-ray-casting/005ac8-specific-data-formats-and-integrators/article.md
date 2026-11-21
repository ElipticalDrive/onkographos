# Specific Data Formats and Integrators


## Overview
This family categorizes direct volume ray casting methods by their specific data storage formats (e.g., 3D textures, fixed-point grids) and the integrators used to process them. It distinguishes between GPU-native texture-based ray marching and CPU-optimized fixed-point arithmetic, addressing performance bottlenecks in memory bandwidth and computational precision for large-scale datasets.

## Shared principles
All methods in this family adhere to the core mechanism of [Direct volume ray casting](19-direct-volume-ray-casting.md): defining rays, sampling volume data, classifying via transfer functions, and compositing using front-to-back integration. The divergence lies in the data representation (texture vs. grid) and the arithmetic precision (floating-point vs. fixed-point) used during sampling and accumulation.

## Subtopics

### [Raymarching with 3D Volume Texture](10839-raymarching-with-3d-volume-texture.md)
This method renders volumetric data by storing the voxel space in a GPU 3D texture (`sampler3D`) and performing raycasting directly in the fragment shader. It replaces 2D texture atlases with a single 3D volume texture, enabling larger XYZ volumes and sharing the in-memory representation with path-tracing and isosurface modes. Unlike meshing methods that require building and uploading geometry, it avoids the overhead of many draw calls by rendering a single screen-covering quad. It requires WebGL2 for `Data3DTexture` support.

### [Fixed-Point Volume Ray Casting](12056-fixed-point-volume-ray-casting.md)
Fixed-Point Volume Ray Casting is a direct volume rendering technique that replaces standard floating-point arithmetic with fixed-point operations for interpolation and accumulation to improve computational efficiency. It is specifically implemented in the Visualization Toolkit (VTK) via the `vtkFixedPointVolumeRayCastMapper` class. Unlike general-purpose ray casting which may rely on hardware floating-point units, this method optimizes performance by intercepting decimal parts and using integer-based arithmetic, making it particularly suitable for large-scale volumetric datasets such as seismic or medical images where memory bandwidth and CPU cycles are constrained. It builds on the standard front-to-back compositing algorithm but introduces optimizations like `MinMaxVolume` generation to skip empty voxels.

### CPU Volume Transport (Multi-Ray)
**Definition**: Solves transport for multiple rays in parallel on CPU by iterating over rays and calling the single-ray solver for each [S1, unofficial].
**Step**: Loop ray from 0 to `num_rays` - 1. Calculate `offset = ray * num_samples`. Call `solve_one_ray` with `sigma + offset`, `source + offset`, `num_samples`, `ds`, and `initial_intensity[ray]`. Store result in `output[ray]` [S1, unofficial].
**Layout**: Input arrays `sigma`, `source`, `initial_intensity` are row-major `[ray][sample]`. Output array `output` is `[ray]` [S1, unofficial].

## Background
The use of 3D textures for ray marching became standard with the advent of programmable GPUs and WebGL2, allowing direct access to volumetric data without intermediate geometry . Fixed-point ray casting was developed to address performance limitations in CPU-based rendering engines like VTK, particularly for large medical and seismic datasets where floating-point precision was not strictly necessary for visual fidelity but consumed significant resources [S1, unofficial].

## Sources
[S1] wyh1309/hpc_foundational_demos, https://github.com/wyh1309/hpc_foundational_demos, fetched 2026-10-26 (unofficial)
