# Ambient Occlusion via Geometry/Neighbors

Also called: Local Geometry-Based Ambient Occlusion, Voxel Neighbor Occlusion

## Overview
This family of methods estimates local ambient occlusion by querying the immediate geometric neighborhood of a point—typically via Signed Distance Fields (SDF), voxel occupancy grids, or vertex neighborhoods—to approximate visibility without explicit ray tracing against triangle meshes or screen-space depth buffers [S1, S2, S3, S4, unofficial]. It sets itself apart from [JAOCC Shading](3634-jaocc-shading.md) and [Local Ambient Occlusion](4977-local-ambient-occlusion.md) by directly evaluating geometric distance or discrete neighbor occupancy to determine occlusion factors, rather than relying on stochastic sampling or precomputed global buffers [G].

## Shared principles
All methods in this family share the core mechanism of evaluating visibility or illumination at a specific point $\mathbf{p}$ within or on the boundary of a volume, based on a light source direction $\mathbf{l}$ or environment [G].

1.  **Visibility/Transmittance Evaluation:** The fundamental operation is determining how much light reaches $\mathbf{p}$ from a source [G]. This involves integrating the extinction coefficient $\sigma_t(\mathbf{x})$ along a path from the source to $\mathbf{p}$ (or vice versa) [G]. The transmittance $T$ is calculated as:
    $$ T = \exp\left(-\int_{\mathbf{p}_{source}}^{\mathbf{p}} \sigma_t(\mathbf{x}) \, d\mathbf{x}\right) $$
    In discrete implementations, this integral is approximated by summing $\sigma_t \cdot \Delta s$ over samples along the ray [G].

2.  **Local Shading Models:** For surface-like appearance in volumes, normals are approximated from the volume gradient $\mathbf{g} = \nabla V$ [G]. Usually, a simple absorption and emission model is used because it is relatively cheap to evaluate and gives good enough results [S1].

3.  **Shadowing Approximation:** Shadows are computed by testing occlusion along the direction to the light source [G].

4.  **Ambient Occlusion:** Indirect illumination is approximated by integrating over the hemisphere or using simplified models (e.g., upward ray for sky light) [G]. The ambient contribution $L_{ambient}$ is often modulated by a shadowing factor based on local density:
    $$ L_{ambient} = L_{sky} \cdot \exp(-D_{up}) \cdot AO_{factor} $$
    where $D_{up}$ is the integrated density along an upward ray [G].

## Subtopics

### [Ambient Occlusion via SDF/Geometry](2527-ambient-occlusion-via-sdf-geometry.md)
This method estimates local ambient occlusion by querying a geometric representation, typically a Signed Distance Field (SDF) or structured voxel grid, to approximate visibility without explicit ray tracing against triangle meshes or screen-space depth buffers. It sets itself apart from [JAOCC Shading](3634-jaocc-shading.md) and [Local Ambient Occlusion](4977-local-ambient-occlusion.md) by directly evaluating geometric distance to determine occlusion factors [G].

### [JAOCC Shading](3634-jaocc-shading.md)
JAOCC Shading is a voxel-based ambient occlusion technique that computes a depth-weighted dimming factor by counting non-empty neighbors in a cubic neighborhood. It addresses hard, physically implausible lighting transitions at voxel boundaries by dynamically computing the normal of the cut voxel face from the local neighborhood geometry instead of using fixed AABB edge normals.

### [Local Ambient Occlusion](4977-local-ambient-occlusion.md)
Local Ambient Occlusion (LAO) methods estimate the blocking of indirect light in volume rendering by analyzing the local density distribution or sampling stochastic rays around a point, rather than relying on global geometry or hard shadow maps. Unlike siblings that use Signed Distance Fields (SDF) [2527-ambient-occlusion-via-sdf-geometry.md] or precomputed voxel buffers [6646-voxel-ambient-occlusion.md], LAO methods typically compute occlusion on-the-fly or via lightweight precomputation tailored to the volume's statistical properties or stochastic sampling, providing softer, more volumetric shadow appearances [4502-local-ambient-occlusion-lao.md].

### [Vertex Ambient Occlusion Approximation](6623-vertex-ambient-occlusion-approximation.md)
Vertex Ambient Occlusion Approximation is a lightweight, post-processing shading technique that darkens vertices based on the local density of solid voxels in their immediate neighborhood, simulating shadowing in crevices and concavities. It differs from [Voxel Ambient Occlusion](6646-voxel-ambient-occlusion.md) by operating on discrete vertex data rather than continuous volume integration, and from [Gradient-based Occlusion Information Computation](9554-gradient-based-occlusion-information-computation.md) by relying on explicit neighbor sampling rather than analytical gradient estimation.

### [Voxel Ambient Occlusion](6646-voxel-ambient-occlusion.md)
Voxel Ambient Occlusion (Voxel AO) estimates ambient occlusion for a surface point by sampling the solidity of surrounding voxels in a grid relative to the surface normal, simulating light blockage from adjacent geometry. It differs from [Ambient Occlusion via SDF/Geometry](2527-ambient-occlusion-via-sdf-geometry.md) by operating on discrete voxel occupancy grids rather than continuous Signed Distance Fields, and from [JAOCC Shading](3634-jaocc-shading.md) by using explicit neighbor queries instead of precomputed cache structures. It builds on [Ambient Occlusion via Geometry/Neighbors](5936-ambient-occlusion-via-geometry-neighbors.md) by providing a specific implementation for voxelized data.

### [Gradient-based Occlusion Information Computation](9554-gradient-based-occlusion-information-computation.md)
This method computes a view-independent occlusion metric for volume data that preserves sharp boundaries between structures, addressing the isotropic blurring inherent in simple spherical mean ambient occlusion. It differs from sibling methods like [Voxel Ambient Occlusion](6646-voxel-ambient-occlusion.md) by modulating the neighborhood mean with a gradient-based weight derived from Perona-Malik diffusion principles, ensuring that edges are not smoothed out during the occlusion calculation.

### Standard Deviation of Ambient Occlusion
#### definition
- [S1] A metric that quantifies the variation of ambient occlusion values along all directions for a voxel, allowing distinction between objects with identical scalar field values but different local structures.
#### formula
- [S1] $O_{std}(p, \vec{d}) = \sqrt{\frac{1}{H} \sum_{i=1}^{H} (A(p, \hat{\omega}_i, \vec{d}))^2 - (\frac{1}{H} \sum_{i=1}^{H} A(p, \hat{\omega}_i, \vec{d}))^2}$ where $H$ is the number of different directions, $A$ is the directional occlusion function, $p$ is the voxel, and $\vec{d}$ is the direction vector.
#### parameter
- [S1] H: The number of different directions sampled for the ambient occlusion calculation.
#### pitfall
- [S1] In simple cases with few objects (e.g., two objects with same intensity/size), using the second derivative field $f''(x)$ as input to $O_{std}$ fails because boundaries are highlighted while interiors become near-zero, causing identical $O_{std}$ values for distinct objects. In these cases, use the original scalar field $f(x)$ as input instead.

## Background
The concept of using local geometric neighborhood queries to approximate ambient occlusion in voxel-based or grid-based volumes has been developed to provide efficient, view-independent shading that captures crevice darkening without the cost of full ray tracing. Techniques such as JAOCC [S3] and Voxel AO [S1] emerged from the need to shade voxel data (e.g., in medical imaging or voxel games) efficiently. The gradient-based approach [S1] was introduced to address the isotropic blurring of standard spherical mean occlusion, preserving sharp boundaries in volume classification. The standard deviation metric [S1] extends this by quantifying directional variation to distinguish objects with identical scalar values but different local structures.

## Sources
[S1] Boundary-structure-aware transfer functions for volume classification, https://doi.org/10.1145/3139295.3139306, fetched 2026-10-31
