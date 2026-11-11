# Ambiguous Face Handling with Connected Components

Also called: White vs Blue component selection

## Overview
This method resolves topological ambiguities in Marching Cubes surface extraction by selecting specific connected components of the isosurface edges within a voxel cell. It prevents the generation of non-manifold edges that arise when standard lookup tables fail to provide a unique triangulation for ambiguous faces [S1]. Unlike general disambiguation strategies, it specifically leverages the connectivity of "white" and "blue" edge components to determine the correct surface patch topology based on the local configuration of ambiguous faces [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A voxel cell containing scalar field values at its vertices, identifying which faces are "ambiguous" (containing two vertices inside $\partial V$ interleaved with two vertices outside $\partial V$) [S1, S2].
*   **Output:** A set of surface patches (triangles) for the cell, ensuring manifold topology.

**Data Structures:**
*   **Edge Components:** The edges of the cube intersecting the isosurface form connected components. These are classified as "white" or "blue" based on the graph coloring of the cube's edge graph [S1].
*   **Ambiguity Count:** A count of ambiguous faces per cell.

**Steps:**
1.  **Identify Ambiguous Faces:** Determine which faces of the current voxel cell are ambiguous. An ambiguous face is defined as having two vertices inside the boundary $\partial V$ interleaved with two vertices outside $\partial V$ [S2].
2.  **Classify Cell Configuration:** Count the number of ambiguous faces in the cell and determine the connectivity of the blue components.
3.  **Select Component Strategy:** Apply the following rules based on the configuration [S1]:
    *   **Case 1: One ambiguous face per cell.** If the cell has only one ambiguous face (or shares one ambiguous face with a neighbor such that only one ambiguous face is effectively processed per cell), use **white connected components** to extract surface patches. Using blue components in this case would produce non-manifold edges [S1].
    *   **Case 2: Three ambiguous faces, multiple blue components.** If a cell has three ambiguous faces and contains more than one blue connected component, use the **blue connected components** to produce the surface patches [S1].
    *   **Case 3: Dual nodes from two cells.** If two adjacent cells each have three ambiguous faces and contain more than one blue component, use these configurations to produce **dual nodes** inside the cells. These dual nodes are then used to extract surface patches [S1].
4.  **Extract Patches:** Generate the triangles using the selected components or dual nodes. Locating dual nodes inside cells rather than on cell edges reduces surface waving and reconstruction artifacts, providing better capture of sharp edges [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_patches(cell):
    # Identify ambiguous faces on the cell boundary
    ambiguous_faces = get_ambiguous_faces(cell.vertices)
    num_ambiguous = len(ambiguous_faces)
    
    # Get connected components of the intersecting edges
    white_components = get_white_connected_components(cell.edges)
    blue_components = get_blue_connected_components(cell.edges)
    
    if num_ambiguous == 1:
        # Case 1: One ambiguous face per cell
        # Use white components to avoid non-manifold edges
        patches = extract_from_components(white_components)
    elif num_ambiguous == 3 and len(blue_components) > 1:
        # Case 2: Three ambiguous faces with multiple blue components
        # Use blue components to produce patches
        patches = extract_from_components(blue_components)
    else:
        # Check for Case 3: Two cells with three ambiguous faces
        # This requires neighbor information
        if is_dual_node_candidate(cell, ambiguous_faces, blue_components):
            # Produce dual nodes inside the cell
            dual_node = compute_dual_node(cell)
            patches = extract_from_dual_node(dual_node)
        else:
            # Fallback or other cases not specified in sources
            patches = extract_standard_marching_cubes(cell)
            
    return patches
```

**Parameters:**
*   **Curvature parameter $\lambda$:** Used in the broader algorithm context (e.g., DMC2 comparison) with a typical value of 0.9 [S1].
*   **Maximal octree depth:** Set to 7 in the referenced implementation [S1].

**Edge Cases and Pitfalls:**
*   **Non-manifold edges:** Using blue components when only one ambiguous face exists per cell results in non-manifold geometry [S1].
*   **Sharp edges:** Standard Marching Cubes may produce degenerated triangles or fail to capture sharp edges; this method's use of internal dual nodes helps mitigate this [S1].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The method produces smoother surfaces and avoids reconstruction artifacts, especially visible in sharp edges, compared to DMC2 [S1]. It is tested on mechanical volumetric objects of size $512 \times 512 \times 128$ voxels [S1].
*   **Strengths:** Avoids non-manifold edges; captures sharp edges better than methods placing dual nodes on edges; reduces surface waving.
*   **Limitations:** Requires identification of ambiguous faces and connected components; specific logic for dual node generation is implied but not fully detailed in the provided text.

## Background
This method is part of an adaptive surface extraction algorithm from anisotropic volumetric data, specifically contouring on generalized octrees [S1]. It addresses the limitations of standard Marching Cubes and variants like DMC2, which can produce degenerated triangles and struggle with sharp edges [S1]. The approach is detailed in the paper "Adaptive surface extraction from anisotropic volumetric data: contouring on generalized octrees" [S1].

## Sources
[S1] Adaptive surface extraction from anisotropic volumetric data: contouring on generalized octrees, https://doi.org/10.1007/s12243-013-0369-4, fetched 2026-10-29
