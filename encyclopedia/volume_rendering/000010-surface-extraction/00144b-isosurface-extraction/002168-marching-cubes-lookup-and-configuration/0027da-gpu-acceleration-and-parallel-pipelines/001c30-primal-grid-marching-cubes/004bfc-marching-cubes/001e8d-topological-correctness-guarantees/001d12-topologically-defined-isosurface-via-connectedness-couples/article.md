# Topologically Defined Isosurface via Connectedness Couples

Also called: MC with (alpha, lambda) couples, Marching-Cubes with topological constraints

## Overview
This method extracts a topologically correct isosurface from a binary volumetric image by enforcing valid connectedness couples $(\alpha, \lambda)$, ensuring the result is a closed 2-manifold without singularities or self-intersections [S1]. Unlike methods that rely on continuous field analysis or geometric insertion of internal vertices, this approach uses discrete connectivity constraints to define the local triangulation rules, guaranteeing that the complete graph of the binary image forms an oriented two-dimensional combinatorial manifold without boundary [S1]. It is distinct from [Topologically Correct Marching Cubes](1368-topologically-correct-marching-cubes.md) and [Asymptotic Decider](790-asymptotic-decider.md) by avoiding per-voxel ambiguity resolution in favor of precomputed local configuration tables derived from topological validity [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A binary volumetric image $I$ (voxel values 0 or 1).
- **Output:** A triangulated surface mesh representing the isosurface $\Sigma_{\alpha,\lambda}(I)$.

**Data Structures:**
- **Configuration Table:** A precomputed lookup table for each of the 256 possible voxel configurations (binary patterns of the 8 corners). There are four tables, one for each valid connectedness couple: $(26, 6)$, $(18, 6)$, $(6, 26)$, and $(6, 18)$ [S1].
- **Local Graph $G_{\alpha}(C_8)$:** A set of oriented loops representing the intersection of the isosurface with a single 8-cube (voxel) $C_8$ [S1].
- **Complete Graph $G_{\alpha,\lambda}(I)$:** The union of all local graphs from the volume, forming the global surface structure [S1].

**Valid Connectedness Couples:**
A pair $(\alpha, \lambda)$ is valid if $\alpha$ is the foreground connectivity and $\lambda$ is the background connectivity. The valid couples are:
1. $(26, 6)$
2. $(18, 6)$
3. $(6, 26)$
4. $(6, 18)$ [S1]

**Algorithm Steps:**

1.  **Precomputation (Build-Local_Surface):**
    For each valid connectedness couple $(\alpha, \lambda)$, precompute the triangulation for all 256 voxel configurations. This is done by calling `Build-Local_Surface` for each configuration $C$ [S1].

2.  **Surface Extraction:**
    Iterate through the volume. For each voxel, determine its binary configuration. Retrieve the corresponding triangulation from the precomputed table for the chosen connectedness couple. Assemble the triangles.

**Detailed Precomputation Procedure (`Build-Local_Surface`):**
The procedure constructs the local triangulated manifold $S_{\alpha}(C_8)$ for a given configuration $C$ and connectivity $\alpha$.

1.  Initialize edge set $E \leftarrow \emptyset$.
2.  For all inner 6-faces $F$ of the voxel $C$:
    - Extract edges $E_F \leftarrow \text{Extract\_edges}(F, g, 6)$.
    - Update $E \leftarrow E \cup E_F$.
3.  Extract loops $L \leftarrow \text{Extract\_loops}(E)$.
4.  Compute the convex hull $H \leftarrow \text{Convex\_Hull}(\text{get\_vertices}(E) \cup \text{get\_1-voxels}(C))$.
5.  While $L$ contains a loop of length $> 3$:
    - Remove a loop $L = (a_1, \dots, a_k)$ from $L$.
    - Initialize `exit` $\leftarrow$ false, $i \leftarrow 1$, $j \leftarrow i + 2$.
    - Repeat:
        - If the chord $(a_i, a_j)$ is inside $H$ (i.e., $(a_i, a_j) \in H$):
            - `exit` $\leftarrow$ true.
        - Else:
            - $j \leftarrow j + 1$.
            - If $j = k$:
                - $i \leftarrow i + 1$.
                - $j \leftarrow i + 2$.
    - Until `exit` is true.
    - Insert the subdivided loop $(a_1, \dots, a_i, a_j, \dots, a_k)$ back into $L$.
    - Insert the chord loop $(a_i, \dots, a_j)$ into the triangulation set $s$.
6.  If Configuration $C$ has strict 26-adjacency and $\alpha = 26$:
    - Replace the current triangulation with special loops: $s \leftarrow \text{build\_special\_loops\_26-adj}(s)$.
7.  Return $s$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_local_surface(configuration: int, alpha: int) -> List[Triangle]:
    """
    Precomputes the triangulation for a single voxel configuration.
    
    Args:
        configuration: Integer 0-255 representing the binary state of the 8 voxel corners.
        alpha: Foreground connectivity (6, 18, or 26).
        
    Returns:
        List of triangles forming the local surface.
    """
    # 1. Initialize edge set
    edges = set()
    
    # 2. Extract edges from inner 6-faces
    # A voxel has 6 inner faces (excluding the outer boundary of the voxel itself)
    for face in get_inner_faces(configuration):
        face_edges = extract_edges_on_face(face, connectivity=6)
        edges.update(face_edges)
        
    # 3. Extract loops from edges
    loops = extract_loops(edges)
    
    # 4. Compute Convex Hull H
    # H contains vertices from the extracted edges and the 1-voxels (corners) of the configuration
    all_vertices = get_vertices(edges)
    voxel_corners = get_1_voxels(configuration) # Indices of corners with value 1
    # Note: The source defines H based on get_vertices(E) U get_1-voxels(C).
    # In the context of the algorithm, this likely refers to the geometric positions 
    # of the vertices involved in the loops and the active corners.
    H = compute_convex_hull(all_vertices + voxel_corners)
    
    triangulation = []
    
    # 5. Subdivide long loops
    while loops:
        # Remove a loop of length > 3
        loop = loops.pop()
        k = len(loop)
        
        if k <= 3:
            # Add triangle directly if it's a triangle
            triangulation.append(make_triangle(loop))
            continue
            
        # Loop subdivision logic
        exit_flag = False
        i = 0
        j = 2 # i + 2
        
        while not exit_flag:
            # Check if chord (loop[i], loop[j]) is in H
            # Note: The source uses 1-based indexing (a1...ak). 
            # Here we use 0-based. a_i corresponds to loop[i].
            # The condition is (ai, aj) in H.
            if is_chord_in_convex_hull(loop[i], loop[j], H):
                exit_flag = True
            else:
                j += 1
                if j == k:
                    i += 1
                    j = i + 2
                    # If i reaches the end, the loop is convex or handled differently?
                    # The source implies the loop terminates when a valid chord is found.
                    # If j wraps around completely without finding a chord, 
                    # the loop is likely convex and should be triangulated as is.
                    # However, the source logic "If j=k then i=i+1, j=i+2" suggests 
                    # continuing the search. If i catches up to j, we stop.
                    if i >= k - 2: 
                        # Fallback: triangulate the convex polygon
                        triangulation.extend(triangulate_polygon(loop))
                        loop = None
                        break
        
        if loop is None:
            continue
            
        # Insert subdivided loop and chord
        # The source says: Insert_loop((a1...ai, aj...ak), s) and Insert_loop(ai...aj), L)
        # This implies splitting the loop into two: one containing the chord, one being the chord itself?
        # Actually, "Insert_loop(..., s)" adds to triangulation, "Insert_loop(..., L)" adds back to loops.
        
        # Split loop at i and j
        # Part 1: a1...ai, aj...ak (the outer part)
        part1 = loop[:i+1] + loop[j:]
        # Part 2: ai...aj (the chord part, which forms a new loop to be triangulated or further split)
        part2 = loop[i:j+1]
        
        # Add part 2 to triangulation? Or is it a chord that creates triangles?
        # The source says: Insert_loop(ai...aj), L). This suggests part2 is added back to loops.
        # And part1 is added to s (triangulation).
        # But part1 is a loop. If it's length > 3, it will be processed again.
        # If part2 is length 3, it becomes a triangle.
        
        if len(part2) == 3:
            triangulation.append(make_triangle(part2))
        else:
            loops.append(part2)
            
        if len(part1) == 3:
            triangulation.append(make_triangle(part1))
        else:
            loops.append(part1)

    # 6. Handle strict 26-adjacency
    if is_strict_26_adjacency(configuration) and alpha == 26:
        triangulation = build_special_loops_26_adj(triangulation)
        
    return triangulation

def is_chord_in_convex_hull(p1, p2, H):
    """
    Checks if the segment p1-p2 lies within the convex hull H.
    In the context of the algorithm, this determines if the chord is "internal"
    and thus a valid diagonal for triangulation.
    """
    # Implementation detail: Check if the midpoint or the segment is contained in H.
    # Since H is the convex hull of the vertices, any chord between vertices 
    # that does not cross the boundary of H is inside.
    return segment_in_hull(p1, p2, H)
```

**Edge Cases and Pitfalls:**
- **Strict 26-Adjacency:** If the voxel configuration exhibits strict 26-adjacency (all 26 neighbors are connected) and $\alpha=26$, the standard subdivision is replaced by special loops generated by `build_special_loops_26-adj`. If $\alpha=18$, the local graph is not modified in this case [S1].
- **Consistency:** The method relies on precomputed tables, ensuring that shared faces between voxels are triangulated consistently, preventing cracks [S1].

## Operating characteristics
- **Time Complexity:** The precomputation is constant (256 configurations $\times$ 4 tables). The extraction phase using surface tracking to build the complete graph is $O(s)$, where $s$ is the number of surfels (typically $O(n^2)$ for a volume of size $n^3$). This is an improvement over the standard $O(n^3)$ complexity of processing every voxel independently if surface tracking is used [S1].
- **Memory Complexity:** Requires storage for 4 lookup tables of size 256 entries each. Each entry contains a list of triangles. The memory footprint is small and constant relative to the volume size [S1].
- **Strengths:** Guarantees topological correctness (closed 2-manifold) for binary volumes. No cracks or non-manifold edges. Compatible with standard Marching Cubes optimizations [S1].
- **Limitations:** Only applicable to binary volumes. Requires precomputation. The choice of connectedness couple affects the surface detail and topology [S1].

## Background
This method was introduced by [S1] as a topologically correct approach for extracting isosurfaces from binary volumetric images. It is based on the theory of digital surfaces and connectedness couples. The method has been implemented and validated with synthetic data and is used in the segmentation of medical images, specifically CT scans (e.g., 256x256x113 resolution) [S1].

## Sources
[S1] Topologically defined isosurfaces, https://doi.org/10.1007/3-540-62005-2_21, fetched 2026-10-27
