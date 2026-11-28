# Implicit Surface Primitives


## Overview
Implicit Surface Primitives define 3D geometry as the zero-level set of a scalar field function $f: \mathbb{R}^3 \to \mathbb{R}$, solving the problem of representing complex, continuous shapes without explicit vertex or polygon data [G]. This family is distinguished from procedural noise methods by its reliance on geometric distance or density metrics rather than stochastic texture generation, and from chunk generation by its analytic, on-demand evaluation nature [G].

## Shared principles
All methods in this family rely on the evaluation of a scalar field function $f$ over a domain $\Omega \subseteq \mathbb{R}^3$ [G]. The core mechanism is the identification of the surface as the locus of points where the function output equals a specific constant, typically zero [G].

**Mathematical Definition:**
The implicit surface is defined as the set of points $\vec{x}$ satisfying:
$$ \{\vec{x} \in \mathbb{R}^3 \mid f(\vec{x}) = 0\} $$
[G]

More generally, an isosurface at isovalue $v$ is defined by:
$$ f(\vec{x}) - v = 0 $$
[G]

**Function Properties:**
1. **Mapping:** The function maps any point in 3D space to a real number: $\forall \vec{x} \in \mathbb{R}^3, f(\vec{x}) \in \mathbb{R}$ [G].
2. **Filter/Kernel:** The function $f$ acts as a filter or form [G].
3. **Support:** The function can be defined globally over the entire domain or as piecewise kernels (local support) [S1].
4. **Sign Convention:** While the zero set defines the surface, the sign of $f(\vec{x})$ often carries semantic meaning [G].

**Evaluation:**
The geometry is not stored as a discrete structure but is computed on-demand [G]. For volume rendering, this means evaluating $f(\vec{x})$ at sample points along rays or within voxels [G].

## Subtopics

### [Metaball Field Evaluation](1017-metaball-field-evaluation.md)
Metaball Field Evaluation generates and updates a 3D scalar density volume representing dynamic metaballs (spheres) using a CPU-GPU pipeline, typically via compute or fragment shaders. It solves the problem of converting discrete, moving atomic positions into a continuous volumetric representation suitable for surface extraction or raymarching. Unlike [Implicit Field Composition](2652-implicit-field-composition.md), which composes general implicit primitives, this method specifically sums inverse-distance or Gaussian contributions from moving spheres to create organic, merging blob shapes.

### [Distance to Hypertextures](1555-distance-to-hypertextures.md)
Distance to Hypertextures is a technique for adding relief or details to surfaces without increasing geometry cost, using procedural texture synthesis evaluated within a volumetric region around the object rather than only on the surface. It deforms implicit surfaces by adding procedural noise to the function's range instead of the domain, enabling sphere tracing with distance bounds. Unlike standard hypertexturing which may deform the domain, this variant affects the range, allowing the use of signed distance functions (SDFs) for efficient rendering.

### [Superelliptic Blends](2051-superelliptic-blends.md)
Superelliptic blends (also known as pseudonorm blends) are a local geometric deformation technique for implicitly defining the union of two or more signed distance functions (SDFs). Unlike global field additions, this method computes a $p$-norm distance to the blended union, allowing for smooth transitions between surfaces controlled by a tightness parameter. It is distinct from standard additive metaballs because it operates on the distance values directly rather than scalar field intensities, and it differs from soft metaballs by using a specific $L_p$-norm formulation.

### [Implicit Field Composition](2652-implicit-field-composition.md)
Implicit Field Composition constructs a complex scalar field by combining multiple primitive fields (such as terrain, spheres, and cuboids) using set-theoretic operations like union, intersection, and difference. It solves the problem of defining complex, multi-material, or hybrid geometries without explicit mesh topology, serving as the primary input generator for dual contouring and ray-marching pipelines. Unlike [Metaball Field Evaluation](1017-metaball-field-evaluation.md), which uses smooth blending functions (e.g., sum of inverse squared distances) to create organic morphing shapes, Implicit Field Composition relies on sharp, non-differentiable Boolean operators (min/max) to preserve hard edges and distinct material boundaries.

### [Soft Metablobby SDF](2753-soft-metablobby-sdf.md)
Soft Metablobby SDF is a procedural implicit surface method that approximates Gaussian blobby fields using piecewise cubic polynomials to avoid expensive exponentiation and iterative root finding. It defines a smooth, blended volume from a set of key points, where the surface is the zero-crossing of the summed field. Unlike standard metaballs that use Gaussian kernels, this method localizes the influence of each point to a finite radius, providing a bounded support that simplifies culling and distance estimation.

### [Implicit Surface Definition](2798-implicit-surface-definition.md)
Implicit Surface Definition provides a mathematical framework for representing 3D geometry as the zero-level set of a scalar field function $f(\vec{x})$. It solves the problem of defining complex, continuous surfaces without explicit topological data structures (like meshes) by evaluating a function at arbitrary points in space. What sets it apart from explicit surface definitions (e.g., polygonal meshes) is that the geometry is defined analytically rather than discretely, allowing for infinite resolution and smooth blending capabilities inherent to the function's algebraic or transcendental nature.

### [Half-Space Field Generation](4006-half-space-field-generation.md)
Half-Space Field Generation creates a binary volume representation by assigning a value of 1 to voxels within a defined region (typically the fluid domain) and 0 to those outside, effectively masking simulation or rendering computations. It differs from continuous implicit primitives by discretizing the domain boundary into a hard binary mask rather than evaluating a smooth scalar field for isosurface extraction. This method is primarily used to restrict physical simulation (such as pressure solving or force application) to active regions, ignoring empty space like air.

### [Implicit Geometry Sampling](6685-implicit-geometry-sampling.md)
Implicit Geometry Sampling is a technique for generating volume data from mathematical implicit functions, providing value and gradient at any point in space without relying on discrete voxel grids. It solves the problem of defining complex, continuous geometry for volume rendering by evaluating a scalar field function $f(\vec{x})$ on-demand. Unlike sibling methods that may rely on precomputed grids or specific primitive compositions, this method provides an abstract interface for sampling arbitrary implicit geometries, returning position, scalar value, and gradient. It builds on [Implicit Surface Primitives](1800-implicit-surface-primitives.md) by implementing the core evaluation mechanism for specific algebraic and transcendental functions.

### [Specific Implicit Primitives](8499-specific-implicit-primitives.md)
Specific Implicit Primitives are concrete scalar field functions $f(\vec{x})$ that define geometric shapes by evaluating a mathematical expression at query points, serving as the atomic building blocks for procedural volume rendering. Unlike generic implicit surface definitions, these primitives provide closed-form analytical solutions (or precise approximations) for specific topologies, enabling efficient ray marching and voxel generation without discrete mesh storage. They are distinguished from sibling methods like [Metaball Field Evaluation](1017-metaball-field-evaluation.md) by their single-object focus and lack of blending logic, and from [Implicit Field Composition](2652-implicit-field-composition.md) by their atomic nature rather than hierarchical combination.

### [Metaball-based Envelope Painting](8803-metaball-based-envelope-painting.md)
Metaball-based Envelope Painting is an interaction model for defining a Volume of Interest (VOI) or Region of Interest (ROI) within unsegmented volumetric data by "painting" an implicit envelope around features of interest. It solves the problem of intuitive, context-preserving volume exploration by replacing complex transfer function tuning with a direct spatial marking metaphor. Unlike techniques using independent surface-bound brushes or fixed-depth peeling regions, this method uses a higher-level shape representation (Metaballs) to create organic, merging envelopes that allow for real-time previewing of the visualization effect before final deposition.

#### Compact Polynomial Kernel
- **Definition**: A kernel family used to define skeleton-based implicit surfaces, characterized by local support required for efficient field evaluation on large skeletons and efficient closed-form evaluation of the integral field equation for line segment primitives with linearly interpolated prescribed thickness [S1].
- **Formula**: The support of an individual primitive is a sphere-cone defined by scaling the two primitive endpoints' radius by a factor $\sigma$ [S1].
- **Parameters**:
    - Degree parameter $i$: A parameter defining the polynomial degree of the kernel [S1].
    - Scale parameter $\sigma$: A parameter scaling the primitive endpoints' radius to define the support of an individual primitive [S1].

#### Implicit Function Evaluation
- **Definition**: Generates voxel coordinates defined by an implicit function [S2, unofficial].
- **Parameters**:
    - $f$: function of form $f(x, y, z)$ returning a numeric value [S2, unofficial].
    - `lower`, `upper`: range of values considered inside the object. Default: `[-Inf, 0]`. Values `<= 0` are inside, `> 0` are outside [S2, unofficial].
    - `scale`: extra scaling factor applied to coordinates before calling function [S2, unofficial].
    - `nx`, `ny`, `nz`: dimensions of the volume within which the function will be evaluated. Default: `nx=51, ny=nx, nz=nx` [S2, unofficial].
- **Layout**: Returns a data.frame of coordinates [S2, unofficial].

## Background
Implicit surfaces have been a fundamental concept in computer graphics since the introduction of metaballs by Wyvill et al. in the late 1980s, which popularized the use of scalar fields for organic shape modeling [G]. The mathematical foundation relies on level set theory, where surfaces are defined as the zero-contour of a higher-dimensional function [G]. In the context of volume rendering and procedural generation, implicit primitives allow for the creation of complex geometry through the composition of simple analytic functions, avoiding the memory overhead of mesh storage [G]. The specific kernel families, such as Cauchy, Inverse, and Compact Polynomial kernels, were developed to address the computational costs of evaluating integral surfaces on skeletal structures, offering local support for efficient culling and closed-form solutions for line segments [S1].

## Sources
[S1] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
[S2] isocubes: Voxel Data Visualization with Isometric Cubes, https://doi.org/10.32614/cran.package.isocubes, fetched 2026-10-26 (unofficial)
