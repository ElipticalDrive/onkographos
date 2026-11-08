# Ambiguous Face Handling with Connected Components

Also called: White vs Blue component selection

## Overview
This method resolves topological ambiguities in isosurface extraction by selecting specific connected components of edges (classified as "white" or "blue") based on the count of ambiguous faces within a cell. It prevents the generation of non-manifold edges, which are invalid in standard manifold surface representations. It differs from general disambiguation by explicitly using graph coloring (white/blue) to determine patch connectivity rather than relying solely on lookup table indices.

## How it works
**Inputs:**
- A voxel grid or octree structure.
- Scalar field values at vertices.
- Identification of ambiguous faces (faces with two interior and two exterior vertices interleaved).
- Connected components of edges within each cell, classified by color (white/blue).

**Outputs:**
- A manifold surface mesh composed of disjoint patches.

**Data Structures:**
- **Cell State:** For each cell, store the number of ambiguous faces ($N_{amb}$).
- **Component Classification:** Each connected component of edges in a cell is labeled as "white" or "blue".
- **Dual Nodes:** Optional nodes placed inside cells for sharp edge capture.

**Steps:**
1. **Identify Ambiguous Faces:** Detect faces where the vertex signs alternate (e.g., $+ - + -$). This creates ambiguity in edge intersection placement [S1].
2. **Classify Components:** Decompose the edge graph within each cell into connected components. Label these components as white or blue based on the specific coloring scheme used by the parent method (Disjoint Patch Decomposition) [S1].
3. **Select Components Based on $N_{amb}$:**
   - **Case 1: One Ambiguous Face per Cell ($N_{amb}=1$):**
     - If two cells share an ambiguous face, using blue components may produce non-manifold edges.
     - **Action:** Use **white** connected components to extract surface patches [S1].
   - **Case 2: Three Ambiguous Faces in One Cell ($N_{amb}=3$):**
     - If the cell contains more than one blue connected component:
     - **Action:** Use **blue** components to produce patches [S1].
   - **Case 3: Two Cells with Three Ambiguous Faces Each:**
     - If two adjacent cells each have three ambiguous faces and contain more than one blue component:
     - **Action:** Use these configurations to produce **dual nodes** inside the cells. Extract surface patches connected to these dual nodes [S1].
4. **Extract Patches:** Generate triangles from the selected components. Dual nodes allow for smoother surfaces and better capture of sharp edges compared to edge-based nodes [S1].

**Parameters:**
- Curvature parameter $\lambda$: Typical value 0.9 [S1].
- Maximal octree depth: Typical value 7 [S1].

**Edge Cases:**
- **Non-manifold edges:** Avoided by strictly following the white/blue selection rules for shared ambiguous faces [S1].
- **Degenerated triangles:** May occur if component selection is incorrect; DMC2 (a related method) is noted to produce more degenerated triangles in sharp edges [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def handle_ambiguous_faces(cell_list):
    for cell in cell_list:
        # Identify ambiguous faces on this cell
        ambiguous_faces = get_ambiguous_faces(cell)
        n_amb = len(ambiguous_faces)
        
        # Get connected components of edges, classified as white or blue
        white_components = get_white_components(cell)
        blue_components = get_blue_components(cell)
        
        if n_amb == 1:
            # Case: One ambiguous face per cell
            # Check if shared with neighbor (implicit in global topology)
            # Use white components to avoid non-manifold edges
            extract_patches(cell, white_components)
            
        elif n_amb == 3:
            # Case: Three ambiguous faces in one cell
            if len(blue_components) > 1:
                # Use blue components to produce patches
                extract_patches(cell, blue_components)
            else:
                # Fallback or other handling not specified in sources
                pass
                
        # Case: Two cells with three ambiguous faces each
        # This requires global check between adjacent cells
        # If cell and its neighbor both have n_amb=3 and >1 blue components:
        #   create_dual_nodes(cell)
        #   extract_patches(cell, dual_nodes)
```

## Subtopics

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Produces smoother surfaces with fewer reconstruction artifacts, especially at sharp edges, compared to DMC2 [S1].
- **Strengths:** Avoids non-manifold edges; dual nodes provide freedom to capture sharp edges; less waving than edge-based dual nodes [S1].
- **Limitations:** DMC2 (comparison) has problems capturing sharp edges and produces more degenerated triangles [S1].

## Background
- **Origin:** Described in the context of adaptive surface extraction from anisotropic volumetric data using generalized octrees [S1].
- **Development:** The method is part of a broader approach to handle topological ambiguities in Marching Cubes by decomposing surfaces into disjoint patches. It specifically addresses the "white vs blue" component selection dilemma in ambiguous face configurations [S1].
- **Usage:** Applied to mechanical volumetric objects (e.g., 512 × 512 × 128 voxels) to improve surface quality [S1].

## Sources
[S1] Adaptive surface extraction from anisotropic volumetric data: contouring on generalized octrees, https://doi.org/10.1007/s12243-013-0369-4, fetched 2026-10-29
