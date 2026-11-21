# Rule-based Marching Method


## Overview
The rule-based marching method is a topology generation technique for volume fraction data that derives partial vertex labeling and uses lookup tables to determine material interface configurations in mixed cells [S1, unofficial]. Unlike methods that force vertex labels via volume averaging or cast data onto dual meshes, this approach implies labels through a small set of logical rules, leaving topology uninitialized when the labeling is ambiguous or insufficient [S1, unofficial]. It is specifically designed to handle multiple materials and complex features like thin filaments by allowing multiple intersections per edge, which standard binary marching cubes do not support [S1, unofficial].

## How it works
The method operates on a mesh where vertices are initially unlabeled because volume fraction data lacks explicit vertex information [S1, unofficial]. It proceeds by deriving a partial labeling of vertices and edges, then using segmentation lookup tables to generate cell-level interface topologies.

### Inputs and Outputs
*   **Input**: A mesh structure (vertices, edges, cells) and volume fraction data $V$ for each cell, specifying the fraction of each material present.
*   **Output**: A set of cell-level interface topologies (surface fragments) for cells where the topology can be unambiguously determined. Cells that fail the rules remain uninitialized.

### Data Structures
1.  **Vertex Label Map $L$**: A map from vertex index $v$ to a material label or `undefined`.
    *   Domain: $\{1, \dots, n\}$ for $n$ materials, or `undefined`.
2.  **Edge Intersection Count $\chi$**: A map from edge index $e$ to an integer representing the number of interface crossings.
    *   Values: $0$, $1$, or $\ge 2$.
3.  **Segmentation Case Tables**: Precomputed lookup tables for 2D and 3D cells.
    *   These tables map a cell's vertex labeling configuration to a set of surface fragments.
    *   Unlike standard Marching Cubes, these tables support multiple intersections per edge and may indicate "ambiguity" or "residual material" states.

### Algorithm Steps

**1. Initialize Vertex Labels**
Initialize $L(v) = \text{undefined}$ for all vertices $v$ in the mesh [S1, unofficial].

**2. Label Vertices**
For each vertex $v$, evaluate the following rules in order. If a rule assigns a label, it is set; otherwise, it remains `undefined`.

*   **Pure Cells Rule**: Let $P_v$ be the set of pure cells (cells containing only one material) neighboring $v$.
    *   If $P_v$ is non-empty and all cells in $P_v$ are of the same material $i$, then $L(v) = i$.
    *   If $P_v$ contains cells of different materials, $L(v)$ remains `undefined`.
    *   If $P_v$ is empty, proceed to the next rule.

*   **Neighborhood Rule**: Let $M_v$ be the set of materials common to all (pure and mixed) cells neighboring $v$.
    *   If $|M_v| = 1$, let $M_v = \{i\}$. Then $L(v) = i$.
    *   Otherwise, $L(v)$ remains `undefined`.

**3. Label Edges and Determine Intersections**
For each edge $e$ connecting vertices $v_1$ and $v_2$:

*   **Initial Intersection Estimate**:
    *   If $L(v_1) = L(v_2)$, then $\chi(e) \in \{0, \ge 2\}$.
    *   If $L(v_1) \neq L(v_2)$, then $\chi(e) \ge 1$.
    *   If either $L(v_1)$ or $L(v_2)$ is `undefined`, $\chi(e)$ is not yet determined.

*   **Pure Cells Rule**: Let $C_e$ be the set of cells sharing edge $e$.
    *   If any cell in $C_e$ is pure (contains only one material), then $\chi(e) = 0$.

*   **Neighborhood Rule**: Let $M_e$ be the set of materials common to all cells in $C_e$.
    *   If $|M_e| = 1$, then $\chi(e) = 0$.

**4. Generate Cell Topology**
For each cell $c$:

*   **Check Vertex Labeling**: If any vertex $v$ of $c$ has $L(v) = \text{undefined}$, leave $c$'s topology uninitialized [S1, unofficial].
*   **Lookup**: If all vertices are labeled, perform a lookup in the appropriate segmentation case table (2D or 3D) using the vertex labels.
*   **Check Ambiguity**: If the case table indicates ambiguity (multiple valid segmentations for the labeling), leave $c$'s topology uninitialized [S1, unofficial].
*   **Check Residual Material**:
    *   Let $M_c$ be the set of materials with nonzero volume fractions in cell $c$.
    *   Let $M'_c$ be the set of material labels assigned to the vertices of $c$.
    *   If $M_c \neq M'_c$, the cell has residual material (internal pockets or multiple intersections not captured by the vertex labels). Leave $c$'s topology uninitialized [S1, unofficial].
*   **Assign Topology**: If all checks pass, assign the surface fragments returned by the case table to cell $c$.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rule_based_marching(mesh, volume_fractions):
    """
    mesh: object with methods get_vertices(), get_edges(), get_cells(), 
          get_neighbors(vertex), get_neighbors(edge), get_neighbors(cell)
    volume_fractions: dict mapping cell_id -> dict mapping material_id -> float
    """
    # Step 1: Initialize Vertex Labels
    L = {v: None for v in mesh.get_vertices()}
    
    # Step 2: Label Vertices
    for v in mesh.get_vertices():
        # Pure Cells Rule
        pure_neighbors = [c for c in mesh.get_neighbors(v) if is_pure(c, volume_fractions)]
        if pure_neighbors:
            materials = set(volume_fractions[c].keys() for c in pure_neighbors)
            if len(materials) == 1:
                L[v] = materials.pop()
            else:
                L[v] = None # Conflicting pure cells
        else:
            # Neighborhood Rule
            all_neighbors = mesh.get_neighbors(v)
            common_materials = None
            for c in all_neighbors:
                mat_set = set(volume_fractions[c].keys())
                if common_materials is None:
                    common_materials = mat_set
                else:
                    common_materials &= mat_set
            
            if common_materials and len(common_materials) == 1:
                L[v] = common_materials.pop()
            else:
                L[v] = None

    # Step 3: Label Edges
    chi = {} # Edge intersection count
    for e in mesh.get_edges():
        v1, v2 = e.vertices
        l1 = L[v1]
        l2 = L[v2]
        
        # Initial estimate based on vertex labels
        if l1 is not None and l2 is not None:
            if l1 == l2:
                chi[e] = "0_or_ge2"
            else:
                chi[e] = "ge1"
        else:
            chi[e] = "unknown"
            
        # Pure Cells Rule for Edge
        cells_sharing_e = mesh.get_neighbors(e)
        if any(is_pure(c, volume_fractions) for c in cells_sharing_e):
            chi[e] = 0
            continue
            
        # Neighborhood Rule for Edge
        common_mat = None
        for c in cells_sharing_e:
            mat_set = set(volume_fractions[c].keys())
            if common_mat is None:
                common_mat = mat_set
            else:
                common_mat &= mat_set
        
        if common_mat and len(common_mat) == 1:
            chi[e] = 0
        else:
            # If chi[e] was "unknown" or "ge1" and rules didn't force 0, it stays as is
            # Note: The source implies these rules determine "to the extent possible"
            pass

    # Step 4: Generate Cell Topology
    topologies = {}
    for c in mesh.get_cells():
        vertices = c.vertices
        labels = [L[v] for v in vertices]
        
        # Check for undefined labels
        if None in labels:
            continue
            
        # Lookup in segmentation table
        # Note: The specific table structure is implementation-dependent
        # but must support multiple intersections per edge.
        result = lookup_segmentation_table(c.dimension, labels)
        
        if result is None or result.is_ambiguous:
            continue
            
        # Check residual material
        cell_materials = set(volume_fractions[c].keys())
        vertex_materials = set(labels)
        
        if cell_materials != vertex_materials:
            continue
            
        topologies[c] = result.fragments
        
    return topologies

def is_pure(cell, volume_fractions):
    """Returns True if cell contains only one material with non-zero fraction."""
    mats = [m for m, f in volume_fractions[cell].items() if f > 0]
    return len(mats) == 1
```

### Parameters and Edge Cases
*   **Ambiguity**: If the segmentation case table indicates multiple valid segmentations for a given labeling, the cell topology is left uninitialized [S1, unofficial].
*   **Residual Material**: If the set of materials in the cell does not match the set of vertex labels, the topology is left uninitialized [S1, unofficial]. This handles cases with internal pockets or multiple intersections.
*   **Multiple Intersections**: The method supports $\chi \ge 2$ intersections per edge, unlike standard Marching Cubes which assumes $\chi \in \{0, 1\}$. This requires custom 2D and 3D case tables [S1, unofficial].
*   **Uninitialized Cells**: Cells that fail any rule are skipped. A subsequent method (e.g., discretized boundary method) may handle these [S1, unofficial].

## Operating characteristics
*   **Complexity**: The vertex and edge labeling steps are linear in the number of vertices and edges, respectively. The cell topology generation is constant time per cell (lookup table access) provided the labeling is complete.
*   **Strengths**: It avoids the artifacts of forced labeling (e.g., volume averaging errors) and handles complex multi-material interfaces with thin filaments better than standard binary methods [S1, unofficial].
*   **Limitations**: It leaves many cells uninitialized, requiring a fallback method. It is specific to volume fraction data and not directly applicable to general scalar fields without adaptation [S1, unofficial].
*   **When to use**: When processing volume fraction data for multi-material reconstruction where interface accuracy is critical and standard marching cubes produce cracks or incorrect topology due to ambiguity or multiple intersections [S1, unofficial].

## Background
The rule-based marching method was developed for the visualization of volume fraction data, particularly in the context of Multi-Interface Reconstruction (MIR) [S1, unofficial]. It was proposed as an improvement over previous approaches such as those by Bonnell et al., who cast volume fraction data onto a dual mesh, and Meredith, who forced vertex labels through volume averaging [S1, unofficial]. The method draws inspiration from the ambiguity handling in Marching Cubes (Nielson and Hamann) but extends it to support multiple intersections per edge [S1, unofficial].

## Sources
[S1] Scientific visualization techniques for volume fraction data and function fields, https://escholarship.org/uc/item/41x9x1gm, fetched 2026-10-27 (unofficial)
