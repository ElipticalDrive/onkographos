# Ray-Plane/Rectangle Intersection
Also called: Ray-Plane Intersection, Ray-Rectangle Intersection, Ray-Square Intersection

## Overview
This family provides closed-form analytical methods to determine the intersection of a ray with planar primitives (infinite planes, rectangles, squares) and related shapes (spheres, cylinders, cubes) by reducing the problem to solving linear or quadratic equations. It differs from [Implicit Surface and SDF Intersection](2795-implicit-surface-and-sdf-intersection.md) by avoiding iterative root-finding, and from [Ray-Triangle Intersection](11312-ray-triangle-intersection.md) by handling axis-aligned or general rectangular domains often used as bounding volumes or ground planes in volume rendering.

## Shared principles
All methods in this family rely on the parametric representation of a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the origin, $\mathbf{d}$ is the direction, and $t \ge 0$ is the distance along the ray.

1.  **Implicit Equation**: The primitive is defined by an implicit function $f(\mathbf{x}) = 0$. For a plane with normal $\mathbf{n}$ and point $\mathbf{p}_0$, $f(\mathbf{x}) = \mathbf{n} \cdot (\mathbf{x} - \mathbf{p}_0)$. For a sphere centered at $\mathbf{c}$ with radius $r$, $f(\mathbf{x}) = \|\mathbf{x} - \mathbf{c}\|^2 - r^2$.
2.  **Substitution**: Substitute $\mathbf{r}(t)$ into $f(\mathbf{x}) = 0$.
    *   For planes, this yields a linear equation $At + B = 0$.
    *   For spheres and cylinders (curved parts), this yields a quadratic equation $At^2 + Bt + C = 0$.
3.  **Solving**:
    *   Linear: $t = -B/A$.
    *   Quadratic: $t = \frac{-B \pm \sqrt{B^2 - 4AC}}{2A}$.
4.  **Validation**:
    *   **Distance**: Check if $t$ lies within the valid ray segment $[t_{min}, t_{max}]$ (typically $t_{min}=0$).
    *   **Domain**: For bounded primitives (rectangles, spheres, cylinders), verify that the intersection point $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ lies within the primitive's bounds (e.g., barycentric coordinates for triangles, radius checks for cylinders, coordinate ranges for axis-aligned rectangles).
5.  **Output**: Return the smallest valid $t$, the intersection point $\mathbf{p}$, and the surface normal $\mathbf{n}$ (computed as $\nabla f(\mathbf{p})$ normalized).

## Subtopics

### [Ray-Sphere Intersection](3539-ray-sphere-intersection.md)
Computes entry and exit distances for a ray intersecting a sphere. It uses a closed-form quadratic solution derived from the sphere's implicit equation. Unlike [Ray-Triangle Intersection](11312-ray-triangle-intersection.md) which requires barycentric checks, or [Implicit Surface and SDF Intersection](2795-implicit-surface-and-sdf-intersection.md) which uses iteration, this method is purely analytical. Choose it for spherical primitives or bounding volumes.

### [Ray-Cylinder Intersection](5014-ray-cylinder-intersection.md)
Computes entry and exit distances for a cylinder defined by radius, height, and rectangular cross-section faces. It differs from [Ray-box intersection](776-ray-box-intersection.md) by handling the curved lateral surface via quadratic solving and cylindrical coordinate checks. Choose it for cylindrical structures or pipe-like volumes.

### [Infinite Plane Ray Intersection](5021-infinite-plane-ray-intersection.md)
Computes the intersection of a ray with an infinite plane defined by a point and a normal. It is typically used as a ground plane kept separate from the scene's BVH for ray intersection testing. Choose it for unbounded planar boundaries or shadow planes.

### [Axis-Aligned Rectangle Hit Test](5279-axis-aligned-rectangle-hit-test.md)
Tests intersection with an axis-aligned rectangle. It is a specialized case of ray-plane intersection where the domain check is simplified to coordinate comparisons. Choose it for fast culling against axis-aligned bounding boxes (AABBs) or screen-space rectangles.

### [Plane-Cube Intersection](11468-plane-cube-intersection.md)
Determines the volume of the intersection between a plane and a cube, resulting in an irregular polyhedron. It extends the method of Salama & Kolb (2005) by constructing an ordered list of face vertices. Choose it when computing the exact volume of a clipped voxel or cube.

#### Ray-Square Intersection
**Definition**: A ray-squared plane intersection test that computes intersection with a square primitive defined by a corner, two edge vectors $\mathbf{v}_1, \mathbf{v}_2$, and normal [S1, unofficial].
**Steps**:
1. Compute the intersection point $\mathbf{p} = \mathbf{o} + \mathbf{d} t_{hit}$ [S1, unofficial].
2. Check if $\mathbf{p}$ is inside the square using barycentric-like coordinates:
   - $\mathbf{w} = \text{normalize}(\mathbf{v}_1 \times \mathbf{v}_2)$
   - $\mathbf{planar\_hitpt\_vector} = \mathbf{p} - \text{corner}$
   - $\alpha = \mathbf{w} \cdot (\mathbf{planar\_hitpt\_vector} \times \mathbf{v}_2)$
   - $\beta = \mathbf{w} \cdot (\mathbf{v}_1 \times \mathbf{planar\_hitpt\_vector})$
   - Return false if $\alpha \le 0$ or $\alpha \ge 1$ or $\beta \le 0$ or $\beta \ge 1$ [S1, unofficial].

## Background
Ray-primitive intersection is a fundamental building block in ray tracing and volume rendering, originating from the early days of computer graphics (Whitted, 1980). The analytical solutions for planes and spheres are standard in all graphics literature. The specific implementations for cylinders and cubes in volume rendering contexts often appear in specialized volumetric path tracing repositories [S1, unofficial]. The plane-cube volume intersection method is based on work by Salama & Kolb (2005) for voxel clipping [S1, unofficial].

## Sources
[S1] ainagutierrez01/Volumetric-Path-Tracing-, https://github.com/ainagutierrez01/Volumetric-Path-Tracing-, fetched 2026-10-26 (unofficial)
