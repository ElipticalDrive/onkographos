# Distance Field Representations and Sampling

Also called: Distance Field Evaluation, SDF Sampling

## Overview
This family covers the methods for evaluating scalar distance values from various distance field representations during ray marching. It solves the problem of efficiently retrieving the signed distance $f(\mathbf{p})$ for a query point $\mathbf{p}$ from discrete, procedural, or specialized data structures. It is distinguished from [Signed Distance Field Representations and Operations](13339-signed-distance-field-representations-and-operations.md) by focusing on the *retrieval* and *sampling* mechanics (handling boundaries, memory layouts, and differentiability) rather than the *construction* or *composition* of the fields themselves.

## Shared principles
The family operates on the principle that a scene is defined by a scalar field $f(\mathbf{p})$ where the zero-level set $f(\mathbf{p}) = 0$ represents the surface. For sphere tracing to be valid, the function must satisfy the Lipschitz condition with constant 1, ensuring $|f(\mathbf{p}_1) - f(\mathbf{p}_2)| \le \|\mathbf{p}_1 - \mathbf{p}_2\|$. This guarantees that the value $f(\mathbf{p})$ is a safe lower bound on the distance to the surface, allowing the ray marcher to step forward by exactly $f(\mathbf{p})$ without tunneling [S2, S5, 9643, unofficial].

Composition is achieved through Boolean operations on the distance values:
*   **Union**: $f_{union}(\mathbf{p}) = \min(f_1(\mathbf{p}), f_2(\mathbf{p}))$ [S39, unofficial].
*   **Intersection**: $f_{intersect}(\mathbf{p}) = \max(f_1(\mathbf{p}), f_2(\mathbf{p}))$ [S39, unofficial].
*   **Subtraction**: $f_{sub}(\mathbf{p}) = \max(-f_1(\mathbf{p}), f_2(\mathbf{p}))$ [S39, unofficial].

Transformations are applied by mapping the query point $\mathbf{p}$ into the local coordinate space of the primitive using the inverse transformation matrix $M^{-1}$ before evaluation [S3850]. Procedural noise (e.g., FBM) is often used to perturb these analytical surfaces, creating hypertextures where the isosurface is defined by $SDF(\mathbf{p}) + B_n(\mathbf{p}) = 0$ [S1, unofficial].

## Subtopics

### [Bounded 3D Distance Texture Ray Marching](1115-bounded-3d-distance-texture-ray-marching.md)
Bounded 3D Distance Texture Ray Marching is a method for evaluating a Signed Distance Function (SDF) using a discrete 3D texture to represent an implicit surface, specifically designed to handle boundary conditions and invalid distance values at texture edges. It solves the problem of sampling artifacts that occur when a ray marches outside the valid interior of a distance texture but remains within the texture's physical volume. Unlike standard texture sampling which may return noisy or invalid data near the edges, this method employs two bounding regions (an inner "red" box and an outer "green" box) to ensure that ray marching stops correctly at the texture boundary or uses valid interior data, preventing the "red box" artifact in ambient occlusion calculations. It builds on [Distance Field Representations and Sampling](8725-distance-field-representations-and-sampling.md).

### Sampler Abstraction
#### definition
A trait-based abstraction for sampling distance fields defined in terms of specific Distance metrics. It allows samplers to abstract sampling across multiple different Distance metrics [S1, unofficial].
#### step
Implementation for ScalarSource: If S implements ScalarSource, Sampler implements Sample<Signed>. The sample method calls self.source.sample_scalar(p) [S1, unofficial]. Implementation for VectorSource: If S implements VectorSource, Sampler implements Sample<Directed>. The sample method calls self.source.sample_vector(p) [S1, unofficial]. Implementation for HermiteSource: If S implements HermiteSource, Sampler implements sample_normal method which calls self.source.sample_normal(p) [S1, unofficial].
#### layout
Sampler struct contains: source (&'a S). Implements Sample<D> trait for Signed and Directed distance types based on source implementing ScalarSource or VectorSource [S1, unofficial].

### Ray Marching through Distance Fields
#### definition
A traversal method where ray marching is performed through distance fields. Naive storage of one distance value per voxel is impractical for large grids due to memory and bandwidth costs [S2, unofficial].
#### step
Memory costs are reduced by lowering field resolution to 1:4 scale (one distance value per 64 voxels), combined with occupancy masks. Masks are queried whenever sampled distance values are zero [S2, unofficial].
#### number
Pre-processing takes 10x more GPU time than XBrickMap. Traversal is not considerably shorter than space partitioning methods (~8% difference) [S2, unofficial].

### OctantDF
#### definition
Space skipping traversal using tiled 256³ 8-directional distance fields at 1:4 resolution, combined with 4³ occupancy bitmasks [S2, unofficial]. A distance field implementation that creates 8 separate manhattan fields, each considering only voxels ahead of the associated ray octant direction. Distances are only propagated across rows of opposing axis directions with respect to the octant [S2, unofficial].
#### layout
Memory cost is 8 bytes per 4³ voxel plus 1 bit per 1³ voxel [S2, unofficial].
#### number
Compared to traditional distance fields, it lowers the overall number of iterations by ~20-25% and increases throughput for incoherent rays by ~30-50% [S2, unofficial]. On Sponza 2k, OctantDF achieves 129.0 Mrays/s for primary rays and 84.9 Mrays/s for primary + 1 diffuse bounce [S2, unofficial].

### Distance to Value Function
#### definition
A ray function that computes the distance along the ray until a specific scalar value $\sigma$ is encountered. Useful to emphasize depth where specific tissue is located [S3, unofficial].

### Probabilistic Directed Distance Fields (PDDFs)
#### definition
A variant of DDFs that differentiably models the inherent discontinuities in the underlying field [S4]. PDDFs are constructed to handle the discontinuities that make standard DDFs difficult to train with differentiable neural networks [S4].

### Signed Height Indicator
#### definition
A 2.5D indicator $\tilde{h} = [u, v, \bar{d}]$ combining UV coordinates and signed distance to resolve ambiguity for points perpendicular to the mesh surface [S5, unofficial].
#### step
Calculate signed distance $\bar{d}$: project point $x$ onto mesh surface at $x_0$, obtain direction $\tilde{r} = x - x_0$, get surface normal $\tilde{n}$ at $x_0$, and compute $\bar{d} = \text{sign}(\tilde{r} \cdot \tilde{n}) \|\tilde{r}\|_2$ [S5, unofficial].
#### formula
Define signed height: $\tilde{h} = [u, v, \bar{d}]$, where $[u, v]$ is the 2D coordinate on the UV map and $\bar{d}$ is the signed distance [S5, unofficial].

## Background
Distance field sampling is a fundamental component of implicit surface rendering, evolving from analytical sphere tracing to complex hierarchical and differentiable representations. Early implementations relied on analytical primitives [S1, unofficial]. As scene complexity increased, discrete 3D textures became common, necessitating boundary handling techniques like the bounded texture approach to prevent sampling artifacts [S1, S2, unofficial]. Concurrently, research into efficient traversal for large voxel grids led to optimized structures like OctantDF and XBrickMap, which trade pre-processing time for traversal speed [S2, unofficial]. More recently, the field has expanded into differentiable rendering with Probabilistic Directed Distance Fields (PDDFs) to support neural network training, and specialized indicators like Signed Height for 2.5D manifold representations [S4][S5, unofficial].

## Sources
[S1] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
[S2] dubiousconst282/VoxelRT, https://github.com/dubiousconst282/VoxelRT, fetched 2026-10-24 (unofficial)
[S3] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S4] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
[S5] UVA: Towards Unified Volumetric Avatar for View Synthesis, Pose rendering, Geometry and Texture Editing, https://doi.org/10.48550/arXiv.2304.06969, fetched 2026-10-29 (unofficial)
