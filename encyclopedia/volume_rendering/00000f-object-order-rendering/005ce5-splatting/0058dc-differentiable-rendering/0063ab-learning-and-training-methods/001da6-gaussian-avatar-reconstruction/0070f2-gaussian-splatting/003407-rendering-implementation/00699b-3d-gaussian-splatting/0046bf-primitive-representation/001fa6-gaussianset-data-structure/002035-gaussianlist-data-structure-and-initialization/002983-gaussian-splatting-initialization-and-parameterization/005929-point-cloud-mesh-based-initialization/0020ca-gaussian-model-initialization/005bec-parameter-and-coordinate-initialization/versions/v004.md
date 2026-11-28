# Parameter and Coordinate Initialization


## Overview
Parameter and Coordinate Initialization establishes the initial values for learnable attributes of 3D Gaussian primitives, including position, covariance, opacity, spherical harmonic coefficients, and inverse rendering material parameters. It solves the problem of starting optimization from a stable, physically plausible state to prevent gradient explosion and ensure convergence. It differs from [Gaussian Model Initialization](2349-gaussian-model-initialization.md) by focusing specifically on the numerical assignment of tensor values rather than the geometric generation of the primitive set, and from [Spherical Harmonics Evaluation](3848-spherical-harmonics-evaluation.md) by being a setup routine rather than a rendering step.

## Shared principles
The family shares the principle of initializing parameters to neutral or geometrically consistent states that match the input data distribution. For color attributes, this involves mapping input RGB values to the appropriate basis functions (e.g., spherical harmonics) while zeroing higher-order terms. For material parameters in inverse rendering, this involves setting albedo, roughness, and emission to values that produce a neutral initial render (e.g., intensity ~0.5) to avoid saturation or vanishing gradients. For coordinate-based feature sampling, this involves projecting 3D positions into 2D feature spaces using linear transformations constrained to preserve locality.

## Subtopics

### [Spherical Harmonics Initialization](3163-spherical-harmonics-initialization.md)
[Spherical Harmonics Initialization](3163-spherical-harmonics-initialization.md) sets the initial view-dependent color coefficients for 3D Gaussian primitives by mapping input RGB colors to the zeroth-order (DC) spherical harmonic basis. It solves the problem of providing a physically plausible starting point for view-dependent appearance optimization, ensuring that the initial radiance matches the observed color while leaving higher-order coefficients at zero. It differs from [Spherical Harmonics Evaluation](3848-spherical-harmonics-evaluation.md) by being a setup routine rather than a rendering step, and from [Splat Initialization from Normalized Coordinates](3595-splat-initialization-from-normalized-coordinates.md) by deriving color from external RGB data rather than spatial coordinates.

### [Parameter Initialization](16048-parameter-initialization.md)
[Parameter Initialization](16048-parameter-initialization.md) establishes neutral or geometrically plausible starting values for material and lighting parameters in inverse rendering pipelines, preventing optimization divergence and gradient explosion. Unlike position or opacity initialization, which derives directly from input geometry, these parameters (albedo, roughness, environment lighting) are initialized to neutral states (e.g., albedo ~0.5, emission 0) to ensure the initial render is physically plausible and stable.

### GS-W Sampling Coordinate Initialization
- **Definition**: Initializes $K$ learnable sampling coordinates per Gaussian point to enable adaptive feature sampling on $K$ feature maps [S1].
- **Steps**:
  1. Assign each Gaussian point $i$ $K$ sampling coordinates $(sc_1^i, sc_2^i, \dots, sc_K^i) \in \mathbb{R}^{K \times 2}$ [S1].
  2. Randomly generate $K$ matrices $(M^1, M^2, \dots, M^K) \in \mathbb{R}^{K \times 2 \times 3}$ [S1].
  3. Constrain each matrix such that the sum of its rows equals 1 [S1].
  4. Multiply each matrix $M^k$ with the Gaussian point's 3D position $X_i \in \mathbb{R}^3$ to compute the initial sampling coordinates [S1].
- **Formula**:
  $$ (sc_1^1, sc_2^2, \dots, sc_k^K)_{init} = (M^1 X_i, M^2 X_i, \dots, M^K X_i) \quad (1) $$
- **Parameters**: The number of feature maps $K$ is typically set to 3 to balance efficiency and effectiveness [S1].

### Gaussian Point Splatting Parameterization
- **Definition**: A method to calculate point size and transparency for volume rendering based on projected voxel size and Gaussian texture coverage [S2, unofficial].
- **Formula**:
  $$ mPointSize = mGauspara \times m\_projectedVoxelSize $$
  $$ mTransParency = 20.2f / \_znum $$
- **Parameters**:
  - $mGauspara$: Gaussian texture coverage ratio. Typical value: 2.0 [S2, unofficial].
  - $m\_projectedVoxelSize$: Calculated projected size of a voxel on screen, dependent on window width and field of view [S2, unofficial].
  - $\_znum$: Number of depth slices or z-resolution, used to calculate transparency [S2, unofficial].

## Background
The methods described in this family originate from the 3D Gaussian Splatting (3DGS) framework [S1] and its extensions for unconstrained image collections and inverse rendering. The standard 3DGS initialization uses a point cloud (e.g., from COLMAP) to set positions and covariances, while color is initialized via spherical harmonics [S1]. Inverse rendering extensions, such as those discussed in [S1] and [S2, unofficial], introduce additional parameters for albedo, roughness, and environment lighting, requiring specialized initialization strategies to ensure stable optimization. The GS-W method [S1] extends this to include learnable sampling coordinates for feature maps. The Gaussian Point Splatting method [S2, unofficial] provides a specific parameterization for point size and transparency in a cloud rendering context.

## Sources
[S1] Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections, https://doi.org/10.1007/978-3-031-73116-7_20, fetched 2026-10-27
[S2] qqdiguo/CloudRender-PointSplatting, https://github.com/qqdiguo/CloudRender-PointSplatting, fetched 2026-10-28 (unofficial)
