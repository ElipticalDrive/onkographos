# Signed Distance Field Representations and Operations

## Overview
This family covers the mathematical definitions, spatial transformations, and sampling mechanisms required to define and query scalar distance fields for ray marching. It solves the problem of representing complex 3D geometry and procedural surfaces as continuous scalar functions $f(\mathbf{p})$, where the zero-level set $f(\mathbf{p}) = 0$ defines the surface [G]. What sets this family apart from its siblings is its focus on the **definition and retrieval** of the distance value itself, distinct from [Terrain Raymarching](1357-terrain-raymarching.md) which handles height-field specific optimizations, [Neon Raymarcher](4157-neon-raymarcher.md) which focuses on volumetric lighting integration, [Menger Sponge Raymarching](4875-menger-sponge-raymarching.md) which implements specific fractal recursion, and [Fractals and Mathematical Surfaces](10024-fractals-and-mathematical-surfaces.md) which deals with iterative complex number mappings [G].

## Shared principles
The family operates on the principle that a scene is defined by a scalar field $f(\mathbf{p})$ where the zero-level set $f(\mathbf{p}) = 0$ represents the surface [G]. For sphere tracing to be valid, the function must satisfy the Lipschitz condition with constant 1, ensuring $|f(\mathbf{p}_1) - f(\mathbf{p}_2)| \le \|\mathbf{p}_1 - \mathbf{p}_2\|$ [G]. This guarantees that the value $f(\mathbf{p})$ is a safe lower bound on the distance to the surface, allowing the ray marcher to step forward by exactly $f(\mathbf{p})$ without tunneling [G].

Composition is achieved through Boolean operations on the distance values:
*   **Union**: $f_{union}(\mathbf{p}) = \min(f_1(\mathbf{p}), f_2(\mathbf{p}))$ [G].
*   **Intersection**: $f_{intersect}(\mathbf{p}) = \max(f_1(\mathbf{p}), f_2(\mathbf{p}))$ [G].
*   **Subtraction**: $f_{sub}(\mathbf{p}) = \max(-f_1(\mathbf{p}), f_2(\mathbf{p}))$ [G].

Transformations are applied by mapping the query point $\mathbf{p}$ into the local coordinate space of the primitive using the inverse transformation matrix $M^{-1}$ before evaluation [G]. Procedural noise (e.g., FBM) is often used to perturb these analytical surfaces, creating hypertextures where the isosurface is defined by $SDF(\mathbf{p}) + B_n(\mathbf{p}) = 0$ [G].

## Subtopics

### [SDF Primitive Definitions](1578-sdf-primitive-definitions.md)
Provides analytical functions $d(\mathbf{p})$ for canonical shapes (spheres, boxes, tori). It focuses on **value-domain** definitions for specific geometric primitives, distinct from domain manipulation or Boolean composition. Choose this for exact metric representations of basic geometry.

### [Distance Field Representations and Sampling](8725-distance-field-representations-and-sampling.md)
Covers the mechanics of retrieving scalar distance values from discrete, procedural, or specialized data structures. It distinguishes itself by focusing on *retrieval* and *sampling* (memory layouts, boundary handling) rather than field construction. Choose this when integrating SDFs with grid-based or texture-based storage.

### [SDF Transformations](11027-sdf-transformations.md)
Modifies the spatial domain by applying geometric transformations (tiling, twisting, animation) to the query point $\mathbf{p}$ before evaluation. It enables complex procedural geometry without altering primitive definitions. Choose this for dynamic instancing, repetition, and deformation effects.

### [Global Distance Query via Local-Global Blending](14434-global-distance-query-via-local-global-blending.md)
Constructs a feature-preserving global signed distance field by blending a local patch-based representation with an ordinary global signed distance field. It solves the problem where local representations are accurate near the surface but fail far away, while global fields often smooth out sharp features. Choose this when needing both local precision and global validity.

## Background
Not given in the sources.

## Sources
