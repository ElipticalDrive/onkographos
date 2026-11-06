# Raytracing Cell-Organized Irregular Volumes

Also called: Cell-based volume raytracing, Raytracing irregular volume data

## Overview
This method enables efficient ray traversal through irregular volumetric datasets composed of non-uniform cells (such as tetrahedra or hexahedra) by maintaining a current cell and stepping through adjacent cells via shared faces [S1]. It solves the problem of navigating complex, potentially concave geometries where standard grid-based traversal fails [S1]. Unlike [Ray-Tetrahedron Intersection](2525-ray-tetrahedron-intersection.md), which treats cells as isolated primitives, this approach explicitly manages cell connectivity to minimize redundant intersection tests [S1]. It differs from [Plücker-based Tetrahedral Cell Intersection](1767-pl-cker-based-tetrahedral-cell-intersection.md) by relying on explicit face-sharing adjacency lists rather than geometric duality tests for navigation [S1].

## How it works
**Inputs:**
- Ray origin $d$ and direction vector $c$ [S1].
- Irregular volume dataset defined by cells (e.g., tetrahedra or hexahedra) [S1].
- Adjacency data structure mapping each face to its neighboring cells [S1].
- Surface boundary information (list of exposed faces) [S1].

**Outputs:**
- Sequence of cells intersected by the ray [S1].
- Integration parameters (density, color) accumulated along the ray path within the volume [G].

**Data Structures:**
- **Cell:** Contains vertex indices and material properties [G].
- **Face:** Defined by three vertex indices $(a, b, c)$. Stores the precomputed normal vector $N = Ab \times Ac$, where $Ab = b - a$ and $Ac = c - a$ [S1]. Stores adjacency: a list of neighbor cell indices. If the list is empty, the face is an exposed boundary [S1].
- **Adjacency Map:** Hash map or array linking Face ID to Neighbor Cell IDs [G].

**Algorithm Steps:**

1. **Initial Intersection:** Intersect the ray against all exposed faces of the dataset to find the first entry point. Select the face with the smallest positive intersection parameter $t$. If no intersection is found, the ray misses the volume [S1].

2. **Cell Traversal Loop:** While the ray has not exited the dataset:
   a. **Identify Current Cell:** Determine the cell containing the current intersection point [S1].
   b. **Intersect Cell Faces:** For each of the four faces of the current tetrahedral cell (or six for hexahedra), compute the intersection parameter $t_{face}$ using the plane intersection formula [S1].
   c. **Verify Face Containment:** For each candidate face with a positive $t_{face} > t_{current}$, verify if the intersection point lies within the triangular face boundaries using barycentric coordinates [S1].
   d. **Select Exit Face:** Choose the face with the smallest valid $t_{face}$ [S1].
   e. **Check Adjacency:** If the selected face has a neighbor cell, update the current cell to the neighbor and repeat step 2. If the selected face has no neighbor (exposed face), the ray has exited the current connected component. Proceed to Step 3 [S1].

3. **Re-entry Check (Concavity Handling):** Since irregular volumes can be concave, the ray may re-enter the dataset. Intersect the ray against all exposed faces again. Find the face with the smallest positive $t_{face}$ that is greater than the current exit $t$. If found, update the current cell to the neighbor of this new entry face and resume Step 2. If no such face exists, the ray has fully exited the dataset [S1].

**Formulas:**

*Plane Intersection Parameter:*
For a ray $R(t) = d + t c$ and a plane defined by vertices $a, b, c$:
$$ t = \frac{(Ab \times Ac) \cdot a - (Ab \times Ac) \cdot d}{(Ab \times Ac) \cdot c} $$
where $Ab = b - a$ and $Ac = c - a$. The cross product $N = Ab \times Ac$ is precomputed and stored with the face [S1].

*Point-in-Triangle Test:*
To verify if the intersection point $P = d + t c$ lies inside the triangle defined by $a, b, c$ with ray direction $e$ (where $e$ is the normalized ray direction $c$):
$$ u = \frac{(Ac \times e) \cdot d - (Ac \times e) \cdot a}{(Ac \times e) \cdot b} $$
$$ v = \frac{(Ab \times e) \cdot d - (Ab \times e) \cdot a}{(Ab \times e) \cdot c} $$
The point is inside the face if $u > 0$, $v > 0$, and $u + v < 1$ [S1].

**Degenerate Cases:**
- **Coincident Vertices:** If a tetrahedron has two coincident vertices, two faces have zero area. These faces are ignored. If the ray enters through a non-degenerate face, it exits through the other non-degenerate face. The distance difference $dt = 0$, so no integration is required [S1].
- **Planar Tetrahedron:** If a tetrahedron is planar (no coincident vertices but zero volume), all face intersection distances are identical. The ray passes through the interior of two specific triangular faces (entry and exit). $dt = 0$, so no integration is required [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raytrace_irregular_volume(ray_origin, ray_direction, volume_data):
    """
    ray_origin: vector d
    ray_direction: vector c
    volume_data: object containing cells, faces, and adjacency map
    """
    current_t = 0.0
    current_cell = None
    epsilon = 1e-9
    infinity = float('inf')
    
    # Step 1: Initial Entry
    entry_face = None
    min_t = infinity
    for face in volume_data.exposed_faces:
        N = face.normal
        denom = dot(N, ray_direction)
        if abs(denom) < epsilon:
            continue
        t = (dot(N, face.vertex_a) - dot(N, ray_origin)) / denom
        if t > epsilon:
            intersection_point = ray_origin + t * ray_direction
            if is_point_in_triangle(intersection_point, face.vertex_a, face.vertex_b, face.vertex_c, ray_direction):
                if t < min_t:
                    min_t = t
                    entry_face = face
    
    if entry_face is None:
        return [] # Ray misses volume
    
    current_t = min_t
    current_cell = entry_face.neighbors[0]
    visited_cells = [current_cell]
    
    while True:
        # Step 2: Traverse current cell
        exit_face = None
        min_t = infinity
        
        for face in current_cell.faces:
            N = face.normal
            denom = dot(N, ray_direction)
            
            if abs(denom) < epsilon:
                continue
            
            t = (dot(N, face.vertex_a) - dot(N, ray_origin)) / denom
            
            if t > current_t + epsilon:
                intersection_point = ray_origin + t * ray_direction
                if is_point_in_triangle(intersection_point, face.vertex_a, face.vertex_b, face.vertex_c, ray_direction):
                    if t < min_t:
                        min_t = t
                        exit_face = face
        
        if exit_face is None:
            break # Should not happen in valid convex cell
        
        current_t = min_t
        
        # Check adjacency
        if len(exit_face.neighbors) == 0:
            # Exited dataset, check for re-entry
            re_entry_face = None
            re_min_t = infinity
            for face in volume_data.exposed_faces:
                N = face.normal
                denom = dot(N, ray_direction)
                if abs(denom) < epsilon:
                    continue
                t = (dot(N, face.vertex_a) - dot(N, ray_origin)) / denom
                if t > current_t + epsilon:
                    intersection_point = ray_origin + t * ray_direction
                    if is_point_in_triangle(intersection_point, face.vertex_a, face.vertex_b, face.vertex_c, ray_direction):
                        if t < re_min_t:
                            re_min_t = t
                            re_entry_face = face
            
            if re_entry_face is not None:
                current_t = re_min_t
                current_cell = re_entry_face.neighbors[0]
                visited_cells.append(current_cell)
                continue
            else:
                break # Fully exited
        else:
            # Move to neighbor
            next_cell = None
            for neighbor in exit_face.neighbors:
                if neighbor != current_cell:
                    next_cell = neighbor
                    break
            
            if next_cell is None:
                re_entry_face = None
                re_min_t = infinity
                for face in volume_data.exposed_faces:
                    N = face.normal
                    denom = dot(N, ray_direction)
                    if abs(denom) < epsilon:
                        continue
                    t = (dot(N, face.vertex_a) - dot(N, ray_origin)) / denom
                    if t > current_t + epsilon:
                        intersection_point = ray_origin + t * ray_direction
                        if is_point_in_triangle(intersection_point, face.vertex_a, face.vertex_b, face.vertex_c, ray_direction):
                            if t < re_min_t:
                                re_min_t = t
                                re_entry_face = face
                
                if re_entry_face is not None:
                    current_t = re_min_t
                    current_cell = re_entry_face.neighbors[0]
                    visited_cells.append(current_cell)
                    continue
                else:
                    break
            
            current_cell = next_cell
            visited_cells.append(current_cell)
            
            # Integration step would occur here using current_cell and previous_t
            # previous_t = current_t (before update)
            
    return visited_cells

def is_point_in_triangle(point, a, b, c, ray_dir):
    Ab = b - a
    Ac = c - a
    e = ray_dir
    
    # Formula from [S1]
    denom_u = dot(cross(Ac, e), b)
    if abs(denom_u) < 1e-9: return False
    u = (dot(cross(Ac, e), point) - dot(cross(Ac, e), a)) / denom_u
    
    denom_v = dot(cross(Ab, e), c)
    if abs(denom_v) < 1e-9: return False
    v = (dot(cross(Ab, e), point) - dot(cross(Ab, e), a)) / denom_v
    
    return u > 0 and v > 0 and (u + v) < 1
```

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Efficient for irregular meshes; avoids redundant intersection tests by following connectivity; handles concave volumes via re-entry logic [S1].
- **Limitations:** Requires precomputed adjacency data; degenerate cells require special handling; re-entry checks can be expensive if the surface is complex [S1].
- **When to use:** Ideal for unstructured grid volumes (e.g., finite element analysis data) where regular grid traversal is not applicable [S1].

## Background
This method was introduced by [S1] as a technique for raytracing irregular volume data. It addresses the limitations of regular grid traversal when applied to unstructured meshes. The approach relies on the geometric properties of the cells (tetrahedra or hexahedra) and their adjacency relationships. For hexahedra, if symmetric tessellation is not possible, the method suggests subdividing each hexahedron into twenty-four tetrahedra by adding a vertex at the center of each face [S1].

## Sources
[S1] Raytracing irregular volume data, https://doi.org/10.1145/99307.99316, fetched 2026-10-25
