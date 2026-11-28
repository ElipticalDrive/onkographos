# Ambiguous Face Handling with Connected Components

Also called: White vs Blue component selection

## Overview
This method handles ambiguous faces in contouring to avoid non-manifold edges by selecting specific connected components (white vs. blue) based on the number of ambiguous faces per cell [S1]. It prevents non-manifold geometry by choosing white components when only one ambiguous face exists per cell, and blue components when a cell contains three ambiguous faces with multiple blue components [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Scalar field values at voxel vertices, identifying which faces are ambiguous [S1].
*   **Output:** A set of surface patches (triangles) for the cell, ensuring manifold topology [S1].

**Data Structures:**
*   **Edge Components:** The edges of the cube intersecting the isosurface form connected components [S1].
*   **Ambiguity Count:** A count of ambiguous faces per cell [S1].

**Steps:**
1.  **Identify Ambiguous Faces:** Determine which faces of the current voxel cell are ambiguous. An ambiguous face is defined as having two vertices inside the boundary $\partial V$ interleaved with two vertices outside $\partial V$ [G].
2.  **Classify Cell Configuration:** Count the number of ambiguous faces in the cell and determine the connectivity of the blue components [S1].
3.  **Select Component Strategy:** Apply the following rules based on the configuration [S1]:
    *   **Case 1: One ambiguous face per cell.** If the cell has only one ambiguous face, use **white connected components** to extract surface patches. Using blue components in this case would produce non-manifold edges [S1].
    *   **Case 2: Three ambiguous faces, multiple blue components.** If a cell has three ambiguous faces and contains more than one blue connected component, use the **blue connected components** to produce the surface patches [S1].
    *   **Case 3: Dual nodes from two cells.** If two adjacent cells each have three ambiguous faces and contain more than one blue component, use these configurations to produce **dual nodes** inside the cells [S1].
4.  **Extract Patches:** Generate the triangles using the selected components or dual nodes. Locating dual nodes inside cells rather than on cell edges reduces surface waving and reconstruction artifacts, providing better capture of sharp edges [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def extract_patches(cell):
    # Identify ambiguous faces: faces with two vertices inside boundary interleaved with two outside
    ambiguous_faces = [f for f in cell.faces if is_interleaved_boundary(f.vertices)]
    num_ambiguous = len(ambiguous_faces)

    # Compute connected components of intersecting edges
    white_components = compute_connected_components(cell.edges, type="white")
    blue_components = compute_connected_components(cell.edges, type="blue")

    if num_ambiguous == 1:
        # Case 1: One ambiguous face per cell -> use white components to avoid non-manifold edges
        patches = extract_from_components(white_components)
    elif num_ambiguous == 3 and len(blue_components) > 1:
        # Case 2: Three ambiguous faces with multiple blue components -> use blue components
        patches = extract_from_components(blue_components)
    else:
        # Case 3: Two adjacent cells with three ambiguous faces and multiple blue components
        # Sources do not specify exact neighbor coordination or dual node placement formula
        if has_neighbor_dual_candidate(cell, ambiguous_faces, blue_components):
            dual_node = compute_dual_node_inside_cell(cell)
            patches = extract_from_dual_node(dual_node)
        else:
            patches = extract_standard_marching_cubes(cell)
    return patches

def is_interleaved_boundary(vertices):
    # Standard check for alternating inside/outside signs on face vertices
    return (vertices[0] > 0 and vertices[1] < 0 and vertices[2] > 0 and vertices[3] < 0) or \
           (vertices[0] < 0 and vertices[1] > 0 and vertices[2] < 0 and vertices[3] > 0)

def compute_connected_components(edges, type):
    # Standard graph traversal to group intersecting edges into white or blue components
    # Implementation depends on specific edge graph coloring; sources do not provide details
    pass

def extract_from_components(components):
    # Generate triangles from selected connected components
    pass

def has_neighbor_dual_candidate(cell, ambiguous_faces, blue_components):
    # Check adjacent cells for Case 3 configuration
    # Sources do not specify exact neighbor lookup logic
    pass

def compute_dual_node_inside_cell(cell):
    # Place dual node inside the cell volume
    # Sources do not specify exact coordinate formula
    pass

def extract_from_dual_node(dual_node):
    # Generate triangles using the internal dual node
    pass

def extract_standard_marching_cubes(cell):
    # Fallback to standard lookup table triangulation
    pass
```

**Parameters:**
*   **Curvature parameter $\lambda$:** Used in the broader algorithm context with a typical value of $0.9$ [S1].
*   **Maximal octree depth:** Set to $7$ in the referenced implementation [S1].

**Edge Cases and Pitfalls:**
*   **Non-manifold edges:** Using blue components when only one ambiguous face exists per cell results in non-manifold geometry [S1].
*   **Sharp edges:** Standard Marching Cubes may produce degenerated triangles or fail to capture sharp edges; this method's use of internal dual nodes helps mitigate this [S1].

## Subtopics

### [Ambiguous Face Definition](7300-ambiguous-face-definition.md)
Defines the topological configuration on a voxel cell face where two vertices are inside the isosurface boundary and two are outside, arranged in an interleaved pattern. It identifies faces requiring special handling to avoid non-manifold edges during surface extraction. Choose this reference when implementing the face identification step. [Ambiguous Face Definition](7300-ambiguous-face-definition.md)

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The method produces smoother surfaces and avoids reconstruction artifacts, especially visible at sharp edges, compared to DMC2 on $512 \times 512 \times 128$ voxel data [S1].
*   **Strengths:** Avoids non-manifold edges; captures sharp edges better than methods placing dual nodes on edges; reduces surface waving [S1].
*   **Limitations:** Requires identification of ambiguous faces and connected components; specific logic for dual node generation and neighbor coordination is implied but not fully detailed in the provided text [S1].

## Background
The method originates from the paper "Adaptive surface extraction from anisotropic volumetric data: contouring on generalized octrees" [S1]. It addresses limitations of standard Marching Cubes and variants like DMC2, which can produce degenerated triangles and struggle with sharp edges [S1]. The approach is detailed in the referenced publication [S1].

## Sources
[S1] Adaptive surface extraction from anisotropic volumetric data: contouring on generalized octrees, https://doi.org/10.1007/s12243-013-0369-4, fetched 2026-10-29
