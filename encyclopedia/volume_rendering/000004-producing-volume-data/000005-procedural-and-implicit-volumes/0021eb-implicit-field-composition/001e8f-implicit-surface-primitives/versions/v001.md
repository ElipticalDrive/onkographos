# Implicit Surface Primitives

Also called: Implicit primitives, SDF primitives, procedural surfaces

## Overview
Implicit Surface Primitives define 3D geometry as the zero-level set (isosurface) of a scalar field function $f(\mathbf{x})$, where the surface is the locus of points where $f(\mathbf{x}) = 0$. This family solves the problem of representing complex, smooth, or dynamic geometry without polygonal meshes, enabling operations like boolean blending, smooth deformation, and efficient ray marching. It is distinguished from polygonal representations by its continuity and from procedural noise volumes by its focus on sharp or smoothly blended boundaries rather than volumetric density gradients.

## Shared principles
All methods in this family evaluate a scalar field $f: \mathbb{R}^3 \to \mathbb{R}$ at a query point $\mathbf{x}$. The sign of $f(\mathbf{x})$ typically indicates spatial position: negative for inside, positive for outside, and zero on the surface. Key shared characteristics include:
1.  **Analytic or Procedural Definition:** The field is defined by mathematical formulas (algebraic, transcendental, or piecewise) rather than sampled data.
2.  **Local or Global Support:** Some primitives (e.g., SDFs) have global support or specific domain restrictions, while others (e.g., kernel-based) have local support for efficiency.
3.  **Composition:** Primitives are often combined using set operations (union, intersection, difference) or blending techniques (e.g., smooth min, superelliptic blends) to create complex shapes.
4.  **Evaluation:** The core operation is a point-wise evaluation, often optimized for GPU parallelism or ray marching step sizes.

## Subtopics

### [Metaball Field Evaluation](1017-metaball-field-evaluation.md)
A procedural method for generating and updating a 3D scalar density volume representing dynamic metaballs (spheres) using compute or fragment shaders. It solves the problem of efficiently computing the scalar field values for a large number of moving volumetric primitives on the GPU, often serving as input for subsequent raycasting or marching cubes algorithms. It distinguishes itself from static implicit surface definitions by supporting real-time animation of blob positions and strengths, and from simple sphere blending by using an inverse-square distance falloff with a threshold offset rather than a smooth Gaussian blend.

### [Torus Implicit Function](1322-torus-implicit-function.md)
The Torus Implicit Function computes the signed distance from a query point to a torus surface defined by a major radius and a minor radius. It solves the problem of evaluating a smooth, continuous scalar field for ray marching and volume rendering of toroidal geometry without polygonal approximation. It is distinguished from [Metaball Field Evaluation](1017-metaball-field-evaluation.md) by providing a single, non-blending primitive, and from [Superelliptic Blends](2051-superelliptic-blends.md) by its specific geometric derivation for a ring shape rather than a generalized superquadric.

### [Superelliptic Blends](2051-superelliptic-blends.md)
Superelliptic blends (also known as pseudonorm blends) are a local blending technique for implicit surfaces that computes the $p$-norm distance to the blended union of signed distance functions (SDFs). Unlike global blending methods, this approach operates only within the overlapping support of the constituent functions, defined by the parameter $\epsilon$. It is distinguished by its computational simplicity and compatibility with sphere tracing, though it introduces gradient discontinuities at the boundary of the blend domain.

### [Soft Metablobby SDF](2753-soft-metablobby-sdf.md)
A soft implicit surface consisting of n key points $p_i$ with radii $R_i$ and threshold $T$, approximating a gaussian distribution.

### [Implicit Surface Definition](2798-implicit-surface-definition.md)
Implicit surfaces are level surfaces of a chosen filter $f$, which can be algebraic or transcendental, global or piecewise kernels.

### [Half-Space Field Generation](4006-half-space-field-generation.md)

### Gyroid Surface Evaluation
- **Definition:** Evaluates the scalar field for a Triply Periodic Minimal Surface (TPMS) Gyroid at point $(x,y,z)$ [S1, unofficial].
- **Formula:** $f(x,y,z) = \sin(sx)\cos(sy) + \sin(sy)\cos(sz) + \sin(sz)\cos(sx)$, where $sx = x \cdot scale, sy = y \cdot scale, sz = z \cdot scale$ [S1, unofficial].
- **Parameter:** `scale`: The scaling factor for the gyroid coordinates. Default value is 4.5 [S1, unofficial].

### Thin Shell Implicit Function
- **Definition:** Implicit function for a thin spherical shell: $f(p) = |x^2+y^2+z^2 - r^2| - \epsilon$ [S2, unofficial].
- **Parameter:** `radius`: Sphere radius for the shell, default 1.0 [S2, unofficial].
- **Parameter:** `epsilon`: Half-thickness of the shell, default 1e-3 [S2, unofficial].

### Compact Polynomial Kernel
- **Definition:** A kernel family used to define skeleton-based implicit surfaces, characterized by local support required for efficient field evaluation on large skeletons and efficient closed-form evaluation of the integral field equation for line segment primitives with linearly interpolated prescribed thickness [S3].
- **Formula:** The support of an individual primitive is a sphere-cone defined by scaling the two primitive endpoints' radius by a factor $\sigma$ [S3].
- **Parameter:** `Degree parameter i`: A parameter defining the polynomial degree of the kernel [S3].
- **Parameter:** `Scale parameter σ`: A parameter scaling the primitive endpoints' radius to define the support of an individual primitive [S3].

## Background
Implicit surfaces were introduced by Wyvill et al. in the context of "blobby" objects for computer graphics [G]. The mathematical foundation relies on level set methods and signed distance functions (SDFs). The use of SDFs for ray marching was popularized by Inigo Quilez and others in the demoscene and real-time graphics community [S1, unofficial]. The theoretical underpinnings of blending and kernel-based implicit surfaces are detailed in academic literature on integral surfaces and TPMS [S3].

## Sources
[S1] nff747/marching-cubes-wgsl, https://github.com/nff747/marching-cubes-wgsl, fetched 2026-10-27 (unofficial)
[S2] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S3] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
