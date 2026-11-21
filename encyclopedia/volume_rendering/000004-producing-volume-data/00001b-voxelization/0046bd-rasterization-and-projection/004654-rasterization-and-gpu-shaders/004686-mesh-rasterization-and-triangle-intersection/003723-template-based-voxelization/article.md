# Template-based Voxelization

Also called: DGtal::MeshVoxelizer, MeshVoxelizer

## Overview
Template-based Voxelization computes the digitization of a continuous triangle or mesh by determining if a voxel belongs to the digitization based on whether a geometric template centered at the voxel intersects the triangle [S1, S3]. It converts a continuous surface mesh into a discrete set of voxels (DigitalSet) [S2, unofficial]. This method is distinguished from sibling methods like AABB-triangle intersection by its reliance on topological separation properties (6- or 26-connectivity) to define the boundary, ensuring that the resulting digital surface separates the interior from the exterior in the specified connectivity sense [S1, S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A mesh (collection of vertices and faces) or a single triangle defined by vertices $A, B, C$. The mesh may contain non-triangular faces.
*   **Input:** A `DigitalSet` defining the output domain (a 3D hyperrectangular grid) and the connectivity type (6 or 26) [S1, S3].
*   **Input:** A `scaleFactor` (double, default 1.0) applied to vertex coordinates before voxelization [S1, S3].
*   **Output:** A `DigitalSet` containing the indices of voxels that intersect the mesh geometry [S2, unofficial].

**Data Structures:**
*   **DigitalSet:** A container modeling `concepts::CDigitalSet` with a 3D space dimension [S1, S3]. It stores the set of voxel indices belonging to the digitization.
*   **Domain:** A `HyperRectDomain` defined by a minimum point (e.g., $(0,0,0)$) and a maximum point (e.g., $(128,128,128)$) [S2, unofficial].
*   **IntersectionTarget:** A type derived from `IntersectionTargetTrait<Space, Separation, 1>::Type` [S3]. This template defines the geometric shape (the "template") used to test for intersection. The shape depends on the `Separation` parameter (6 or 26) [S1, S3].
*   **TriangleOrientation:** An enum (`TRIANGLE_OUTSIDE`, `TRIANGLE_INSIDE`, `TRIANGLE_ONEDGE`, `TRIANGLE_ONVERTEX`) used by the 2D containment test [S1, S3].

**Parameters:**
*   **Separation:** Integer constant, must be 6 or 26 [S1, S3].
    *   **6:** Corresponds to 6-connectivity (face-adjacent voxels). The digitization of a Euclidean plane ensures no 6-path exists from one side to the other [S1, S3].
    *   **26:** Corresponds to 26-connectivity (vertex/edge/face-adjacent voxels) [S1, S3].
*   **scaleFactor:** Double value applied to mesh vertex coordinates. Default is 1.0 [S1, S3].

**Algorithm Steps:**
1.  **Initialization:** Instantiate the `MeshVoxelizer` template class with the `DigitalSet` type and the chosen `Separation` connectivity (6 or 26) [S2, unofficial].
2.  **Preprocessing (Mesh):** If the input mesh contains non-triangular faces, naively triangulate them using a triangle fan centered at the first vertex (index 0) [S1, S3].
    *   *Pitfall:* If a face is not convex, this triangulation strategy produces incorrect digitization results [S1, S3].
3.  **Scaling:** Apply the `scaleFactor` to the coordinates of all mesh vertices [S3].
4.  **Iteration:** For each triangle (or triangulated face) in the mesh:
    a.  Extract vertices $A, B, C$ and compute the bounding box `bbox` of the triangle in voxel space [S3].
    b.  Call `voxelizeTriangle` with $A, B, C$, the normal $n$, and `bbox` [S3].
5.  **Voxelization (Triangle):** Inside `voxelizeTriangle`:
    a.  Iterate over all voxel indices $v$ within the `bbox` [S3].
    b.  Construct the template centered at voxel $v$ based on the `Separation` parameter [S1, S3].
    c.  Test if the template intersects the triangle [S3].
    d.  If intersection is true, insert voxel $v$ into `outputSet` [S3].
    e.  *Boundary Check:* If voxel $v$ is outside the domain of `outputSet`, skip it [S1, S3].
6.  **Completion:** Return the populated `outputSet` [S2, unofficial].

**Core Formula:**
A voxel $v$ belongs to the digitization if and only if the template $T_v$ centered at $v$ intersects the triangle $T$ [S1, S3]:
$$ v \in \text{Digitization} \iff T_v \cap T \neq \emptyset $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class MeshVoxelizer:
    def __init__(self, DigitalSet, Separation):
        # Separation must be 6 or 26
        assert Separation in [6, 26]
        self.Separation = Separation
        self.IntersectionTarget = IntersectionTargetTrait(DigitalSet.Space, Separation, 1)

    def voxelize(self, outputSet, aMesh, scaleFactor=1.0):
        # Preprocess mesh: triangulate non-triangular faces using fan at index 0
        triangles = []
        for face in aMesh.faces:
            if len(face) == 3:
                triangles.append(face)
            else:
                # Naive triangulation fan at vertex 0
                v0 = face[0]
                for i in range(1, len(face) - 1):
                    triangles.append([v0, face[i], face[i+1]])
        
        # Scale vertices
        scaled_vertices = [v * scaleFactor for v in aMesh.vertices]
        
        for tri_indices in triangles:
            A = scaled_vertices[tri_indices[0]]
            B = scaled_vertices[tri_indices[1]]
            C = scaled_vertices[tri_indices[2]]
            
            # Compute bounding box in voxel space
            bbox = compute_bounding_box(A, B, C)
            
            self.voxelize_triangle(outputSet, A, B, C, bbox)

    def voxelize_triangle(self, outputSet, A, B, C, bbox):
        # Iterate voxels in bounding box
        for v in bbox.voxels():
            # Check if voxel is within outputSet domain
            if not outputSet.domain.contains(v):
                continue
            
            # Construct template centered at v
            template = self.IntersectionTarget.template_at(v)
            
            # Test intersection
            if self.IntersectionTarget.intersects(template, Triangle(A, B, C)):
                outputSet.insert(v)

    def point_is_inside_2d_triangle(self, A, B, C, p):
        # Uses OrientationFunctor: InHalfPlaneBySimple3x3Matrix
        # Returns TRIANGLE_OUTSIDE, TRIANGLE_INSIDE, TRIANGLE_ONEDGE, or TRIANGLE_ONVERTEX
        pass
```

**Subtopics:**
*   **6-Separating Digitization:** Uses 6-connectivity templates. Ensures no 6-path crosses the digitized plane. Choose when strict topological separation in face-adjacent connectivity is required [S1, S3].
*   **26-Separating Digitization:** Uses 26-connectivity templates. Allows for thicker boundaries or different topological properties. Choose when 26-connectivity background separation is needed [S1, S3].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of triangles and the volume of their bounding boxes. For each triangle, the algorithm iterates over all voxels in the bounding box and performs a template intersection test [S1, S3].
*   **Memory Complexity:** Proportional to the number of voxels in the output `DigitalSet` [S2, unofficial].
*   **Performance:** CPU straightforward implementation [S1, S3]. Performance depends heavily on the resolution (scaleFactor) and the surface area of the mesh relative to the voxel grid [S2, unofficial].
*   **Strengths:** Guarantees topological separation properties (6- or 26-separability) [S1, S3]. Simple to implement [S1].
*   **Limitations:** Incorrect results for non-convex faces due to naive triangulation [S1, S3]. Skips voxels outside the defined domain [S1, S3].
*   **When to use:** When topological correctness (separability) is more important than speed or handling complex non-convex geometry directly.
*   **When not to use:** For meshes with many non-convex faces, or when high-performance GPU implementation is required (this is a CPU method) [S1, S3].

## Background
Template-based Voxelization is a CPU straightforward implementation of the method described in reference [78] (also cited as [79] in some versions) [S1, S3]. It is part of the DGtal library, a C++ library for geometric modeling and digital geometry analysis [S1, S3]. The method relies on the concept of "separability" in digital topology, where the choice of connectivity (6 or 26) determines the topological properties of the digitized surface [S1, S3].

## Sources
[S1] MeshVoxelizer Aim: A class for computing the digitization of a triangle or a Mesh., https://www.dgtal.org/doc/1.4/classDGtal_1_1MeshVoxelizer.html, fetched 2026-10-26
[S2] shapes/exampleMeshVoxelizer.cpp, https://www.dgtal.org/doc/1.4/shapes_2exampleMeshVoxelizer_8cpp-example.html, fetched 2026-10-26 (unofficial)
[S3] MeshVoxelizer Aim: A class for computing the digitization of a triangle or a Mesh., https://dgtal-team.github.io/doc-pr/pr1697/classDGtal_1_1MeshVoxelizer.html, fetched 2026-10-26
