# Signed Distance Field Representations and Operations
Also called: SDF families, Distance Field Families

## Overview
This family encompasses the fundamental mathematical definitions, spatial transformations, and sampling mechanisms required to define and query scalar distance fields for ray marching. It solves the problem of representing complex 3D geometry and procedural surfaces as continuous scalar functions $f(\mathbf{p})$, where the zero-level set $f(\mathbf{p}) = 0$ defines the surface. What sets this family apart from its siblings is its focus on the **definition and retrieval** of the distance value itself, distinct from [Terrain Raymarching](1357-terrain-raymarching.md) which handles height-field specific optimizations, [Neon Raymarcher](4157-neon-raymarcher.md) which focuses on volumetric lighting integration, [Menger Sponge Raymarching](4875-menger-sponge-raymarching.md) which implements specific fractal recursion, and [Fractals and Mathematical Surfaces](10024-fractals-and-mathematical-surfaces.md) which deals with iterative complex number mappings.

## Shared principles
The family operates on the principle that a scene is defined by a scalar field $f(\mathbf{p})$ where the zero-level set $f(\mathbf{p}) = 0$ represents the surface. For sphere tracing to be valid, the function must satisfy the Lipschitz condition with constant 1, ensuring $|f(\mathbf{p}_1) - f(\mathbf{p}_2)| \le \|\mathbf{p}_1 - \mathbf{p}_2\|$. This guarantees that the value $f(\mathbf{p})$ is a safe lower bound on the distance to the surface, allowing the ray marcher to step forward by exactly $f(\mathbf{p})$ without tunneling [S2, S5, 9643].

Composition is achieved through Boolean operations on the distance values:
*   **Union**: $f_{union}(\mathbf{p}) = \min(f_1(\mathbf{p}), f_2(\mathbf{p}))$ [S39, unofficial].
*   **Intersection**: $f_{intersect}(\mathbf{p}) = \max(f_1(\mathbf{p}), f_2(\mathbf{p}))$ [S39, unofficial].
*   **Subtraction**: $f_{sub}(\mathbf{p}) = \max(-f_1(\mathbf{p}), f_2(\mathbf{p}))$ [S39, unofficial].

Transformations are applied by mapping the query point $\mathbf{p}$ into the local coordinate space of the primitive using the inverse transformation matrix $M^{-1}$ before evaluation [S3850]. Procedural noise (e.g., FBM) is often used to perturb these analytical surfaces, creating hypertextures where the isosurface is defined by $SDF(\mathbf{p}) + B_n(\mathbf{p}) = 0$ [S1].

## Subtopics

### [SDF Primitive Definitions](1578-sdf-primitive-definitions.md)
Provides analytical functions $d(\mathbf{p})$ for canonical shapes (spheres, boxes, tori). It focuses on **value-domain** definitions for specific geometric primitives, distinct from domain manipulation or Boolean composition. Choose this for exact metric representations of basic geometry.

### [Distance Field Representations and Sampling](8725-distance-field-representations-and-sampling.md)
Covers the mechanics of retrieving scalar distance values from discrete, procedural, or specialized data structures. It distinguishes itself by focusing on *retrieval* and *sampling* (memory layouts, boundary handling) rather than field construction. Choose this when integrating SDFs with grid-based or texture-based storage.

### [SDF Transformations](11027-sdf-transformations.md)
Modifies the spatial domain by applying geometric transformations (tiling, twisting, animation) to the query point $\mathbf{p}$ before evaluation. It enables complex procedural geometry without altering primitive definitions. Choose this for dynamic instancing, repetition, and deformation effects.

## Background
The concept of Signed Distance Fields (SDFs) in computer graphics was popularized by Greg Turk's "Blobby Volumes" (1989) and later formalized for ray marching by Inigo Quilez in his online articles (2009-2010) [S1, S2, S5, 9643]. The Lipschitz continuity requirement for sphere tracing was rigorously established in the context of ray marching by Matt Pharr and Greg Humphreys in "Physically Based Rendering" (2010) and further detailed by Quilez [S2, S5, 9643]. The use of Boolean operations (min/max) for constructive solid geometry (CSG) with SDFs is a standard technique in procedural modeling, attributed to various contributors in the demoscene and real-time rendering communities [S39, unofficial].

## Sources
[S1] Inigo Quilez, "Signed Distance Functions," https://iquilezles.org/articles/distfunctions/, 2009-2023.
[S2] Inigo Quilez, "Sphere Tracing," https://iquilezles.org/articles/raymarching/, 2009-2023.
[S3] Inigo Quilez, "Transformations," https://iquilezles.org/articles/distfunctions/, 2009-2023.
[S5] Matt Pharr, Jakob Wenzel, Greg Humphreys, "Physically Based Rendering: From Theory to Implementation," 3rd ed., Morgan Kaufmann, 2016.
[S6] Inigo Quilez, "SDF Primitives," https://iquilezles.org/articles/distfunctions/, 2009-2023.
[S7] Inigo Quilez, "Box SDF," https://iquilezles.org/articles/distfunctions/, 2009-2023.
[S8] Inigo Quilez, "Torus SDF," https://iquilezles.org/articles/distfunctions/, 2009-2023.
[S14] Inigo Quilez, "Signed Distance Functions," https://iquilezles.org/articles/distfunctions/, 2009-2023.
[S15] Inigo Quilez, "Constructive Solid Geometry," https://iquilezles.org/articles/distfunctions/, 2009-2023.
[S3850] Inigo Quilez, "Transformations," https://iquilezles.org/articles/distfunctions/, 2009-2023.
[S39] Inigo Quilez, "Constructive Solid Geometry," https://iquilezles.org/articles/distfunctions/, 2009-2023.
[9643] Inigo Quilez, "Sphere Tracing," https://iquilezles.org/articles/raymarching/, 2009-2023.
