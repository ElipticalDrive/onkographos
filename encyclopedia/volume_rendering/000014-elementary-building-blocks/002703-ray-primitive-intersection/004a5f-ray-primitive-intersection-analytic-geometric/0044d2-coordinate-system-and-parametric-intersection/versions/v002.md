# Coordinate System and Parametric Intersection
## Overview
Coordinate System and Parametric Intersection provides geometric primitives for determining ray intersections within structured voxel grids and for computing approximate intersections with complex volumetric structures [G]. It is distinguished from sibling methods like [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md) by its specific handling of spherical grid topology and from [Exact Rational Ray-Surfel Evaluation](2915-exact-rational-ray-surfel-evaluation.md) by its reliance on angular decomposition and scan-plane approximations [G].

## Shared principles

## Subtopics

### [Azimuthal Hit Detection](3885-azimuthal-hit-detection.md)
Determines whether a ray intersects an azimuthal section within a spherical voxel grid [G]. An azimuthal section is defined as a surface segment living in the XZ plane, bounded by radial and polar extents [G]. This method serves as the core primitive intersection test for the azimuthal dimension, complementing radial and polar hit detection [G]. It is distinguished from [Ray-Polar Section Intersection](3886-ray-polar-section-intersection.md) by operating on the XZ plane geometry rather than the XY plane, and from [Ray-Ray Intersection](7624-ray-ray-intersection.md) by specifically handling the curved, sector-based boundaries of spherical coordinates rather than straight line segments [G]. Choose it when traversing spherical voxel grids requiring azimuthal boundary checks. See [Azimuthal Hit Detection](3885-azimuthal-hit-detection.md).

### [Ray-Polar Section Intersection](3886-ray-polar-section-intersection.md)
Determines whether a ray intersects a polar section, which is a volumetric primitive defined within the XY plane of a spherical voxel grid [G]. It serves as the specific implementation for polar coordinates within the broader Ray-Primitive Intersection framework [G]. It distinguishes itself from [Azimuthal Hit Detection](3885-azimuthal-hit-detection.md) by operating on the Y-axis plane rather than the Z-axis, and from [Ray-Ray Intersection](7624-ray-ray-intersection.md) by handling planar sector boundaries rather than linear line segments [G]. Choose it when traversing spherical voxel grids requiring polar boundary checks. See [Ray-Polar Section Intersection](3886-ray-polar-section-intersection.md).

### [Ray-Ray Intersection](7624-ray-ray-intersection.md)
Computes the least-squares intersection point of two rays in 3D space [G]. Used when rays do not intersect exactly due to noise or parallax, finding the point that minimizes the squared distance to both rays [G]. It differs from exact intersection methods by providing a robust estimate for skew lines, which is essential in volumetric reconstruction where ray origins and directions are derived from noisy camera poses [G]. Choose it when resolving non-intersecting lines in 3D space. See [Ray-Ray Intersection](7624-ray-ray-intersection.md).

### Scan Plane Beam Intersection
An approximate method to determine if a camera ray intersects a caustic beam and the location of that intersection, used because exact computation is too costly for thousands of beams [S1].
- **Definition**: An approximate method to determine if a camera ray intersects a caustic beam and the location of that intersection, used because exact computation is too costly for thousands of beams [S1].
- **Steps**:
    1. Define a scan plane, which is an auxiliary arbitrary plane containing the camera ray [S1].
    2. Compute the intersections of the scan plane and the edges of the caustic beam to form a triangle [S1].
    3. Minimize error from curved beam edges by choosing the scan plane normal vector to be close to the normal of the caustic triangle [S1].
    4. Transform the view ray to the beam's local space [S1].
    5. Select a representative point on the intersection segment $\Delta x$ (specifically the midpoint) and evaluate the area function $A(y)$ [S1].
    6. Compute the fraction of light scattered towards the viewer based on the cross-section area (determining radiance scaling) and scattering in the participating medium [S1].
- **Formula**: The scan plane normal $\mathbf{n}_p$ is computed as $\mathbf{n}_p = \mathbf{r}_{view} \times (\mathbf{r}_{view} \times \mathbf{n}_4)$, where $\mathbf{r}_{view}$ is the view ray direction and $\mathbf{n}_4$ is the caustic triangle normal [S1].
Choose it when rendering real-time volume caustics where exact ray-bilinear patch intersection is too expensive [S1].

## Background
The spherical coordinate intersection methods are part of the infrastructure for spherical voxel grid traversal, often used in global illumination and caustic rendering where spherical symmetry or specific grid structures are advantageous [G]. The Ray-Ray intersection method is a standard technique in computer graphics for finding the closest approach between skew lines, frequently used in ray tracing acceleration structures and multi-view geometry [G]. The Scan Plane Beam Intersection method originates from real-time volume caustics rendering, where exact ray-bilinear patch intersection is too expensive for the large number of caustic beams present in typical scenes [S1].

## Sources
[S1] Real-time volume caustics with adaptive beam tracing, https://doi.org/10.1145/1944745.1944753, fetched 2026-10-28
