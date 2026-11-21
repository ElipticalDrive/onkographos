# Implicit Surface Primitives

Also called: Implicit Fields, Scalar Field Primitives

## Overview
Implicit Surface Primitives define 3D geometry as the zero-level set of a scalar field function $f: \mathbb{R}^3 \to \mathbb{R}$, solving the problem of representing complex, continuous shapes without explicit vertex or polygon data [S2, unofficial]. Unlike explicit meshes, geometry is defined by the *value* of a function rather than a list of points, allowing for seamless composition of shapes via algebraic operations and infinite resolution at any scale [S2, unofficial]. This family is distinguished from procedural noise methods by its reliance on geometric distance or density metrics rather than stochastic texture generation, and from chunk generation by its analytic, on-demand evaluation nature.

## Shared principles
All methods in this family rely on the evaluation of a scalar field function $f$ over a domain $\Omega \subseteq \mathbb{R}^3$ [S2, unofficial]. The core mechanism is the identification of the surface as the locus of points where the function output equals a specific constant, typically zero [S1, S2, unofficial].

**Mathematical Definition:**
The implicit surface is defined as the set of points $\vec{x}$ satisfying:
$$ \{\vec{x} \in \mathbb{R}^3 \mid f(\vec{x}) = 0\} $$
[S2, unofficial]

More generally, an isosurface at isovalue $v$ is defined by:
$$ f(\vec{x}) - v = 0 $$
[S2, unofficial]

**Function Properties:**
1.  **Mapping:** The function maps any point in N-dimensional space (specifically 3D for volume rendering) to a real number:
    $$ \forall \vec{x} \in \mathbb{R}^3, f(\vec{x}) \in \mathbb{R} $$
    [S1]
2.  **Filter/Kernel:** The function $f$ acts as a filter or form. It can be algebraic (polynomials) or transcendental (trigonometric, exponential) [S2, unofficial].
3.  **Support:** The function can be defined globally over the entire domain or as piecewise kernels (local support) [S2, unofficial].
4.  **Sign Convention:** While the zero set defines the surface, the sign of $f(\vec{x})$ often carries semantic meaning. In Signed Distance Functions (SDFs), $f(\vec{x}) > 0$ indicates outside, $f(\vec{x}) < 0$ indicates inside, and $f(\vec{x}) = 0$ is on the surface [S1]. For general implicit fields, the sign may indicate different material regions or be irrelevant if only the zero-crossing is used [S1, S2, unofficial].

**Evaluation:**
The geometry is not stored as a discrete structure but is computed on-demand. For volume rendering, this means evaluating $f(\vec{x})$ at sample points along rays or within voxels [Implicit Surface Primitives](1800-implicit-surface-primitives.md).

## Subtopics

### [Metaball Field Evaluation](1017-metaball-field-evaluation.md)
Metaball Field Evaluation is a CPU-GPU pipeline that generates and updates a 3D density volume representing dynamic metaballs using compute shaders, often accompanied by an optional flag volume for empty space skipping. It solves the problem of evaluating a dynamic scalar field composed of moving spheres to calculate density at query points $(x,y,z)$ for subsequent volume rendering or isosurface extraction. Unlike static implicit primitives, this method explicitly handles temporal animation of ball positions and powers, optimizing the field update stage for parallel dispatch across voxel grids.

### [Distance to Hypertextures](1555-distance-to-hypertextures.md)
Distance to Hypertextures is a technique for adding geometric relief and detail to surfaces without increasing polygon count by evaluating procedural noise within a volumetric region around the object rather than solely on the surface. It deforms implicit surfaces by adding procedural noise to the function's range instead of the domain, enabling the use of sphere tracing with safe distance bounds. This method bridges the gap between surface textures and volumetric phenomena, allowing for the modeling of complex, organic details.

### [Superelliptic Blends](2051-superelliptic-blends.md)
Superelliptic blends (also known as pseudonorm blends) are a local blending technique for implicit surfaces that computes the $p$-norm distance to the blended union of signed distance functions (SDFs). Unlike global blending methods, it operates only within a local domain defined by the support of the input fields. It is distinguished by its reliance on the Ricci formulation of implicit surfaces, where surfaces are defined by non-negative functions equal to one, and its characteristic gradient discontinuities at the boundary of the blend domain.

### [Implicit Field Composition](2652-implicit-field-composition.md)
Implicit Field Composition constructs a complex signed distance field (SDF) by combining multiple primitive fields (such as terrain, cuboids, and spheres) using set-theoretic operations. It solves the problem of defining intricate, multi-component geometry for dual contouring or ray marching without explicit mesh storage. It differs from [Metaball Field Evaluation](1017-metaball-field-evaluation.md) by using hard boolean operations (max/min) rather than smooth blending, and from [Superelliptic Blends](2051-superelliptic-blends.md) by relying on exact SDF primitives rather than approximate smooth unions.

### [Soft Metablobby SDF](2753-soft-metablobby-sdf.md)
Soft Metablobby SDF defines an implicit surface by blending contributions from a set of key points using a piecewise cubic polynomial that approximates a Gaussian distribution. It differs from standard Metaball Field Evaluation by using a localized cubic support rather than an exponential decay, which avoids expensive exponentiation and iterative root finding while still producing smooth, organic blends.

### [Implicit Surface Definition](2798-implicit-surface-definition.md)
Implicit surface definition represents 3D geometry as the zero-level set of a scalar field function $f: \mathbb{R}^3 \to \mathbb{R}$. It solves the problem of defining complex, continuous shapes without explicit vertex or polygon data by evaluating a mathematical expression at query points. What sets it apart from explicit surface definitions (like meshes) is that the geometry is defined by the *value* of a function rather than a list of points, allowing for seamless composition of shapes via algebraic operations and infinite resolution at any scale.

### [Half-Space Field Generation](4006-half-space-field-generation.md)
Half-Space Field Generation defines a volumetric domain by partitioning space into two disjoint regions (typically "inside" and "outside") using a binary scalar field or a level set threshold. It solves the problem of restricting simulation or rendering computations to a specific geometric subset, effectively ignoring empty space or non-fluid regions. Unlike smooth implicit primitives that blend continuously, this method produces a sharp discontinuity, making it distinct from [Metaball Field Evaluation](1017-metaball-field-evaluation.md) and [Soft Metablobby SDF](2753-soft-metablobby-sdf.md). It is primarily used to mask out irrelevant voxels in fluid simulations or ray marching to save bandwidth and compute time.

### [Implicit Geometry Sampling](6685-implicit-geometry-sampling.md)
Implicit Geometry Sampling is a technique for generating volume data from mathematical implicit functions, providing scalar value and gradient at any point in space without relying on discrete voxel grids for the definition. It serves as the evaluation engine for procedural and implicit volumes, allowing for analytic or procedural definition, local or global support, and composition via set operations or blending [Implicit Surface Definition](2798-implicit-surface-definition.md). Unlike discrete volume methods, it computes values on-the-fly, optimizing for GPU parallelism or ray marching step sizes [Implicit Surface Definition](2798-implicit-surface-definition.md).

### [Specific Implicit Primitives](8499-specific-implicit-primitives.md)
Specific Implicit Primitives provides concrete implementations of standard geometric shapes (spheres, boxes, cylinders) as implicit functions, serving as the atomic building blocks for higher-level composition methods. It differs from general field composition by providing optimized, exact distance calculations for canonical primitives rather than procedural or blended approximations.

### [Metaball-based Envelope Painting](8803-metaball-based-envelope-painting.md)
Metaball-based Envelope Painting is an interactive visualization technique that defines a Volume of Interest (VOI) by "painting" an implicit envelope around features in unsegmented volumetric data. It distinguishes itself from sibling methods like [Metaball Field Evaluation](1017-metaball-field-evaluation.md) by integrating the field definition directly into a real-time interaction loop that supports previewing the envelope's effect before commitment, using a "thick paint" metaphor to merge organic shapes. Unlike [Distance to Hypertextures](1555-distance-to-hypertextures.md) or [Superelliptic Blends](2051-superelliptic-blends.md) which focus on static field composition, this method prioritizes user-guided exploration and editing through two distinct painting modes (3D Surface and 2D Screen) combined with MIDA visualization.

#### Compact Polynomial Kernel
- **Definition**: A kernel family used to define skeleton-based implicit surfaces, characterized by local support required for efficient field evaluation on large skeletons and efficient closed-form evaluation of the integral field equation for line segment primitives with linearly interpolated prescribed thickness [S1].
- **Formula**: The support of an individual primitive is a sphere-cone defined by scaling the two primitive endpoints' radius by a factor $\sigma$ [S1].
- **Parameters**:
    - Degree parameter $i$: A parameter defining the polynomial degree of the kernel [S1].
    - Scale parameter $\sigma$: A parameter scaling the primitive endpoints' radius to define the support of an individual primitive [S1].

## Background
Implicit surfaces have been a fundamental concept in computer graphics since the introduction of metaballs by Wyvill et al. in the late 1980s, which popularized the use of scalar fields for organic shape modeling. The mathematical foundation relies on level set theory, where surfaces are defined as the zero-contour of a higher-dimensional function. In the context of volume rendering and procedural generation, implicit primitives allow for the creation of complex geometry through the composition of simple analytic functions, avoiding the memory overhead of mesh storage. The specific kernel families, such as Cauchy, Inverse, and Compact Polynomial kernels, were developed to address the computational costs of evaluating integral surfaces on skeletal structures, offering local support for efficient culling and closed-form solutions for line segments [S1].

## Sources
[S1] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
