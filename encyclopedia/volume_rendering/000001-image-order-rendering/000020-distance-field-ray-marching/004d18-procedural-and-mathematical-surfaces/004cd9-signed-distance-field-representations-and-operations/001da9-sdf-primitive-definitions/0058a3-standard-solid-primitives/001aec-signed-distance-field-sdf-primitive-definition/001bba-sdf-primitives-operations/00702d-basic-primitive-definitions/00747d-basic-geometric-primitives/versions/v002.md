# Basic Geometric Primitives

Also called: Basic SDF Primitives, Standard Solid Primitives

## Overview
Basic Geometric Primitives provide exact Euclidean signed distance functions (SDFs) for fundamental shapes, serving as the atomic building blocks for procedural geometry in distance-field ray marching. They differ from sibling primitives like the [Ellipsoid](2397-ellipsoid-signed-distance-function.md) or [Capsule](2439-capsule-signed-distance-function.md) by defining canonical, analytically simple volumes (sphere, box, plane) that form the basis for complex scene construction via boolean operations.

## Shared principles
All primitives in this family compute a scalar field $d(p)$ representing the shortest Euclidean distance from a query point $p$ to the surface of the shape. The sign of $d(p)$ indicates spatial containment:
*   $d(p) < 0$: The point $p$ is inside the volume.
*   $d(p) = 0$: The point $p$ lies exactly on the surface.
*   $d(p) > 0$: The point $p$ is outside the volume.

This signed distance property ensures that ray marching steps are safe: the step size never exceeds the distance to the surface, preventing ray overshooting and intersection errors . The functions are differentiable almost everywhere, allowing the gradient $\nabla d$ to be used as the surface normal for shading [Parent].

## Subtopics

### [Signed Box Distance Function](1092-signed-box-distance-function.md)
Computes the exact distance to an axis-aligned box. It distinguishes interior from exterior space and maintains sharp 90-degree corners, unlike rounded variants.

### [Plane Signed Distance Function](1102-plane-signed-distance-function.md)
Defines an infinite plane using a normal and offset. It is computationally trivial and serves as a ground plane or boundary for boolean operations [Parent].

### [Sphere Signed Distance Function](1237-sphere-signed-distance-function.md)
Computes the distance to a sphere surface. It is the fundamental 3D volumetric primitive, enabling efficient step-size estimation in ray marching.

## Background
Signed Distance Functions for basic primitives were popularized in computer graphics by Martin Held and later refined by Ian McEwan, Andrew Kensler, and others for real-time ray marching . The sphere SDF is among the oldest and most widely used implicit primitives due to its simplicity and rotational symmetry . The box SDF requires careful implementation to avoid underestimation in corner regions, a pitfall noted in early implementations .

## Sources
