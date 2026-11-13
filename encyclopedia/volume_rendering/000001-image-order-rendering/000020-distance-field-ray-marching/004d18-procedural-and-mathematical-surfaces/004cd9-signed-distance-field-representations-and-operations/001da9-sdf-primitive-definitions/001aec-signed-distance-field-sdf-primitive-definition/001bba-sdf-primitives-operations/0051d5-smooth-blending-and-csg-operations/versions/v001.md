# Smooth Blending and CSG Operations

Also called: Smooth Minimum, SDF Boolean Operations, Constructive Solid Geometry with SDFs

## Overview
This family covers the construction of complex geometry from primitives using Constructive Solid Geometry (CSG) operations (union, intersection, subtraction) and the smoothing of these operations to create organic transitions. It solves the problem of creating non-polygonal, continuous surfaces from analytical distance functions. It is distinguished from [SDF Domain Manipulation Operators](1730-sdf-domain-manipulation-operators.md) by focusing on boolean composition rather than spatial transformation, and from [Basic Primitive Definitions](5346-basic-primitive-definitions.md) by combining multiple primitives into a single field.

## Shared principles
The core mechanism relies on the property that the signed distance field (SDF) of a union of shapes is the minimum of their individual SDFs, and the intersection is the maximum.

**Union:**
The SDF for the union of two shapes $S_1$ and $S_2$ is:
$$ S_{union}(\mathbf{x}) = \min(S_1(\mathbf{x}), S_2(\mathbf{x})) $$

**Intersection:**
The SDF for the intersection of two shapes $S_1$ and $S_2$ is:
$$ S_{intersect}(\mathbf{x}) = \max(S_1(\mathbf{x}), S_2(\mathbf{x})) $$

**Subtraction:**
Subtracting shape $S_2$ from $S_1$ is equivalent to intersecting $S_1$ with the inverse of $S_2$ (where the sign is flipped):
$$ S_{subtraction}(\mathbf{x}) = \max(S_1(\mathbf{x}), -S_2(\mathbf{x})) $$

These operations allow for the construction of arbitrary complex geometries from a list of primitive edits [S1, unofficial].

## Subtopics

### Smooth Minimum Function
A function used to blend primitive shapes over a fixed radius, allowing for smoothed addition and subtraction in CSG operations [S1, unofficial].

**Definition**
The standard smooth minimum function has an infinite range, reaching zero at infinity. However, for real-time applications like the game *Dreams*, a modified formula is used that blends over a fixed radius [S1, unofficial].

**Formula**
The specific formula is not provided in the sources [S1, unofficial].

**Parameter**
*   **Blend radius:** A specified radius over which shapes blend. This replaces the infinite range of the standard smooth minimum [S1, unofficial].

**Usage**
Used in *Dreams* for creating intuitive sculpting tools for user-generated content, allowing edits to be union-ed or subtracted with smooth transitions [S1, unofficial].

### SDF Transformations and Operations
This topic covers the historical context and general application of SDF operations.

**History**
Inigo Quilez produced the first results recognizable as modern SDF in 2007, demonstrating techniques for efficient soft shadows, smooth blending, and domain repetition [S2, unofficial].

### Composite SDF via Union and Subtraction
This topic details the implementation of complex composite structures, specifically using bubbles on a panel.

**Definition**
The SDF for bubbles is defined as the union of $N$ ellipsoids. Since ellipsoid SDFs lack a simple closed form, a lower bound of the signed distance field is used. Each bubble $i$ has position $\mathbf{x}_i$, semi-axis lengths $\mathbf{r}_i$, and orientation angles $\theta_i$ in the $\{X, Z\}$ plane and $\phi_i$ in the $\{Y, Z\}$ plane [S3].

**Formula**
The full bubble field SDF is the union of all $N$ individual distance functions:
$$ S_{bubbles}(\mathbf{x}) = \bigcup_{i=1}^N B_i(\mathbf{x} - \mathbf{x}_i, \mathbf{r}_i, \theta_i, \phi_i) $$
where the union operator $\bigcup$ is defined by $\bigcup(S_1, S_2) = \min(S_1, S_2)$ [S3].

The bubbles are subtracted from the displaced panel using the intersection operator $\bigcap$, defined by $\bigcap(S_1, S_2) = \max(S_1, -S_2)$. The final distance function is:
$$ S_{final} = \bigcap(S_{panel}, S_{bubbles}) $$
[S3]

**Layout**
To manage a high number of bubbles efficiently, a regular voxel grid is defined inside the panel. Each voxel contains a maximum of one bubble to prevent overlapping. This allows $S_{bubbles}$ to be interpreted as a single SDF and traversed using a fast voxel traversal algorithm [S3].

## Background
Signed Distance Fields have a natural affinity for Constructive Solid Geometry (CSG), where solid 3D shapes can be constructed through unions and subtractions between primitive shapes [S1, unofficial]. In 2007, Inigo Quilez demonstrated techniques for smooth blending and domain repetition, marking the emergence of modern SDF techniques [S2, unofficial]. In the game *Dreams*, SDFs are used to create simple and intuitive sculpting tools for user-generated content. Objects, called 'Things', are constructed from lists of 'edits' (primitive shapes described by analytical SDFs). Edits are applied in a linear list, supporting binary union, subtraction, and smoothed operations [S1, unofficial].

## Sources
[S1] Real-time Rendering and Construction of Signed Distance Fields, https://jambuttenshaw.github.io/devportfolio/Dissertation.pdf, fetched 2026-10-28 (unofficial)
[S2] Raymarching Distance Fields, https://iquilezles.org/articles/raymarchingdf/, fetched 2026-10-27 (unofficial)
[S3] Parabolic Sphere Tracing of Signed Distance Fields for Old Glass Modelling and Rendering, https://doi.org/10.1109/tvcg.2025.3613853, fetched 2026-11-05
