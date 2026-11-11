# Topologically Defined Isosurfaces

Also called: Digital Geometry Connectedness Marching Cubes

## Overview
Topologically Defined Isosurfaces is a method for extracting a topologically correct, closed, oriented 2-manifold from a binary volumetric image by modifying the Marching Cubes configuration table based on digital geometry connectedness [S1]. It solves the problem of standard isosurface extraction producing surfaces that do not accurately represent the underlying discrete volume topology, specifically ensuring the surface separates foreground from background without singularities or self-intersections [S1]. Unlike empirical corrections or continuous field analysis methods, this approach guarantees consistency by enforcing valid connectedness couples $(\alpha, \lambda)$ that define foreground and background adjacency rules, ensuring the generated surface is a closed 2-manifold consistent with the inverse image [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A binary volumetric image $I$ (voxels are 0 or 1). The image must have 0-voxels or 1-voxels on the border of the support [S1].
*   **Parameters:** A valid connectedness couple $(\alpha, \lambda)$ where $\alpha \in \{6, 18, 26\}$ is the foreground (1-voxel) connectedness and $\lambda \in \{6, 18, 26\}$ is the background (0-voxel) connectedness. Valid couples are $(6, 18), (6, 26), (18, 6), (26, 6)$ [S1].
*   **Output:** A closed, oriented 2-manifold surface $S$ without self-intersections [S1].

**Data Structures:**
*   **Binary Picture $I$:** A 3D grid of integers $\{0, 1\}$.
*   **8-cube:** A set of 8 voxels $v_0 \dots v_7$ forming a voxel unit. Voxels are indexed by binary coding of their position within the cube. Two voxels $v_i, v_j$ are 26-connected if $i \neq j$ [S1].
*   **4-face:** An inner face of an 8-cube composed of 4 voxels. Each 4-face is shared by exactly two adjacent 8-cubes.
*   **Link:** A couple of two 6-connected voxels on a 4-face.
*   **Local Graph $G_{gs}$:** A graph constructed for each 8-cube containing oriented edges and vertices.
*   **Complete Graph $G(I)$:** The union of all local graphs from all 8-cubes in $I$.

**Numbered Steps:**

1.  **Decomposition:** Decompose the binary picture $I$ into the set of all 8-cubes. An 8-cube is defined by its 8 constituent voxels $v_0 \dots v_7$, ordered by binary coding of the index [S1].

2.  **Face Orientation:** For each 8-cube, identify its six inner 4-faces. Orient each inner 4-face such that they are ordered (e.g., using Hamming coding for the 4-face configuration) [S1].

3.  **Link Identification:** For each ordered 4-face, identify its four links. A link is a pair of adjacent 6-connected voxels on the face. The value of a link is determined by the voxel values (0 or 1) [S1].

4.  **Local Graph Construction (4-face rules):** Apply the following rules to each link on an ordered 4-face to generate vertices and oriented edges:
    *   If link value is $(0, 0)$ or $(1, 1)$: No vertex is created.
    *   If link value is $(0, 1)$: Create one vertex between the voxels; add an **incoming** oriented edge on this vertex staying on the 4-face.
    *   If link value is $(1, 0)$: Create one vertex between the voxels; add an **outgoing** oriented edge on this vertex staying on the 4-face.
    
    *Note:* 14 of the 16 4-configuration cases provide an immediate result. Cases 6 and 9 are indeterminate and require special handling based on the chosen valid connectedness couple $(\alpha, \lambda)$ [S1]. Arbitrary choices for invalid couples (e.g., $(6,6), (18,18), (18,26), (26,18), (26,26)$) produce locally different surfaces in the inverse picture, violating consistency [S1].

5.  **Handle Strict 26-Adjacency:** Check if the 8-cube contains two strictly 26-adjacent 1-voxels (if $\alpha=26$) or two strictly 26-adjacent 0-voxels (if $\lambda=26$). If so, the local graph is transformed: the two disconnected loops generated on the 4-faces are triangulated to prevent the surface from intersecting the adjacency 3-map of the strictly 26-adjacent voxels [S1].

6.  **Assemble Local Graph:** Construct the local graph $G_{gs}$ for the 8-cube by gathering all oriented edges generated on the six oriented inner 4-faces. According to Theorem 3, if $(\alpha, \lambda)$ is a valid couple, $G_{gs}$ is a set of oriented loops of length $\ge 3$, disconnected from each other [S1].

7.  **Assemble Complete Graph:** Construct the complete graph $G(I)$ by extending the set of all local graphs built from all 8-cubes of $I$. Each edge of the complete graph belongs to exactly two loops with different orientations [S1].

8.  **Derive Isosurface:** Derive the final isosurface from the complete graph. If loops have length $> 3$, they are subdivided (triangulated) to form a combinatorial manifold. The embedding of this redefined local graph in $\mathbb{R}^3$ does not intersect the adjacency 3-map of strictly 26-adjacent voxels [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def topologically_defined_isosurfaces(binary_volume, alpha, lambda_val):
    """
    Extracts a topologically correct isosurface from a binary volume.
    
    Args:
        binary_volume: 3D numpy array of 0s and 1s.
        alpha: Foreground connectedness (6, 18, or 26).
        lambda_val: Background connectedness (6, 18, or 26).
        
    Returns:
        A list of triangles representing the closed 2-manifold.
    """
    # Validate connectedness couple
    valid_couples = [(6, 18), (6, 26), (18, 6), (26, 6)]
    if (alpha, lambda_val) not in valid_couples:
        raise ValueError(f"Invalid connectedness couple ({alpha}, {lambda_val}). Must be one of {valid_couples}")

    # Initialize global graph structure to store edges and vertices
    # Using a dictionary to map edge identifiers to vertices and orientation
    global_edges = {} 
    # global_edges[(v1, v2)] = {'orientation': 'in'/'out', 'face_id': ...}
    
    # Step 1 & 2: Iterate over all 8-cubes
    for x in range(binary_volume.shape[0] - 1):
        for y in range(binary_volume.shape[1] - 1):
            for z in range(binary_volume.shape[2] - 1):
                # Extract 8-cube voxels v0...v7
                # v0=(x,y,z), v1=(x+1,y,z), v2=(x,y+1,z), v3=(x+1,y+1,z)
                # v4=(x,y,z+1), v5=(x+1,y,z+1), v6=(x,y+1,z+1), v7=(x+1,y+1,z+1)
                cube_voxels = [
                    binary_volume[x, y, z],
                    binary_volume[x+1, y, z],
                    binary_volume[x, y+1, z],
                    binary_volume[x+1, y+1, z],
                    binary_volume[x, y, z+1],
                    binary_volume[x+1, y, z+1],
                    binary_volume[x, y+1, z+1],
                    binary_volume[x+1, y+1, z+1]
                ]
                
                # Step 3 & 4: Process six inner 4-faces
                # Define the 6 faces, each as a list of 4 voxel indices and their link pairs
                # Links are pairs of adjacent voxels on the face.
                # Orientation of 4-faces must be consistent (e.g., Hamming coding order)
                faces = get_inner_faces_indices() 
                
                for face in faces:
                    # face contains 4 voxel indices and 4 link definitions
                    # Links are ordered clockwise on the oriented face
                    
                    # Determine 4-face configuration value based on cube_voxels
                    config = compute_4face_config(cube_voxels, face)
                    
                    # Process each link on the face
                    for link in face.links:
                        v_a, v_b = link.voxels
                        val_a = cube_voxels[v_a]
                        val_b = cube_voxels[v_b]
                        
                        # Apply rules
                        if val_a == 0 and val_b == 0:
                            continue # No vertex
                        elif val_a == 1 and val_b == 1:
                            continue # No vertex
                        elif val_a == 0 and val_b == 1:
                            # Create vertex, incoming edge
                            vertex_id = create_vertex_between(v_a, v_b)
                            add_oriented_edge(vertex_id, orientation='in', face_id=face.id)
                        elif val_a == 1 and val_b == 0:
                            # Create vertex, outgoing edge
                            vertex_id = create_vertex_between(v_a, v_b)
                            add_oriented_edge(vertex_id, orientation='out', face_id=face.id)
                        
                        # Special handling for indeterminate cases 6 and 9
                        # This depends on alpha and lambda_val
                        if config in [6, 9]:
                            resolve_indeterminate_case(config, alpha, lambda_val, face, global_edges)

                # Step 5: Handle strict 26-adjacency within the 8-cube
                if alpha == 26:
                    if has_strictly_26_adjacent_1s(cube_voxels):
                        triangulate_disconnected_loops_in_local_graph(global_edges, cube_voxels, alpha, lambda_val)
                if lambda_val == 26:
                    if has_strictly_26_adjacent_0s(cube_voxels):
                        triangulate_disconnected_loops_in_local_graph(global_edges, cube_voxels, alpha, lambda_val)

    # Step 7: Complete graph is implicitly built by iterating all cubes and adding edges
    # Step 8: Derive surface from complete graph
    # Theorem 5 states G(I) is an oriented 2D combinatorial manifold.
    # Extract loops from global_edges.
    loops = extract_loops_from_global_graph(global_edges)
    
    # Subdivide loops of length > 3
    triangles = []
    for loop in loops:
        if len(loop) > 3:
            triangles.extend(subdivide_loop_to_triangles(loop))
        else:
            # Length 3 is already a triangle
            triangles.append(loop)
            
    return triangles

def get_inner_faces_indices():
    """Returns the 6 inner faces of an 8-cube, each with 4 voxel indices and 4 links."""
    # Implementation details for face definition and link ordering
    # Must ensure consistent orientation across shared faces
    pass

def compute_4face_config(cube_voxels, face):
    """Computes the 4-bit configuration value for the 4-face."""
    pass

def create_vertex_between(v_a, v_b):
    """Creates a new vertex geometry between two voxels."""
    pass

def add_oriented_edge(vertex_id, orientation, face_id):
    """Adds an oriented edge to the global structure."""
    pass

def resolve_indeterminate_case(config, alpha, lambda_val, face, global_edges):
    """
    Resolves cases 6 and 9 based on valid connectedness couple.
    """
    # Logic derived from Figure 1.b in [S1]
    # Ensures consistency with inverse image
    pass

def has_strictly_26_adjacent_1s(cube_voxels):
    """Checks if any two 1-voxels in the cube are strictly 26-adjacent."""
    pass

def has_strictly_26_adjacent_0s(cube_voxels):
    """Checks if any two 0-voxels in the cube are strictly 26-adjacent."""
    pass

def triangulate_disconnected_loops_in_local_graph(global_edges, cube_voxels, alpha, lambda_val):
    """
    Transforms the local graph by triangulating disconnected loops
    to prevent intersection with adjacency 3-map.
    """
    pass

def extract_loops_from_global_graph(global_edges):
    """
    Extracts oriented loops from the complete graph.
    Each edge belongs to exactly two loops with different orientations.
    """
    pass

def subdivide_loop_to_triangles(loop):
    """
    Subdivides a loop of length > 3 into triangles.
    """
    pass
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels, as each voxel is part of a constant number of 8-cubes and each 8-cube is processed in constant time [S1].
*   **Memory Complexity:** Linear with respect to the number of voxels, storing the global graph structure [S1].
*   **Strengths:** Guarantees topological correctness (closed, oriented 2-manifold, no self-intersections) and consistency with the inverse image [S1]. It ensures the surface lies within limits defined by voxel connections [S1].
*   **Limitations:** Restricted to binary volumes. The choice of connectedness couple $(\alpha, \lambda)$ affects the surface geometry; invalid couples produce incorrect topologies [S1]. Requires careful handling of indeterminate cases (6 and 9) and strict 26-adjacency [S1].
*   **When to use:** When topological correctness and manifold properties are critical, and the input data is binary.
*   **When not to use:** For scalar fields requiring continuous interpolation or when the underlying topology does not align with the chosen digital connectivity rules.

## Background
The method was introduced in the paper "Topologically defined isosurfaces" [S1]. It addresses the limitations of the standard Marching Cubes algorithm, which can produce surfaces that do not represent the underlying volume topology correctly [S1]. The approach exploits the underlying discrete topology of voxels, providing a formal proof of the validity of the generated isosurface [S1]. It demonstrates that the graph defining the isosurface is closely linked to the surfel-adjacency graph of the digital surface of the same image [S1].

## Sources
[S1] Topologically defined isosurfaces, https://doi.org/10.1007/3-540-62005-2_21, fetched 2026-10-27
