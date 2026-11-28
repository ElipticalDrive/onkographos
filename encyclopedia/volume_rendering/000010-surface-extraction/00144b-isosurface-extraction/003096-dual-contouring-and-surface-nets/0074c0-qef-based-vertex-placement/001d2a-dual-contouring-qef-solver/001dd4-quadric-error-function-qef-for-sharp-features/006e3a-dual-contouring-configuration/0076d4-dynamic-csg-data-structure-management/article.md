# Dynamic CSG Data Structure Management

Also called: Dynamic Octree Vertex/Topology Management

## Overview
A data structure management scheme for maintaining a mesh generated from a multi-resolution octree during Constructive Solid Geometry (CSG) operations, involving dynamic addition and removal of vertices and polygons [S1, unofficial]. It enables efficient incremental updates to the dual contouring output by keeping the vertex and topology arrays consistent without full regeneration, distinguishing it from static meshing approaches that require rebuilding the entire structure after geometric modifications [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A multi-resolution octree representing the implicit volume, a vertex array, and a topology array.
*   **Output:** Updated vertex and topology arrays reflecting the CSG operation (union, intersection, difference).

**Data Structures**
The system relies on two primary arrays:

1.  **Vertex Array:** Each entry corresponds to a vertex in the mesh.
    *   `position`: 3D coordinates.
    *   `normal`: Surface normal.
    *   `tex_coords`: Texture coordinates.
    *   `polygon_indices`: A list of indices into the topology array, representing all quads containing this vertex.
    *   `node_ptr`: A pointer to the octree node that generated this vertex.
2.  **Topology Array:** Each entry represents a polygon (quad).
    *   `vertex_indices`: A list of four indices into the vertex array.

**Algorithm Steps**

The management scheme operates through four primitive operations: `Add Vertex`, `Remove Vertex`, `Add Polygon`, and `Remove Polygon`. These are invoked during or after CSG operations to maintain consistency.

1.  **Initial Construction:**
    *   Perform a single pass over the octree to generate vertices.
    *   Populate the vertex array with positions, normals, etc.
    *   Set `node_ptr` for each vertex to the corresponding octree node.
    *   Initialize all `polygon_indices` lists as empty.
    *   The topology array is initially empty.

2.  **Initial Contouring:**
    *   Run the multi-resolution contouring algorithm (as defined in Dual Contouring).
    *   For each generated polygon:
        *   Append the polygon (4 vertex indices) to the end of the topology array.
        *   For each of the four vertices in the polygon, append the new topology index to that vertex's `polygon_indices` list.

3.  **CSG Update (Post-Operation):**
    *   After CSG modifies the octree (adding/removing heterogeneous nodes), the vertex and topology arrays are updated via the primitives below.
    *   Perform a truncated version of the full polygon generation pass to add new polygons into the model for the affected regions [S1, unofficial].

4.  **Primitive: Add Vertex**
    *   Triggered when a CSG operation adds a new heterogeneous node.
    *   Append a new entry to the end of the vertex array.
    *   Initialize `polygon_indices` as empty.
    *   Set `node_ptr` to the new heterogeneous node.

5.  **Primitive: Add Polygon**
    *   Triggered when a new quad is generated.
    *   Append the polygon (4 vertex indices) to the end of the topology array.
    *   For each of the four vertices referenced by the polygon, append the new topology index to that vertex's `polygon_indices` list.

6.  **Primitive: Remove Polygon**
    *   Input: `poly_idx` (index of the polygon to remove).
    *   1. Retrieve the four vertex indices from `topology[poly_idx]`.
    *   2. For each vertex index `v_idx`:
        *   Request `vertex[v_idx]` to remove `poly_idx` from its `polygon_indices` list.
    *   3. Let `last_poly_idx` be the index of the last element in the topology array.
    *   4. If `poly_idx != last_poly_idx`:
        *   Copy `topology[last_poly_idx]` into `topology[poly_idx]`.
        *   Let `new_poly_idx` be `last_poly_idx`.
        *   Retrieve the four vertex indices from the *new* `topology[poly_idx]` (which was just copied).
        *   For each vertex index `v_idx` in this polygon:
            *   Request `vertex[v_idx]` to renumber its reference to this polygon from `new_poly_idx` to `poly_idx`.
    *   5. Remove the last element from the topology array.

7.  **Primitive: Remove Vertex**
    *   Triggered when a heterogeneous node is modified or deleted.
    *   Input: `v_idx` (index of the vertex to remove).
    *   1. Retrieve the list of polygon indices `P_list` from `vertex[v_idx].polygon_indices`.
    *   2. Sort `P_list` in descending order (highest index to lowest). *Note: Processing from highest to lowest eliminates problems arising from renumbering the polygons in the deleted vertex's polygon list [S1, unofficial].*
    *   3. For each `p_idx` in `P_list`:
        *   Call `Remove Polygon(p_idx)`.
    *   4. Let `last_v_idx` be the index of the last element in the vertex array.
    *   5. If `v_idx != last_v_idx`:
        *   Copy `vertex[last_v_idx]` into `vertex[v_idx]`.
        *   Update `vertex[v_idx].node_ptr.node_index` (the index stored in the node pointing to this vertex) to `v_idx`.
        *   Retrieve the list of polygon indices `P_moved` from `vertex[v_idx].polygon_indices`.
        *   For each `p_idx` in `P_moved`:
            *   Retrieve the polygon `topology[p_idx]`.
            *   Find the position of `last_v_idx` within `topology[p_idx]`.
            *   Replace `last_v_idx` with `v_idx` in `topology[p_idx]`.

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Structures
class Vertex:
    position: Vector3
    normal: Vector3
    tex_coords: Vector2
    polygon_indices: List[int]
    node_ptr: Node # Contains node_index

class Topology:
    vertex_indices: List[int] # Size 4

class MeshData:
    vertices: List[Vertex]
    topology: List[Topology]

    def add_vertex(self, node: Node) -> int:
        """Add a vertex for a new heterogeneous node."""
        v = Vertex()
        v.position = Vector3(0, 0, 0) # Placeholder, filled by contouring
        v.normal = Vector3(0, 0, 0)
        v.tex_coords = Vector2(0, 0)
        v.polygon_indices = []
        v.node_ptr = node
        self.vertices.append(v)
        return len(self.vertices) - 1

    def add_polygon(self, v_indices: List[int]) -> int:
        """Add a polygon (quad) defined by 4 vertex indices."""
        poly = Topology()
        poly.vertex_indices = v_indices
        self.topology.append(poly)
        poly_idx = len(self.topology) - 1
        
        for v_idx in v_indices:
            self.vertices[v_idx].polygon_indices.append(poly_idx)
            
        return poly_idx

    def remove_polygon(self, poly_idx: int):
        """Remove a polygon from the topology list."""
        if poly_idx >= len(self.topology):
            return # Error handling
        
        # 1. Update vertices to remove reference to this polygon
        v_indices = self.topology[poly_idx].vertex_indices
        for v_idx in v_indices:
            self._remove_polygon_from_vertex(v_idx, poly_idx)
            
        # 2. Compact topology array
        last_idx = len(self.topology) - 1
        if poly_idx != last_idx:
            # Move last polygon to the hole
            self.topology[poly_idx] = self.topology[last_idx]
            
            # 3. Renumber the moved polygon's references in vertices
            new_v_indices = self.topology[poly_idx].vertex_indices
            for v_idx in new_v_indices:
                self._renumber_polygon_in_vertex(v_idx, last_idx, poly_idx)
                
        # 3. Remove last element
        self.topology.pop()

    def _remove_polygon_from_vertex(self, v_idx: int, poly_idx: int):
        """Helper to remove a specific polygon index from a vertex's list."""
        v = self.vertices[v_idx]
        if poly_idx in v.polygon_indices:
            v.polygon_indices.remove(poly_idx)

    def _renumber_polygon_in_vertex(self, v_idx: int, old_idx: int, new_idx: int):
        """Helper to renumber a polygon index in a vertex's list."""
        v = self.vertices[v_idx]
        for i, idx in enumerate(v.polygon_indices):
            if idx == old_idx:
                v.polygon_indices[i] = new_idx
                break

    def remove_vertex(self, v_idx: int):
        """Remove a vertex and all its associated polygons."""
        if v_idx >= len(self.vertices):
            return # Error handling
            
        # 1. Get polygons and sort descending to handle renumbering safely
        p_indices = self.vertices[v_idx].polygon_indices
        p_indices.sort(reverse=True)
        
        # 2. Remove each polygon
        for p_idx in p_indices:
            self.remove_polygon(p_idx)
            
        # 3. Compact vertex array
        last_idx = len(self.vertices) - 1
        if v_idx != last_idx:
            # Move last vertex to the hole
            self.vertices[v_idx] = self.vertices[last_idx]
            
            # 4. Update the node pointer in the moved vertex's node
            # The node that generated this vertex needs to point to the new index
            moved_vertex = self.vertices[v_idx]
            if moved_vertex.node_ptr is not None:
                moved_vertex.node_ptr.node_index = v_idx
                
            # 5. Renumber the vertex index in all polygons referencing the moved vertex
            # The moved vertex now sits at v_idx, but its old polygons still 
            # reference last_idx. We must update those references.
            p_indices_moved = self.vertices[v_idx].polygon_indices
            for p_idx in p_indices_moved:
                self._renumber_vertex_in_polygon(p_idx, last_idx, v_idx)
                
        # 6. Remove last element
        self.vertices.pop()

    def _renumber_vertex_in_polygon(self, p_idx: int, old_v_idx: int, new_v_idx: int):
        """Helper to renumber a vertex index within a specific polygon."""
        poly = self.topology[p_idx]
        for i, v_idx in enumerate(poly.vertex_indices):
            if v_idx == old_v_idx:
                poly.vertex_indices[i] = new_v_idx
                break
```

**Parameters and Edge Cases**
*   **Precision:** Not given in the sources.
*   **Boundaries:** The compacting logic (swapping with the last element) assumes the arrays are dense. If `v_idx == last_idx` or `poly_idx == last_idx`, no swap is performed, avoiding self-overwrite issues.
*   **Degenerate Input:** If a vertex has no polygons, `remove_vertex` simply compacts the vertex array. If a polygon has invalid vertex indices, behavior is undefined (standard array bounds check applies).

## Operating characteristics
*   **Time Complexity:**
    *   `Add Vertex`: $O(1)$.
    *   `Add Polygon`: $O(1)$ amortized (assuming constant number of vertices per polygon).
    *   `Remove Polygon`: $O(V_p)$ where $V_p$ is the number of vertices in the polygon (constant 4) plus $O(V_p)$ for renumbering. Effectively $O(1)$.
    *   `Remove Vertex`: $O(P_v \cdot V_p)$ where $P_v$ is the number of polygons connected to the vertex. In the worst case (highly connected vertex), this is $O(N)$ where $N$ is the number of polygons.
*   **Memory Complexity:** $O(V + P)$ where $V$ is the number of vertices and $P$ is the number of polygons. The structure stores explicit lists of polygon indices for each vertex, leading to redundancy but enabling fast adjacency traversal.
*   **Strengths:** Allows for incremental updates during CSG without rebuilding the entire mesh. The compacting strategy ensures memory efficiency.
*   **Limitations:** Removing a vertex with high valence (many connected polygons) can be expensive. The explicit storage of polygon indices in vertices increases memory usage compared to a simple index buffer.
*   **When to use:** When performing dynamic CSG operations on dual-contoured meshes where incremental updates are required.
*   **When not to use:** For static meshes or when the cost of vertex removal dominates the application runtime.

## Background
Originated in the context of Dual Contouring for multi-resolution octrees. The method was described by Schaefer and Warren in their work on Dual Contouring, often referred to as "The Secret Sauce" [S1, unofficial]. It addresses the need to maintain mesh consistency when the underlying octree structure changes due to CSG operations, ensuring that the vertex and topology arrays remain synchronized with the octree nodes.

## Sources
[S1] PDF Dual Contouring: "The Secret Sauce", https://people.eecs.berkeley.edu/~jrs/meshpapers/SchaeferWarren2.pdf, fetched 2026-10-27 (unofficial)
