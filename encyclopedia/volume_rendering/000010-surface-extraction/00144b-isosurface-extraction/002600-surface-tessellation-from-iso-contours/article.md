# Surface Tessellation from Iso-Contours

Also called: Contour Reconstruction, Contour based reconstruction, Ribbon surface reconstruction, Triangulation of contours

## Overview
Surface Tessellation from Iso-Contours is a surface extraction method that represents iso-surfaces by generating polygonal meshes from stacks of 2D contours extracted from consecutive slices of parallel planes within a data volume [S1, unofficial][S2]. It solves the problem of reconstructing a continuous 3D surface from discrete 2D cross-sections by triangulating the "ribbon" of surface area between adjacent contour sets [S1, unofficial]. Unlike voxel-based methods like [Marching Cubes](30-isosurface-extraction.md) which operate on cell interiors, this method operates directly on the boundary curves, differing primarily in how it establishes topological correspondence and triangulates the space between contours [S1, unofficial][S2].

## How it works
**Inputs:** A sequence of 2D scalar slices (volumetric data) or pre-extracted 2D contour sets.
**Outputs:** A triangular mesh representing the iso-surface.

**Data Structures:**
1.  **Slice Data:** A 2D array of scalar values per slice, indexed by $(x, y, z)$ where $z$ is the slice index.
2.  **Contour Set $C_z$:** For each slice $z$, a list of closed polygonal chains (contours). Each contour is a list of vertices $(x, y)$.
3.  **Classification Tree:** A directed graph where nodes represent contours and edges represent topological relationships.
    *   **Nodes:** Each node $N_{z,i}$ corresponds to the $i$-th contour in slice $z$.
    *   **Edges:** A directed edge exists from a node in slice $z$ to a node in slice $z+1$ if the corresponding contours have a geometric correspondence.
4.  **Geometric Correspondence Map:** A mapping between vertices of contour $S$ in slice $z$ and vertices of contour(s) $P$ in slice $z+1$.

**Steps:**

1.  **Data Preparation:**
    *   Read data slices into memory [S2].
    *   Interpolate pixels to achieve structured slices to avoid pixilation artifacts [S2].
    *   Extract 2D contours for each slice using thresholding or iso-contour following routines [S1, unofficial][S2]. Let $S = \{S_0, S_1, \dots, S_m\}$ be the set of contours in slice $z$, and $P = \{P_0, P_1, \dots, P_n\}$ be the set of contours in slice $z+1$ [S2].

2.  **Topological Reconstruction (Classification Tree):**
    *   Build a classification tree where the number of layers equals the number of slices [S2].
    *   Each node on layer $z$ corresponds to a contour in slice $z$ [S2].
    *   Determine edges between nodes in adjacent layers by projecting the contours of adjacent slices into a 2D plane [S2].
    *   If contours intersect in the projection, an edge exists between the corresponding nodes [S2].
    *   *Note:* This process does not consider nested contours; if contour $c$ is nested in $b$, the relation between $a$ (previous slice) and $c$ is ignored [S2].

3.  **Geometric Correspondence Determination:**
    *   Classify the relationship between adjacent contours into four basic types: pole, cylinder, independent branch, and connected branch [S2].
    *   For each point $S_0$ in contour set $S$, select its two nearest points $P_0$ and $P_1$ from the adjacent contour set $P$ based on the determined geometric correspondence and the classification tree [S2].

4.  **Triangulation (Ribbon Construction):**
    *   Form a triangle $S_0 P_0 P_1$ for each point $S_0$ in set $S$ [S2].
    *   Repeat for all points in $S$ to construct the triangular patches of the surface between the two slices [S2].

**Alternative Triangulation Strategies:**
*   **Minimum Cost Path (Keppel):** Associate all possible triangular tiling arrangements with a directed graph. Search the graph to identify a minimum cost path corresponding to a surface that maximizes the enclosed volume [S1, unofficial].
*   **Minimum Area (Fuchs et al.):** Minimize the surface area of the tessellated ribbon [S1, unofficial]. Limited to simply branching structures [S1, unofficial].
*   **Shortest Diagonal (Christiansen & Sederburg):** Map contours onto a unit square. Select triangulation based on the shortest diagonal. Ambiguous branching is resolved by user interaction [S1, unofficial].
*   **Maximally Smooth (Xu & Lu):** Treat reconstruction as a functional minimization problem solved by a partial differential equation method, constraining the sum of curvatures to be minimum [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def surface_tessellation_from_iso_contours(slices, threshold):
    """
    Reconstructs a triangular mesh from 2D slices using contour tessellation.
    
    Args:
        slices: List of 2D numpy arrays representing scalar fields for each z-slice.
        threshold: Scalar value to extract the iso-contour.
        
    Returns:
        vertices: Array of 3D vertices (x, y, z).
        triangles: Array of triangle indices.
    """
    vertices = []
    triangles = []
    
    # Step 1: Extract contours for each slice
    contours = []
    for z, slice_data in enumerate(slices):
        # Extract 2D contours at the given threshold
        # Returns a list of closed polygonal chains, each chain is a list of (x, y)
        contour_list = extract_2d_contours(slice_data, threshold)
        contours.append(contour_list)
        
    # Step 2: Build Classification Tree and Determine Correspondences
    # This step determines which contours in slice z connect to which in slice z+1
    correspondences = [] # List of lists, where correspondences[z] contains mappings for slice z to z+1
    
    for z in range(len(contours) - 1):
        current_contours = contours[z]
        next_contours = contours[z+1]
        
        # Project contours to 2D plane (already 2D) and check intersections
        # Build adjacency between contours
        adjacency = build_contour_adjacency(current_contours, next_contours)
        
        # Determine geometric correspondence for each vertex
        # For each point in current_contours, find nearest points in next_contours
        vertex_mappings = []
        for contour_idx, current_contour in enumerate(current_contours):
            # Find the corresponding contour(s) in next_contours based on adjacency
            # If multiple, use classification tree logic (pole, cylinder, etc.)
            # For simplicity, assume a single corresponding contour or split logic
            corresponding_contour = get_corresponding_contour(current_contour, next_contours, adjacency)
            
            if corresponding_contour is None:
                vertex_mappings.append([])
                continue
                
            mappings_for_contour = []
            for point in current_contour:
                # Find two nearest points in the corresponding contour
                p0, p1 = find_two_nearest_points(point, corresponding_contour)
                if p0 is not None and p1 is not None:
                    mappings_for_contour.append((point, p0, p1))
                else:
                    mappings_for_contour.append(None)
            vertex_mappings.append(mappings_for_contour)
        correspondences.append(vertex_mappings)

    # Step 3: Generate Vertices and Triangles
    # Vertices are stored with their z-coordinate
    for z, contour_list in enumerate(contours):
        for contour in contour_list:
            for point in contour:
                # Store vertex with z-index
                vertices.append((point[0], point[1], z))
                
    # Generate triangles from correspondences
    for z, vertex_mappings in enumerate(correspondences):
        # vertex_mappings[z] corresponds to contours in slice z connecting to slice z+1
        for contour_idx, mappings in enumerate(vertex_mappings):
            if mappings is None:
                continue
            for i, mapping in enumerate(mappings):
                if mapping is None:
                    continue
                s0, p0, p1 = mapping
                
                # Find indices of these points in the global vertices list
                # Note: In a real implementation, we need a lookup from (x,y,z) to vertex index
                idx_s0 = get_vertex_index(s0[0], s0[1], z)
                idx_p0 = get_vertex_index(p0[0], p0[1], z+1)
                idx_p1 = get_vertex_index(p1[0], p1[1], z+1)
                
                # Form triangle
                triangles.append((idx_s0, idx_p0, idx_p1))
                
    return vertices, triangles

def extract_2d_contours(slice_data, threshold):
    """
    Extracts closed contours from a 2D slice using marching squares or similar.
    Returns list of lists of (x, y) tuples.
    """
    # Implementation depends on specific contour extraction algorithm
    # This is a placeholder for the actual extraction logic
    pass

def build_contour_adjacency(current_contours, next_contours):
    """
    Determines which contours in current_contours have geometric relations
    with contours in next_contours based on intersection/proximity.
    """
    # Project and check intersections
    adjacency = {i: [] for i in range(len(current_contours))}
    for i, c1 in enumerate(current_contours):
        for j, c2 in enumerate(next_contours):
            if contours_intersect(c1, c2):
                adjacency[i].append(j)
    return adjacency

def contours_intersect(c1, c2):
    """
    Checks if two 2D contours intersect or are close enough to be considered related.
    """
    # Simple distance check or intersection test
    pass

def get_corresponding_contour(current_contour, next_contours, adjacency):
    """
    Selects the specific contour from next_contours that corresponds to current_contour.
    Uses classification tree logic (pole, cylinder, etc.).
    """
    # If only one adjacent contour, return it
    if len(adjacency[current_contour.index]) == 1:
        return next_contours[adjacency[current_contour.index][0]]
    # If multiple, use geometric correspondence rules
    # This is complex and often requires user interaction or heuristic
    pass

def find_two_nearest_points(point, contour):
    """
    Finds the two nearest points on a contour to a given point.
    """
    distances = [(p, dist(point, p)) for p in contour]
    distances.sort(key=lambda x: x[1])
    if len(distances) >= 2:
        return distances[0][0], distances[1][0]
    return None, None

def get_vertex_index(x, y, z):
    """
    Lookup vertex index from coordinates.
    """
    # In practice, use a hash map or spatial index
    pass
```

## Subtopics

### [Contour Spectrum](11703-contour-spectrum.md)
A data-driven technique that assimilates Surface Fitting resources to discover isosurfaces after an initial specification (e.g., intensity and gradient magnitude). It differs by using data-driven assimilation rather than fixed geometric correspondence rules.

## Operating characteristics
**Time Complexity:** Depends on the number of contours and vertices. Building the classification tree is $O(N^2)$ where $N$ is the number of contours if naive intersection checking is used. Triangulation is linear in the number of vertices.
**Memory Complexity:** Proportional to the number of contours and vertices. The classification tree adds overhead.
**Strengths:**
*   Efficient for data that is naturally organized in slices (e.g., medical CT/MRI) [S1, unofficial].
*   Can handle complex topologies if the correspondence rules are robust [S2].
*   Allows for manual intervention to resolve ambiguities [S1, unofficial].
**Limitations:**
*   Ambiguities arise when more than one contour exists on a slice, making it difficult to determine connections [S1, unofficial].
*   The edge determination process does not consider nested contours, leading to topological errors in such cases [S2].
*   Requires interpolation to avoid pixilation, adding computational cost [S2].
*   Limited to simply branching structures for some minimum-area algorithms [S1, unofficial].
**When to use:** When the input data is already segmented into contours or when slice-based processing is preferred.
**When not to use:** When nested contours are common and automatic handling is required, or when a fully volumetric approach is more suitable.

## Background
Surface Tessellation from Iso-Contours is one of the first methods used to represent iso-surfaces, dating back to the early days of volumetric rendering [S1, unofficial]. Keppel was one of the first to devise a method to reconstruct the ribbon surface defined by two consecutive contours [S1, unofficial]. Fuchs et al. presented a solution minimizing surface area [S1, unofficial]. Christiansen and Sederburg achieved similar results by mapping contours to a unit square [S1, unofficial]. Xu and Lu solved the problem using a partial differential equation method to maximize smoothness [S1, unofficial]. The THREA04 system, developed by Mazziotta et al. in 1976, was capable of displaying 3D images from 20 sections using threshold segmentation and automatic boundary detection [S1, unofficial]. The method is widely used in medical imaging for 3D reconstruction from CT or MRI slices [S2].

## Sources
[S1] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
[S2] Comparison of Surface Reconstruction Method for CT based on VTK, https://www.atlantis-press.com/proceedings/icaiees-13/10839, fetched 2026-10-28
