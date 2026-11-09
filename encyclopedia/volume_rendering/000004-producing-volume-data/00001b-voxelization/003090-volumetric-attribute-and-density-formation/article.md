# Volumetric Attribute and Density Formation


## Overview
Volumetric Attribute and Density Formation is the family of methods that map continuous geometric primitives or simulation data into discrete voxel grids by assigning specific physical or visual attributes (such as density, velocity, color, or occupancy) to each cell. It solves the problem of converting raw input data into a structured volume representation suitable for subsequent rendering or simulation steps. It differs from its siblings [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) and [Point Cloud Voxelization](1000-point-cloud-voxelization.md) by focusing on the semantic assignment of attribute values (e.g., density stretching, velocity interpolation, or coverage factors) rather than just binary occupancy or rasterization coverage.

## Shared principles
All methods in this family build on [Voxelization](12-voxelization.md) and share the following mechanism:
1.  **Coordinate Mapping**: Defining a grid resolution (number of cells per axis) and a bounding box (origin and extent) [G]. A world coordinate $x$ is mapped to a grid index $idx$ via:
    $$ idx = \lfloor \frac{x - origin}{voxel\_size} \rfloor $$
    where $voxel\_size$ is the physical dimension of a voxel [S1, unofficial][S3].
2.  **Attribute Evaluation**: Evaluating the input primitive (mesh, point, or parametric function) at specific locations (centers, intersections, or projections) to determine the value to store [G].
3.  **Value Assignment**: Storing the computed attribute in the voxel. This may include occupancy flags, density, color, normal vectors, or velocity fields derived from the source geometry or simulation [S12].

## Subtopics

### [Interior Voxelize](995-interior-voxelize.md)
Determines occupancy by sampling points and testing containment. Chosen for closed meshes where simple point-in-polyhedron tests suffice.

### [Shell-based Voxelization](1748-shell-based-voxelization.md)
Marks voxels within a shell radius of primitives. Chosen to define spatial support for boundary operations.

### [Velocity Voxelization](3082-velocity-voxelization.md)
Computes instantaneous velocity at boundary cells via triangle-slice intersection. Chosen for fluid simulation with moving obstacles.

### [Anisotropic Voxelization](4331-anisotropic-voxelization.md)
Stores directional data in separate grids for each of the six voxel faces. Chosen to prevent normal cancellation in thin models and reduce light leaking.

### [Box-filtering Density Estimation](5987-box-filtering-density-estimation.md)
Downsamples binary slicemaps using exact integer counting. Chosen for speed when smoothness is not required.

### [Volume Density Stretching](8033-volume-density-stretching.md)
Adjusts density based on geometric deformation (Jacobian). Chosen to simulate mass conservation in deforming volumes.

### [Multi-pattern Volumetric Construction via Voxel Multiplication](8874-multi-pattern-volumetric-construction-via-voxel-multiplicati.md)
Constructs 3D structures by multiplying orthogonal 2D pattern projections. Chosen for multi-view pattern encoding.

### [Coverage Factor Voxelization](9079-coverage-factor-voxelization.md)
Computes scalar coverage factors via supersampling in octrees. Chosen for anti-aliased density gradients and accurate volume integration.

### RGB-Alpha Volume Formation
- **Definition**: The process of forming an RGB-Alpha volume from raw data, which is a 3D four-vector data set where the first three components are R, G, and B color components and the last component, Alpha, represents opacity [S1, unofficial].
- **Steps**: Map data values to opacity values to act as a classification of the data one is interested in. Place an opaque background behind the RGB-Alpha volume [S1, unofficial].
- **Parameters**: Alpha value: 0 means totally transparent, 1 means totally opaque [S1, unofficial].

### Size Volume Computation
- **Definition**: A three-step method to create a size volume $S$ from an input volume, used as a tertiary attribute for transfer functions [S2, unofficial].
- **Steps**: Perform scale-space computation using anisotropic diffusion to create the scale space with better localization. Perform scale detection using the classical normalized Laplacian kernel to detect blobs as local maxima both in spatial and scale domains. Perform back projection utilizing Shepard's interpolation with Wendland polynomials for the detected blob tuple $(x, y, z, t)$. For each voxel, keep only the largest size value if it is part of features with multiple sizes, masking out smaller features [S2, unofficial].
- **Pitfalls**: Smaller features get masked out by larger ones. To avoid this, allow the user to specify an intensity range to compute a scale space specifically for that range [S2, unofficial].

### Unbiased Photon Plane/Volume Estimation
- **Definition**: A method by Bitterli and Jarosz that extends density estimation to higher-dimensional expressions (photon plane, photon volume) and uses a zero-order estimator to achieve unbiased results [S3].
- **Pitfalls**: Estimators suffer from singularities and require at least two bounces in the medium past any surface, leaving remaining transport to other techniques [S3].

## Background
The concept of forming volumetric attributes from raw data is fundamental to medical volume rendering and scientific visualization. Early work in medical volume rendering emphasized the formation of RGB-Alpha volumes from datasets such as cloud point data, slide data (e.g., MRI), and surface data, requiring special techniques and transfer functions to map data values to opacity for classification [S1, unofficial]. Later developments introduced tertiary attributes like size information for transfer function design, using scale-space computation and back-projection to detect blobs in spatial and scale domains [S2, unofficial]. In the context of participating media, density estimation was extended to higher-dimensional expressions like photon planes and volumes to achieve unbiased results, although these methods face singularities and bounce requirements [S3].

## Sources
[S1] Medical Volume Rendering Techniques, https://arxiv.org/pdf/1802.07710v1, fetched 2026-10-27 (unofficial)
[S2] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S3] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
