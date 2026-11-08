# Robust Watertight Manifold Surface Generation

Also called: Octree-based manifold generation, ShapeNet manifold pipeline

## Overview
This method converts an input triangle mesh into a watertight 2-manifold surface with roughly uniformly distributed vertices using an octree-based signed distance field (SDF) representation and isosurface extraction [S1, unofficial]. It differs from uniform grid marching cubes by adaptively refining only the volume near the surface, reducing storage and computation linearly with the final mesh size [S1, unofficial]. Unlike sibling methods that fit explicit surface primitives (e.g., QEF) or use complex occlusion handling, this pipeline guarantees topological correctness by defining "positive" space via boundary-connected BFS and resolving non-manifold ambiguities through vertex/edge splitting [S1, unofficial].

## How it works
**Inputs:** A triangle mesh $M$ with vertices and faces.
**Outputs:** A watertight 2-manifold triangle mesh with vertices projected onto the original geometry.

**Data Structures:**
*   **Octree Node:**
    *   `bbox`: Axis-aligned bounding box (min/max coordinates).
    *   `triangles`: List of triangles from $M$ contained in or intersecting `bbox`.
    *   `status`: Enum {Empty, Occupied}.
    *   `children`: Array of 8 child nodes (only present if `status` is Occupied and resolution < target).
*   **Connection Map:** Implicitly built during traversal; stores adjacency between occupied nodes and their neighbors (empty or occupied) to facilitate face extraction.

**Parameters:**
*   **Target Resolution:** Unit cube length $L = 0.01$ in the normalized space. This ensures the output mesh has $>10,000$ triangles [S1, unofficial].
*   **Root Bounding Box:** Extends from $-1.1$ to $1.1$ in each dimension to fully contain the normalized mesh (range $-1$ to $1$) [S1, unofficial].

**Steps:**

1.  **Normalization:** Translate and uniformly scale $M$ so its geometric center is at the origin and its bounding box dimensions range from $-1$ to $1$ [S1, unofficial].

2.  **Octree Construction:**
    *   Initialize root node with `bbox` $[-1.1, 1.1]^3$ and `triangles` = all triangles in $M$. Set `status` = Occupied.
    *   Recursively subdivide any node with `status` = Occupied and `bbox` side length $> 0.01$:
        *   Split into 8 children.
        *   For each child, compute its sub-bbox.
        *   Assign triangles from the parent that intersect the child's sub-bbox.
        *   If a child has no triangles, set `status` = Empty. Otherwise, `status` = Occupied.
        *   Recurse on children with `status` = Occupied [S1, unofficial].

3.  **Connection Building:**
    *   Recursively traverse the octree to build adjacency information between neighboring nodes.
    *   For each node, process its 8 children.
    *   For each of the 12 pairs of neighboring children (corresponding to the 12 edges of a cube), call `ConnectNodes` [S1, unofficial].
    *   `ConnectNodes(Nl, Nr)`:
        *   If both `Nl` and `Nr` are Empty, return.
        *   Define `SubNl`: If `Nl` is Empty, `SubNl` = $\{Nl, Nl, Nl, Nl\}$. Else, `SubNl` = `Nl.children` $\{s_1, s_2, s_3, s_4\}$ (the 4 children adjacent to the interface).
        *   Define `SubNr`: If `Nr` is Empty, `SubNr` = $\{Nr, Nr, Nr, Nr\}$. Else, `SubNr` = `Nr.children` $\{s_5, s_6, s_7, s_8\}$ (the 4 children adjacent to the interface).
        *   Recursively call `ConnectNodes(SubNl[i], SubNr[i])` for $i=1 \dots 4$ [S1, unofficial].

4.  **Volume Classification (Positive/Negative):**
    *   Initialize all boundary cubes (nodes at the finest resolution or on the root boundary) as **Positive** if they do not contain triangles.
    *   Perform a Breadth-First Search (BFS) from the boundary into the empty space. Any empty cube reachable from the boundary without passing through an occupied cube is marked **Positive**.
    *   All remaining empty cubes (not reachable from boundary) are marked **Negative** (inside the surface).
    *   Occupied cubes are considered **Zero** (intersecting) [S1, unofficial].

5.  **Isosurface Extraction:**
    *   Extract faces between **Positive** cubes and **Occupied** cubes.
    *   Recursively visit all occupied nodes and their neighbors.
    *   For each pair of adjacent cubes where one is Positive and the other is Occupied, generate the interface face(s) [S1, unofficial].

6.  **Non-Manifold Resolution:**
    *   Detect ambiguities where two cubes meet at a single vertex or share an edge connecting 4 faces.
    *   Resolve by splitting the vertex into three or the edge into two, assuming volumes are connected [S1, unofficial].

7.  **Vertex Projection:**
    *   For each extracted vertex, find the nearest triangle in the original mesh $M$ using the stored triangle-node relations.
    *   Iteratively move the vertex closer to its nearest triangle along the direction parallel to the triangle's normal with a fixed small step size.
    *   Apply Laplacian smoothing after each projection step to prevent face flips [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def BuildOctree(triangles, bbox):
    """
    Recursively builds the octree.
    triangles: List of triangles in the current bounding box.
    bbox: Bounding box of the current node.
    Returns: OctreeNode
    """
    node = OctreeNode()
    node.bbox = bbox
    node.triangles = triangles
    
    if not triangles:
        node.status = Empty
        node.children = None
        return node
    
    node.status = Occupied
    
    # Check if finest resolution reached
    if bbox.side_length() <= 0.01:
        node.children = None
        return node
    
    # Subdivide
    node.children = [None] * 8
    sub_bboxes = get_sub_bboxes(bbox)
    
    for i in range(8):
        # ContainedBy returns triangles from 'triangles' that intersect sub_bboxes[i]
        sub_triangles = ContainedBy(triangles, sub_bboxes[i])
        node.children[i] = BuildOctree(sub_triangles, sub_bboxes[i])
        
    return node

def BuildConnection(node):
    """
    Builds connections between neighboring nodes.
    """
    if node.status == Empty:
        return
    
    # Recursively build connections inside children
    for child in node.children:
        if child:
            BuildConnection(child)
            
    # Build connections between neighboring children
    # Neighbor[i] returns indices of two children that share an edge
    for i in range(12):
        idx1, idx2 = Neighbor[i]
        child1 = node.children[idx1]
        child2 = node.children[idx2]
        ConnectNodes(child1, child2)

def ConnectNodes(Nl, Nr):
    """
    Connects two neighboring nodes, handling size mismatches.
    """
    if Nl is None or Nl.status == Empty:
        SubNl = [Nl, Nl, Nl, Nl] # Use placeholder for empty
    else:
        # s1..s4 are indices of children adjacent to the interface
        SubNl = [Nl.children[s1], Nl.children[s2], Nl.children[s3], Nl.children[s4]]
        
    if Nr is None or Nr.status == Empty:
        SubNr = [Nr, Nr, Nr, Nr] # Use placeholder for empty
    else:
        # s5..s8 are indices of children adjacent to the interface
        SubNr = [Nr.children[s5], Nr.children[s6], Nr.children[s7], Nr.children[s8]]
        
    for i in range(4):
        ConnectNodes(SubNl[i], SubNr[i])

def ClassifyVolumes(root):
    """
    Marks empty cubes as Positive (outside) or Negative (inside).
    """
    # Initialize queue with boundary empty nodes
    queue = get_boundary_empty_nodes(root)
    for node in queue:
        node.volume_type = Positive
        
    # BFS
    while queue not empty:
        current = queue.pop()
        for neighbor in get_neighbors(current):
            if neighbor.volume_type is None and neighbor.status == Empty:
                neighbor.volume_type = Positive
                queue.push(neighbor)
                
    # All remaining empty nodes are Negative
    for node in all_empty_nodes(root):
        if node.volume_type is None:
            node.volume_type = Negative

def ExtractSurface(root):
    """
    Extracts faces between Positive and Occupied cubes.
    """
    faces = []
    for node in all_occupied_nodes(root):
        for neighbor in get_neighbors(node):
            if neighbor.volume_type == Positive:
                # Generate face between node and neighbor
                face = GenerateInterfaceFace(node, neighbor)
                faces.append(face)
    return faces

def ProjectVertices(vertices, original_mesh):
    """
    Projects extracted vertices onto the original mesh.
    """
    for vertex in vertices:
        # Find nearest triangle in original_mesh
        nearest_tri = FindNearestTriangle(vertex, original_mesh)
        
        # Iterative projection
        for step in range(max_iterations):
            # Move along normal direction
            vertex = vertex + step_size * nearest_tri.normal
            # Re-find nearest triangle if needed or update
            # Apply Laplacian smoothing to neighbors to prevent flip
            ApplyLaplacianSmoothing(vertex, neighbors)
            
    return vertices
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of triangles in the final mesh, as the octree resolution is coarser in empty volumes [S1, unofficial].
*   **Memory Complexity:** Linear with respect to the final mesh size, significantly less than a uniform grid for large volumes with thin surfaces [S1, unofficial].
*   **Performance:** Guarantees correct 2-manifold topology for all ShapeNet models tested [S1, unofficial].
*   **Limitations:** Approximately 5% of models may contain face flips, primarily due to thin face structures in the original mesh [S1, unofficial].
*   **When to use:** When a watertight, manifold representation is required from a potentially non-manifold or noisy input mesh, and uniform grid storage is prohibitive.
*   **When not to use:** When high-frequency geometric details are lost due to the coarse resolution in empty space, or when the input mesh has extreme thin structures that cause projection failures.

## Background
The method was introduced in a paper focusing on processing ShapeNet models [S1, unofficial]. It builds on the concept of isosurface extraction (similar to Marching Cubes [Lorensen and Cline 1987]) but uses an octree to represent the signed distance field, inspired by Meagher's octree encoding [Meagher 1982]. The pipeline was designed to handle the variety of ShapeNet models, including those with holes and thin structures, ensuring topological correctness [S1, unofficial].

## Sources
[S1] Robust Watertight Manifold Surface Generation Method for ShapeNet Models, https://arxiv.org/abs/1802.01698v1, fetched 2026-10-26 (unofficial)
