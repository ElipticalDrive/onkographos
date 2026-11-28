# Plane-Cube Intersection
Also called: Plane-Cube Intersection Volume, Volume from a plane slicing a cube

## Overview
This method computes the volume of the irregular polygon formed by the intersection of a plane and a cube [S1, unofficial]. It is specifically designed for scenarios where a planar boundary slices through a volumetric cell, such as determining the fraction of a cell that is partially ionized or covered by a burning front in computational astrophysics [S1, unofficial]. It differs from standard ray-cube intersection by outputting the resulting volume rather than just intersection points, and extends the approach of Salama & Kolb 2005 by explicitly constructing an ordered list of face vertices to facilitate this volume calculation [S1, unofficial].

## How it works
**Inputs:**
- A unit cube defined by 8 vertices $\mathbf{V}_1$ through $\mathbf{V}_8$, enumerated and ordered as specified by the source [S1, unofficial].
- A plane parameterized by spherical angles $\theta$ and $\phi$ and the normal ray length $dl$ [S1, unofficial].

**Outputs:**
- The volume of the intersection polygon.
- The set of intersection points on the cube edges.

**Data Structures:**
- Vertices $\mathbf{V}_i$: 3D Cartesian coordinates $(x, y, z)$ [S1, unofficial].
- Edges $E_{i \rightarrow j}$: Defined by pairs of vertex indices $(i, j)$ [S1, unofficial].
- Intersection points: 3D coordinates derived from the edge intersection logic.

**Steps:**

1. **Define the Plane Normal:**
   Compute the Cartesian coordinates of the normal ray $\mathbf{n}$ using spherical angles $\theta$ and $\phi$ and length $dl$:
   $$ \mathbf{n} = [dl \cos(\theta), \ dl \sin(\theta)\sin(\phi), \ dl \cos(\phi) ] $$
   Derive the unit normal vector $\mathbf{\hat{n}} = \mathbf{n} / \|\mathbf{n}\|$ [S1, unofficial].

2. **Define the Plane Equation:**
   The plane equation is given by:
   $$ dl = \mathbf{\hat{n}} \cdot \mathbf{x} $$
   where $\mathbf{x}$ represents the Cartesian coordinates of any point lying on the plane [S1, unofficial].

3. **Iterate Over Cube Edges:**
   For each edge $E_{i \rightarrow j}$ connecting vertex $\mathbf{V}_i$ to $\mathbf{V}_j$:
   a. Define the edge line parametrically:
      $$ E_{i \rightarrow j} = \mathbf{V}_i + \lambda (\mathbf{V}_j - \mathbf{V}_i) $$
   b. Calculate the intersection parameter $\lambda$:
      $$ \lambda = \frac{ dl - \mathbf{\hat{n}} \cdot \mathbf{V}_i }{\mathbf{\hat{n}} \cdot (\mathbf{V}_j - \mathbf{V}_i ) } $$
   c. Check validity: An intersection exists on the segment if and only if $0 \le \lambda \le 1$ [S1, unofficial].
   d. If valid, compute the intersection point $\mathbf{P} = \mathbf{V}_i + \lambda (\mathbf{V}_j - \mathbf{V}_i)$.

4. **Construct the Polygon:**
   Collect all valid intersection points. The resulting shape is an inscribed irregular polygon with 3 to 6 vertices [S1, unofficial].

5. **Compute Volume:**
   Construct an ordered list of the face vertices of the resulting polygon. Use this ordered list to compute the volume of the irregular polygon [S1, unofficial]. The specific algorithm for ordering vertices and calculating the volume from the list is not detailed in the source [S1, unofficial].

**Parameters:**
- $\theta$: Spherical angle, $0 \le \theta \le 2\pi$ [S1, unofficial].
- $\phi$: Spherical angle, $0 \le \phi \le \pi$ [S1, unofficial].
- $dl$: Length of the normal ray (distance from origin) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Degenerate Intersections:** If $\mathbf{\hat{n}} \cdot (\mathbf{V}_j - \mathbf{V}_i) = 0$, the edge is parallel to the plane. The source does not specify handling for this case, but numerically it requires a guard to avoid division by zero [G].
- **Vertex Ordering:** The volume calculation depends on an "ordered list of face vertices." Incorrect ordering will yield incorrect volumes or self-intersecting meshes. The source implies a specific enumeration of the cube vertices $\mathbf{V}_1$ to $\mathbf{V}_8$ is required, but does not provide the coordinate mapping [S1, unofficial].
- **Polygon Complexity:** The number of vertices of the intersecting polygon can vary between 3 and 6, showing "wild patterns" as $\theta$ and $\phi$ change, even if the volume changes smoothly [S1, unofficial].

## Subtopics

### Cube-Plane Edge Intersection
Computes intersection points between a plane and the 12 edges of a unit cube [S2, unofficial]. Iterates over the 12 edges and computes the parameter $t$ using the plane equation $Ax + By + Cz + D = 0$ [S2, unofficial]. If $0 < t < 2$, the intersection point is computed and added to the result [S2, unofficial]. The parameter is calculated as $t = -\frac{A x_0 + B y_0 + C z_0 + D}{A (x_1 - x_0) + B (y_1 - y_0) + C (z_1 - z_0)}$ [S2, unofficial]. A known pitfall is that the condition $t>0 \ \&\& \ t<2$ allows intersections slightly outside the unit cube since the edge length is 2 in this parameterization [S2, unofficial].

## Operating characteristics
**Time Complexity:** Constant time $O(1)$ relative to scene size, as it operates on a single cube with a fixed number of edges (12 edges for a cube) [S1, unofficial].
**Memory Complexity:** Constant space $O(1)$ to store vertices and intersection points [S1, unofficial].
**Strengths:** Provides an exact analytical volume for planar cuts through cubic cells, useful for adaptive mesh refinement or astrophysical simulations involving partial cell coverage [S1, unofficial].
**Limitations:** Requires careful handling of vertex ordering for volume integration. The complexity of the intersection polygon topology (3 to 6 vertices) may require case-specific logic for robust mesh generation [S1, unofficial].
**When to use:** When a planar boundary intersects a cubic volume element and the exact volume of the intersection is required [S1, unofficial].
**When not to use:** For non-planar cuts or non-cubic primitives [S1, unofficial].

## Background
The method was introduced by F.X. Timmes in the context of computational astrophysics, specifically for calculating the fraction of a cell that may be partially ionized or covered by a burning front [S1, unofficial]. It extends the ideas presented by Salama & Kolb in 2005 [S1, unofficial]. The tool `cube_and_plane.tbz` is provided as part of a collection of codes by Timmes [S1, unofficial].

## Sources
[S1] volume of intersection between a plane and a cube - cococubed, https://cococubed.com/code_pages/raybox.shtml, fetched 2026-10-29 (unofficial)
[S2] nevermoe/SmokeSimulation, https://github.com/nevermoe/SmokeSimulation, fetched 2026-10-26 (unofficial)
