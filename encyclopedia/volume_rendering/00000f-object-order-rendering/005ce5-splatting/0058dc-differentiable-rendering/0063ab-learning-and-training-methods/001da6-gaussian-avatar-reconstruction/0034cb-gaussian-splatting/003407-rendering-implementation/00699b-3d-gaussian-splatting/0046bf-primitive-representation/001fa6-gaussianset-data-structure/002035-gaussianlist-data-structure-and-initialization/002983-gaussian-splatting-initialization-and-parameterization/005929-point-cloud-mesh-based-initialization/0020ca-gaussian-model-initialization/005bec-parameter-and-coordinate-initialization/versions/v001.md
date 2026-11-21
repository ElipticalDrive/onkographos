# Parameter and Coordinate Initialization

## Overview
Parameter and Coordinate Initialization establishes the initial values for learnable attributes of 3D Gaussian primitives, including spherical harmonic coefficients, sampling coordinates, and material parameters, before the optimization process begins. It differs from sibling methods by focusing on the specific numerical setup of internal state variables (such as SH coefficients and feature sampling matrices) rather than the geometric placement of the Gaussians themselves.
## Shared principles
The family shares the principle of mapping external input data (RGB colors, 3D positions, or reference images) into the specific tensor formats required by the rendering pipeline's forward pass. This involves allocating zero-initialized tensors and applying deterministic transformations (such as normalization constants or linear projections) to set the base state.
## Subtopics

### [Spherical Harmonics Initialization](3163-spherical-harmonics-initialization.md)
See [Spherical Harmonics Initialization](3163-spherical-harmonics-initialization.md).

### GS-W Sampling Coordinate Initialization
Initializes $K$ learnable sampling coordinates per Gaussian point to enable adaptive feature sampling on $K$ feature maps [S1]. Each Gaussian point $i$ is assigned $K$ sampling coordinates $(sc_1^i, sc_2^i, \dots, sc_K^i) \in \mathbb{R}^{K \times 2}$ [S1]. To ensure that points close to each other have similar initial sampling coordinates, linear transformations are used to convert the 3D position $X_i \in \mathbb{R}^3$ of each Gaussian point into $K$ sampling coordinates [S1]. This is achieved by randomly generating $K$ matrices $(M^1, M^2, \dots, M^K) \in \mathbb{R}^{K \times 2 \times 3}$, constraining each matrix such that the sum of its rows equals 1, and multiplying each matrix $M^k$ with the Gaussian point's position $X_i$ [S1]. The initialization formula is:
$$ (sc_1^1, sc_2^2, \dots, sc_k^K)_{init} = (M^1 X_i, M^2 X_i, \dots, M^K X_i) \quad (1) $$
The number of feature maps $K$ is typically set to 3 to balance efficiency and effectiveness [S1].

### Parameter Initialization
Initializes material parameters for rendering. Emission is initialized to 0, roughness to 0.5, and specular to 0.5 [S2]. Spatially-varying diffuse color is initialized by building a median texture [S2]. This involves iterating over all available reference pictures and projecting them in texture space [S2]. An online median estimator is employed to avoid memory concerns when handling tens of thousands of views [S2]. The median is preferred over the mean filter to suppress view-dependent effects such as specular reflection [S2].

### Gaussian Point Splatting Parameterization
A method to calculate point size and transparency for volume rendering based on projected voxel size and Gaussian texture coverage [S3, unofficial]. The point size is calculated as:
$$ mPointSize = mGauspara \times m\_projectedVoxelSize $$
The transparency is calculated as:
$$ mTransParency = 20.2f / \_znum $$
Here, $mGauspara$ is the Gaussian texture coverage ratio, with a typical value of 2.0 [S3, unofficial]. $m\_projectedVoxelSize$ is the calculated projected size of a voxel on screen, dependent on window width and field of view [S3, unofficial]. $\_znum$ is the number of depth slices or z-resolution, used to calculate transparency [S3, unofficial].

## Background
The initialization of spherical harmonic coefficients for 3D Gaussian Splatting follows the guidelines of the original 3DGS paper, where hyperparameters are set according to established practices [S1]. The use of median textures for material initialization originates from differentiable rendering techniques for complex indoor scenes [S2]. The specific parameterization for point splatting in cloud rendering is derived from custom implementations [S3, unofficial].

## Sources
[S1] Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections, https://doi.org/10.1007/978-3-031-73116-7_20, fetched 2026-10-27
[S2] Material and Lighting Reconstruction for Complex Indoor Scenes with Texture-space Differentiable Rendering, https://doi.org/10.2312/sr.20211292, fetched 2026-10-29
[S3] qqdiguo/CloudRender-PointSplatting, https://github.com/qqdiguo/CloudRender-PointSplatting, fetched 2026-10-28 (unofficial)
