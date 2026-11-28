# Fast Convex Polygon Wrapper (FCPW) Accelerator

Also called: Mesh Handler Implementation, MeshQueryHandler

## Overview
The Fast Convex Polygon Wrapper (FCPW) is an acceleration structure and query interface designed for efficient geometric operations on triangle meshes, specifically ray-tetrahedron edge intersections and closest-point projections. It serves as the core geometric backend for the `MeshQueryHandler`, enabling fast spatial queries required by subgrid marching algorithms. Unlike general-purpose BVHs, FCPW is specialized for handling convex polygonal primitives (triangles) within a mesh context, optimizing for the specific access patterns of tetrahedral cell intersection tests.

## How it works
The FCPW system operates in two phases: construction and querying. It relies on a preprocessed representation of the mesh geometry.

**Inputs and Outputs:**
*   **Construction Input:** A list of polygon indices (`polygons`) mapping to vertex positions (`positions`).
*   **Query Input:** A ray (origin $P$, direction $L$) or a query point $q$.
*   **Query Output:** Intersection parameters ($t$), surface normals, or closest points.

**Data Structures:**
The source code indicates the existence of an accelerator object returned by `get_fcpw_accel`. The exact internal layout (e.g., BVH depth, node size, or specific spatial hashing scheme) is not explicitly detailed in the provided snippets, but it is constructed from `TriangleSoup` data (vertices and faces). The `MeshQueryHandler` stores this accelerator instance.

**Algorithm Steps:**

1.  **Acceleration Structure Construction:**
    Initialize the accelerator by processing the mesh geometry.
    ```python
    # Pseudocode based on [S1, unofficial]
    def build_fcpw_accelerator(positions, polygons):
        """
        Constructs the FCPW accelerator from mesh data.
        positions: List of 3D vertex coordinates.
        polygons: List of triangle indices (e.g., [v0, v1, v2]).
        Returns: An accelerator object 'accel'.
        """
        # The specific spatial partitioning or bounding volume hierarchy
        # construction logic is encapsulated in get_fcpw_accel.
        # [G] Typically involves building bounding volumes for each triangle
        # and organizing them into a tree structure for culling.
        accel = get_fcpw_accel(polygons, positions)
        return accel
    ```

2.  **Ray-Tetrahedron Edge Intersection:**
    To find intersections between a ray and the edges of a tetrahedron (represented as a set of triangles/polygons), the system uses `find_single_tet_edge_intersections_fcpw`.
    ```python
    def query_intersections(tet_positions, accel, use_robust=True, record_normals=True):
        """
        Finds intersections of a ray with tetrahedron edges using FCPW.
        tet_positions: Vertex positions of the tetrahedron.
        accel: The FCPW accelerator.
        use_robust: Boolean flag for robust numerical handling.
        record_normals: Boolean flag to compute surface normals.
        Returns: Arrays for intersection times and normals.
        """
        # The ray is implicitly defined by the context of the caller
        # (likely derived from the ray-tetrahedron intersection method).
        # This function delegates the heavy lifting to the FCPW structure.
        times, normals = find_single_tet_edge_intersections_fcpw(
            tet_positions,
            accel,
            use_robust=use_robust,
            record_normals=record_normals
        )
        return times, normals
    ```

3.  **Closest Point Query:**
    For proximity queries, `closest_point_fcpw` finds the point on the mesh surface closest to a query point $q$.
    ```python
    def query_normal(q, accel):
        """
        Finds the closest point on the mesh to query point q and its normal.
        q: The 3D query point.
        accel: The FCPW accelerator.
        Returns: closest_point (3D vector), normal (3D vector).
        """
        closest_point, normal = closest_point_fcpw(q, accel)
        return closest_point, normal
    ```

**Parameters:**
*   `useRobust`: A boolean flag (default `True` in some contexts) that enables numerically robust intersection tests, likely handling degenerate cases or precision issues [S1, unofficial].
*   `recordNormals`: A boolean flag determining whether surface normals are computed and returned during intersection queries [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Geometry:** The `useRobust` flag suggests that standard floating-point comparisons may fail for coplanar or nearly coplanar triangles, requiring epsilon-based tolerances or exact arithmetic [S1, unofficial].
*   **Mesh Topology:** The method assumes a valid `TriangleSoup` representation. Invalid indices or non-manifold geometry may lead to undefined behavior if not preprocessed [S1, unofficial].

## Subtopics

### [Convex Polygon Falloff Function](10023-convex-polygon-falloff-function.md)
A monotonic falloff function for convex polygons, mapping distance to a value between 0 and 1. Used for smooth blending or weighting in surface queries.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. The use of an "accelerator" implies sub-linear query time ($O(\log N)$ or similar) relative to the number of triangles, but the specific complexity class is not defined.
*   **Memory Complexity:** Not given in the sources. Depends on the internal structure of the FCPW accelerator (e.g., BVH node count).
*   **Strengths:** Specialized for mesh queries; integrates seamlessly with the `MeshQueryHandler` pipeline; supports robust intersection testing.
*   **Limitations:** Specific to convex polygonal meshes; performance characteristics are not quantified in the provided text.
*   **When to use:** When performing ray-triangle or ray-tetrahedron edge intersections within a subgrid marching framework where mesh queries are frequent.
*   **When not to use:** For non-mesh primitives or when exact analytical solutions without acceleration structures are preferred for small datasets.

## Background
The Fast Convex Polygon Wrapper (FCPW) is part of the `subgrid-marching` project by H. Baktash [S1, unofficial]. It is implemented within the `MeshQueryHandler` class, which serves as an alternative to CGAL-based query handlers when CGAL is not available. The method is designed to support the generation of subgrid meshes from implicit fields or point clouds, requiring efficient geometric queries for surface reconstruction and tetrahedralization.

## Sources
[S1] hbaktash/subgrid-marching, https://github.com/hbaktash/subgrid-marching, fetched 2026-10-27 (unofficial)
