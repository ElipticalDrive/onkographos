# Volumetric Snapping

Also called: Delaunay Cover Construction, Snapping, Watertight Triangulation of Point Clouds

## Overview
Volumetric Snapping is an algorithm that constructs an interpolating triangular mesh from a closed point cloud of arbitrary genus by voxelizing an intermediate Delaunay cover, triangulating the voxel surface, and snapping vertices to the nearest point in the cloud [S1, unofficial]. It solves the problem of generating a watertight, manifold mesh from unstructured point data by using a voxel grid as a topological scaffold [S1, unofficial]. What sets it apart from direct surface reconstruction methods (like Poisson Reconstruction) is its reliance on a boolean voxelization of a Delaunay-based barrier, ensuring topological correctness (watertightness) before geometric approximation [S1, unofficial]. It differs from pure marching cubes by using the point cloud itself to define the geometry via snapping, rather than an implicit scalar field [S1, unofficial].

## How it works
**Inputs:** A closed point cloud $P = \{p_1, \dots, p_N\}$ and a voxel grid size $h$ [S1, unofficial].
**Outputs:** A valid, manifold triangular mesh interpolating the point cloud [S1, unofficial].

**Data Structures:**
1. **Point Cloud:** List of 3D coordinates [S1, unofficial].
2. **Delaunay Cover:** A set of triangles $T_{DC}$ forming a barrier between inside and outside regions [S1, unofficial].
3. **Voxel Grid:** A 3D boolean array or sparse structure indicating "black" (intersecting cover) and "white" (empty) voxels [S1, unofficial].
4. **Mesh:** List of triangles defined by vertex coordinates [S1, unofficial].

**Parameters:**
* $h$: Voxel grid size. Defined as the minimal voxel edge length if voxels are rectangular. Determines the level of detail [S1, unofficial].
* $k$: Number of nearest neighbors for constructing the Delaunay cover. Typical value: $k \in [15, 25]$ [S1, unofficial].

**Steps:**

1. **Construct the Delaunay Cover:**
   For each point $p_i \in P$:
   a. Find the $k$ nearest neighbors of $p_i$ [S1, unofficial].
   b. Approximate the tangent plane at $p_i$ using the least squares plane through these $k$ neighbors [S1, unofficial].
   c. Project the $k$ nearest neighbors onto this tangent plane [S1, unofficial].
   d. Compute the Local Planar Delaunay Triangulation of these projected points [S1, unofficial].
   e. Map the resulting triangles back to 3D space [S1, unofficial].
   f. The Delaunay Fan $DFan(p_i)$ is the set of these triangles sharing $p_i$ [S1, unofficial].
   g. The Delaunay Cover $DC(P)$ is the union of all $DFan(p_i)$ for all $p_i \in P$ [S1, unofficial].
   *Note:* In 3D, $DC(P)$ is neither a consistent triangulation nor does it have correctly oriented triangle normals [S1, unofficial].

2. **Refine the Delaunay Cover:**
   To ensure proper voxelization, the triangles in $DC(P)$ must be sampled such that the distance between sample points is smaller than $h$ [S1, unofficial].
   a. Iterate through all triangles in $DC(P)$ [S1, unofficial].
   b. If any side of a triangle is longer than $h$, subdivide it using a 1-to-4 split (inserting midpoints of edges) [S1, unofficial].
   c. Repeat until all triangle sides are $\le h$ [S1, unofficial].

3. **Voxelize the Delaunay Cover:**
   Create a regular voxel grid. A voxel is colored "black" if it intersects the refined Delaunay cover, and "white" otherwise [S1, unofficial].
   *Implementation Note:* Instead of explicit triangle-voxel intersection tests, the sources suggest using the refined sample points to create the voxelization straightforwardly, or using an efficient scan-conversion method [S1, unofficial].

4. **Triangulate the Voxel Surface:**
   Extract the exterior surface of the black voxels. The sources describe using a 2D tracing algorithm that processes the voxel grid slice by slice (requiring $n_x + n_y + n_z$ slices) [S1, unofficial]. This yields a watertight triangulation of the voxel surface [S1, unofficial].

5. **Snap to Point Cloud:**
   For each vertex $v$ on the triangulated exterior voxel surface:
   a. Find the closest point $p_{\text{nearest}} \in P$ to $v$ [S1, unofficial].
   b. Replace $v$ with $p_{\text{nearest}}$ [S1, unofficial].
   *Note:* Snapping is not injective; multiple grid vertices may snap to the same point, or adjacent vertices may snap to the same point, creating degenerate and double triangles [S1, unofficial].

6. **Clean-up Triangulation:**
   Process the snapped mesh to ensure it is a topologically manifold triangulation. Perform the following removals in order [S1, unofficial]:
   a. **Remove degenerate triangles:** Triangles with at least two coinciding vertices [S1, unofficial].
   b. **Remove multiple triangles:** Triangles having the exact same set of vertices [S1, unofficial].
   c. **Remove dangling triangles:** Triangles that do not have three neighboring triangles [S1, unofficial].
   d. **Remove non-manifold edges:** Edges shared by more than two triangles. Collapse them by snapping one end of the edge to the other [S1, unofficial].
   e. **Re-clean:** After edge collapse, re-execute steps a-c to remove new degenerate, double, and dangling triangles introduced by the collapse [S1, unofficial].

7. **Optimization (Optional):**
   Required only if the voxel grid is coarse relative to the inter-point distance [S1, unofficial].
   a. Iterate between Laplacian smoothing and re-snapping (1-2 iterations suffice) [S1, unofficial].
   b. **Laplacian Smoothing:** Replace each vertex with the centroid of its neighbors [S1, unofficial].
   c. **Edge Swapping:** Optionally swap edges in the 1-ring neighborhoods of vertices with valency 4 to improve diffusion, but only if no new valency 4 vertices are created [S1, unofficial].
   d. **Re-snap:** Snap the smoothed vertices back to the nearest points in $P$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volumetric_snapping(point_cloud, h, k):
    # Step 1: Construct Delaunay Cover
    delaunay_cover = []
    for p in point_cloud:
        neighbors = get_k_nearest_neighbors(p, point_cloud, k)
        tangent_plane = fit_least_squares_plane(neighbors)
        projected_neighbors = project_to_plane(neighbors, tangent_plane)
        local_triangles = delaunay_triangulation_2d(projected_neighbors)
        for tri in local_triangles:
            delaunay_cover.append(map_to_3d(tri, tangent_plane, p))

    # Step 2: Refine Delaunay Cover
    refined_cover = []
    for tri in delaunay_cover:
        while max_edge_length(tri) > h:
            refined_cover.extend(subdivide_1_to_4(tri))
            # Remove old tri from list to avoid infinite loop
            refined_cover.remove(tri)
    delaunay_cover = refined_cover

    # Step 3: Voxelize
    voxel_grid = create_voxel_grid(delaunay_cover, h)

    # Step 4: Triangulate Surface
    surface_triangles = trace_voxel_surface(voxel_grid)

    # Step 5: Snap
    snapped_mesh = []
    for tri in surface_triangles:
        new_vertices = []
        for v in tri.vertices:
            closest_p = get_nearest_neighbor(v, point_cloud)
            new_vertices.append(closest_p)
        snapped_mesh.append(Triangle(new_vertices))

    # Step 6: Clean-up
    clean_mesh = clean_manifold(snapped_mesh)

    # Step 7: Optimization (if coarse)
    if is_grid_coarse(h, point_cloud):
        for _ in range(2):
            clean_mesh = laplacian_smooth(clean_mesh)
            for tri in clean_mesh:
                for i in range(len(tri.vertices)):
                    tri.vertices[i] = get_nearest_neighbor(tri.vertices[i], point_cloud)
            clean_mesh = clean_manifold(clean_mesh)

    return clean_mesh

# Helper definitions
def get_k_nearest_neighbors(p, cloud, k):
    distances = [dist(p, q) for q in cloud]
    indices = sorted(range(len(distances)), key=lambda i: distances[i])
    return [cloud[i] for i in indices[:k]]

def fit_least_squares_plane(points):
    centroid = sum(points) / len(points)
    centered = [p - centroid for p in points]
    cov = sum([outer(c, c) for c in centered])
    eigenvalues, eigenvectors = eig(cov)
    return (centroid, eigenvectors[:, 0])

def project_to_plane(points, plane):
    centroid, normal = plane
    return [p - dot(p - centroid, normal) * normal for p in points]

def delaunay_triangulation_2d(points):
    return compute_2d_delaunay(points)

def map_to_3d(tri_2d, plane, p_i):
    centroid, normal = plane
    return [centroid + v_2d for v_2d in tri_2d]

def max_edge_length(tri):
    return max(dist(tri[0], tri[1]), dist(tri[1], tri[2]), dist(tri[2], tri[0]))

def subdivide_1_to_4(tri):
    m0 = (tri[0] + tri[1]) / 2
    m1 = (tri[1] + tri[2]) / 2
    m2 = (tri[2] + tri[0]) / 2
    return [Triangle([tri[0], m0, m2]), Triangle([m0, tri[1], m1]), Triangle([m2, m1, tri[2]]), Triangle([m0, m1, m2])]

def create_voxel_grid(refined_cover, h):
    return mark_voxels(refined_cover, h)

def trace_voxel_surface(voxel_grid):
    return extract_boundary(voxel_grid)

def is_grid_coarse(h, point_cloud):
    avg_dist = compute_average_neighbor_distance(point_cloud)
    return h > avg_dist

def laplacian_smooth(mesh):
    new_vertices = {}
    for tri in mesh:
        for v in tri.vertices:
            if v not in new_vertices:
                neighbors = get_vertex_neighbors(v, mesh)
                new_vertices[v] = sum(neighbors) / len(neighbors)
    return replace_vertices(mesh, new_vertices)

def clean_manifold(mesh):
    mesh = [t for t in mesh if not has_coincident_vertices(t)]
    mesh = remove_duplicate_triangles(mesh)
    mesh = remove_dangling_triangles(mesh)
    mesh = collapse_non_manifold_edges(mesh)
    mesh = [t for t in mesh if not has_coincident_vertices(t)]
    mesh = remove_duplicate_triangles(mesh)
    mesh = remove_dangling_triangles(mesh)
    return mesh

def remove_duplicate_triangles(mesh):
    seen = set()
    unique = []
    for t in mesh:
        key = tuple(sorted(t.vertices))
        if key not in seen:
            seen.add(key)
            unique.append(t)
    return unique

def remove_dangling_triangles(mesh):
    return [t for t in mesh if count_shared_edges(t, mesh) == 3]

def collapse_non_manifold_edges(mesh):
    return collapse_edges(mesh)
```

## Subtopics

### Delaunay Triangulation Condition for Snapping
A sufficient condition for snapping a grid to obtain a Delaunay triangulation, based on the Voronoi diagram of the sample points [S1, unofficial]. It defines the distance between two Voronoi cells as the shortest distance between any two points lying on the boundary of these cells [S1, unofficial]. The condition requires that the grid diagonal (longest triangle side) be smaller than the distance between any two non-neighboring Voronoi cells [S1, unofficial]. Under this condition, there are only three essential configurations yielding non-degenerate triangles: 1) Voronoi vertex inside grid cell, 2) Voronoi vertex in neighboring cell, 3) Voronoi vertex outside grid cell (both gray triangles snap to same Delaunay triangle) [S1, unofficial].

## Operating characteristics
**Time Complexity:** Not explicitly given as a function of $N$ or $h$, but the contouring step consumes 70% of the time for the Moose 3K model [S1, unofficial]. The Delaunay cover construction takes 31% of the time for the Cycladic Head model [S1, unofficial].
**Space Complexity:** $O(n^2)$ worst case, where $n$ is the number of voxels in one dimension [S1, unofficial]. This is achieved by using a 2D tracing algorithm requiring $n_x + n_y + n_z$ slices, whereas a 3D flooding algorithm would require $O((n_x + n_y + n_z)^3)$ storage [S1, unofficial].
**Measured Performance:** On a Pentium 4 2.4Ghz with 512MB RAM, the Moose 3K model took 4.8s total [S1, unofficial]. The Cycladic Head model took 76s total [S1, unofficial].
**Strengths:** Produces watertight, manifold meshes; handles arbitrary genus; high grid resolutions are possible due to the Delaunay cover and space-efficient contouring [S1, unofficial].
**Limitations:** Cannot be applied to point clouds with holes unless the holes are aligned with the boundaries of the bounding box [S1, unofficial]. Snapping is not injective, requiring a clean-up step [S1, unofficial]. Coarse grids produce overlapping triangles or miss original points [S1, unofficial].

## Background
The method was proposed in a paper presented at GRAPP 2007 (International Conference on Computer Graphics Theory and Applications) by authors from CMA/IFI, University of Oslo, and Celestijnenlaan, KU Leuven [S1, unofficial]. It addresses the need for mesh representation from 3D laser scanner data, which produces huge amounts of point data [S1, unofficial]. The algorithm builds on the concept of shrink-wrapping and uses the Delaunay cover to ensure topological correctness [S1, unofficial].

## Sources
[S1] Volumetric Snapping: Watertight Triangulation of Point Clouds, https://www.scitepress.org/PublishedPapers/2007/20798/pdf/index.html, fetched 2026-10-27 (unofficial)
