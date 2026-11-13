# Specific Implicit Primitives


## Overview
Specific Implicit Primitives are concrete scalar field functions $f(\vec{x})$ that define geometric shapes by evaluating a mathematical expression at query points, serving as the atomic building blocks for procedural volume rendering [S1, S2, S3, unofficial]. Unlike generic implicit surface definitions, these primitives provide closed-form analytical solutions (or precise approximations) for specific topologies, enabling efficient ray marching and voxel generation without discrete mesh storage [S1, unofficial][S2, unofficial]. They are distinguished from sibling methods like [Metaball Field Evaluation](1017-metaball-field-evaluation.md) by their single-object focus and lack of blending logic, and from [Implicit Field Composition](2652-implicit-field-composition.md) by their atomic nature rather than hierarchical combination.

## Shared principles
This family shares the core mechanism defined in [Implicit Surface Primitives](1800-implicit-surface-primitives.md): the geometry is defined as the zero-level set (or isosurface at value $v$) of a scalar field function $f: \mathbb{R}^3 \to \mathbb{R}$ [S1, S2, unofficial]. The evaluation is on-demand, computing $f(\vec{x})$ for any $\vec{x}$ in the domain $\Omega$ [Implicit Surface Primitives](1800-implicit-surface-primitives.md).

## Subtopics

### [Torus Implicit Function](1322-torus-implicit-function.md)
Computes the exact signed distance to a torus surface defined by major radius $R$ and minor radius $r$. It is chosen when exact analytical distance is required for stable ray marching, avoiding the numerical instability of general polynomial solvers.

### [Metaball Field Evaluation](1017-metaball-field-evaluation.md)
Evaluates the sum of radial basis functions from multiple centers. Chosen for organic, blobby shapes where fields blend.

### [Distance to Hypertextures](1555-distance-to-hypertextures.md)
Computes distance to surfaces defined by procedural texture patterns. Chosen for complex, non-analytic surface details.

### [Superelliptic Blends](2051-superelliptic-blends.md)
Uses superelliptic equations to blend primitives with smooth transitions. Chosen for hard-surface modeling with customizable sharpness.

### [Implicit Field Composition](2652-implicit-field-composition.md)
Combines multiple implicit fields using set operations (union, intersection, difference). Chosen for constructing complex scenes from simple primitives.

### [Soft Metablobby SDF](2753-soft-metablobby-sdf.md)
Provides a smooth, differentiable approximation of metaball fields. Chosen for gradient-based shading and soft-body simulation.

### [Implicit Surface Definition](2798-implicit-surface-definition.md)
General framework for defining arbitrary implicit surfaces. Chosen when no standard primitive fits the geometry.

### [Half-Space Field Generation](4006-half-space-field-generation.md)
Defines volumes using linear inequalities (half-spaces). Chosen for convex polyhedral shapes and constructive solid geometry.

### [Implicit Geometry Sampling](6685-implicit-geometry-sampling.md)
Generates discrete voxel data from implicit definitions. Chosen for rasterization-based rendering pipelines.

### [Metaball-based Envelope Painting](8803-metaball-based-envelope-painting.md)
Uses metaball fields to define weight distributions for painting. Chosen for texture and material blending.

### Gyroid Surface Evaluation
- **Definition**: Evaluates the scalar field for a Triply Periodic Minimal Surface (TPMS) Gyroid at point $(x,y,z)$ [S1, unofficial].
- **Formula**: $f(x,y,z) = \sin(sx)\cos(sy) + \sin(sy)\cos(sz) + \sin(sz)\cos(sx)$, where $sx = x \cdot scale, sy = y \cdot scale, sz = z \cdot scale$ [S1, unofficial].
- **Parameter**: `scale`: The scaling factor for the gyroid coordinates. Default value is 4.5 [S1, unofficial].

### Thin Shell Implicit Function
- **Definition**: Implicit function for a thin spherical shell: $f(p) = |x^2+y^2+z^2 - r^2| - \epsilon$ [S2, unofficial].
- **Parameter**: `radius`: Sphere radius for the shell, default 1.0 [S2, unofficial].
- **Parameter**: `epsilon`: Half-thickness of the shell, default 1e-3 [S2, unofficial].

### Rectangular Prism Generation
- **Definition**: Generates a rectangular prism centered at the origin [S3, unofficial].
- **Parameter**: `x, y, z`: prism dimensions. Default: 5x5x5 [S3, unofficial].
- **Layout**: Returns a data.frame of voxel coordinates [S3, unofficial].
- **Pitfall**: Only odd side lengths are generated. Even dimensions are rounded up to the next odd number due to integer coordinate quantization [S3, unofficial].

## Background
The concept of implicit primitives originates from the work of William E. Lorensen and Harvey E. Cline, who introduced the Marching Cubes algorithm in 1987, which popularized the use of scalar fields for surface extraction [S1, unofficial][S2, unofficial]. The specific analytical distance functions, such as the torus SDF, were later formalized in the context of ray marching and signed distance fields (SDFs) by researchers like Inigo Quilez and others in the real-time rendering community [S1, unofficial]. The Gyroid surface, a TPMS, has roots in differential geometry and minimal surface theory, applied to procedural modeling in computer graphics [S1, unofficial]. Rectangular prism generation for voxel visualization is a more recent development in data visualization libraries, such as the `isocubes` R package, which focuses on efficient voxel coordinate generation for isometric rendering [S3, unofficial]. These primitives are widely used in real-time ray tracing, procedural texture generation, and scientific visualization where memory efficiency and geometric precision are critical [S1, unofficial][S2, unofficial][S3, unofficial].

## Sources
[S1] nff747/marching-cubes-wgsl, https://github.com/nff747/marching-cubes-wgsl, fetched 2026-10-27 (unofficial)
[S2] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S3] isocubes: Voxel Data Visualization with Isometric Cubes, https://doi.org/10.32614/cran.package.isocubes, fetched 2026-10-26 (unofficial)
