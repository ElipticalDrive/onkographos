# Flexible Isosurface


## Overview
Flexible Isosurfaces is a visualization technique that extends the standard isosurface concept by allowing individual contours to be displayed, manipulated, and annotated at different isovalues [S1]. Unlike conventional methods that extract a single surface for a global threshold, this method leverages the contour tree to manage a set of independent contours, each potentially having a distinct isovalue [S1]. It sets itself apart from siblings like [Surface Tessellation from Iso-Contours](3696-surface-tessellation-from-iso-contours.md) by treating the contour tree as a direct control structure for annotation and dynamic topology management, rather than just a data structure for extraction.

## How it works
The method operates on a discretized scalar field, building on the shared principles of [Contour and Line Extraction](22941-contour-and-line-extraction.md). It relies on a 1-1 correspondence between contours, isosurface seeds, and superarcs of the contour tree [S1].

**Inputs and Outputs:**
*   **Input:** A scalar field $\phi(\mathbf{x})$, a contour tree $T$ (constructed via the algorithm of Carr, Snoeyink & Axen [S1]), and a set of user-defined active contours.
*   **Output:** A set of rendered contour geometries, each associated with a specific superarc and isovalue.

**Data Structures:**
1.  **Contour Tree ($T$):** Nodes represent critical points (minima, maxima, saddles); edges (superarcs) represent intervals between critical points.
2.  **Path Seeds:** Each superarc in $T$ stores a "path seed," which is a single cell (or point) that uniquely identifies the contour corresponding to that superarc [S1].
3.  **Active Contour List:** A list of tuples `(superarc_id, isovalue)` representing the currently displayed flexible isosurface [S1].
4.  **Display List ID:** Stored on each superarc, referencing an OpenGL display list containing the extracted geometry [S1].
5.  **Color Palette:** A small set of colors used to visually distinguish contours [S1].

**Algorithm Steps:**

1.  **Construct Contour Tree and Path Seeds:**
    *   Compute the contour tree for the scalar field.
    *   During construction, identify path seeds. For each edge in the join/split tree, transfer the path seed to the corresponding superarc in the contour tree [S1].
    *   This step adds constant cost per edge, maintaining the asymptotic complexity of the contour tree algorithm [S1].

2.  **Define Flexible Isosurface:**
    *   The user selects a set of active superarcs and assigns an isovalue to each [S1].
    *   Store these annotations on the contour tree. Specifically, mark which superarcs are active and store the associated isovalue for each [S1].
    *   *Note:* The isovalue stored on a superarc may differ from the natural isovalue range of that superarc if the user explicitly overrides it, though typically the superarc defines the topological component [S1].

3.  **Color Assignment:**
    *   Assign a color to each superarc to distinguish contours visually [S1].
    *   The color is determined by:
        $$ \text{Color} = \text{Palette}[\text{superarc\_ID} \pmod{\text{NumColors}}] $$
    *   Where `Palette` is a small set of easily distinguished colors [S1].

4.  **Extract Contour Geometry:**
    *   For each active superarc in the flexible isosurface:
        a. Retrieve the path seed stored on the superarc [S1].
        b. Generate a single seed cell for the contour using this path seed [S1].
        c. Extract the contour geometry using the continuation method of Wyvill, McPheeters & Wyvill [S1]. This method traces the contour starting from the seed cell.
        d. Store the resulting geometry in an OpenGL display list for efficient rendering [S1].
        e. Store the OpenGL display list ID on the superarc for future reference [S1].

5.  **Render:**
    *   Render each active contour using its associated OpenGL display list and the color assigned in Step 3 [S1].
    *   Optionally, render the contour tree (contour spectrum) with colored tags on active superarcs to allow visual association between the 3D contours and the topological structure [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flexible_isosurface_extraction(scalar_field, contour_tree, active_contours):
    """
    scalar_field: The input 3D scalar data.
    contour_tree: The pre-computed contour tree with path seeds attached to superarcs.
    active_contours: A list of tuples (superarc_id, isovalue) representing user selection.
    """
    
    # Step 3: Color Assignment (Pre-computed or cached)
    palette = [RED, GREEN, BLUE, YELLOW, CYAN, MAGENTA] # Small palette
    num_colors = len(palette)
    
    for superarc in contour_tree.superarcs:
        superarc.color = palette[superarc.id % num_colors]

    # Step 4: Extract and Store Geometry
    for superarc_id, isovalue in active_contours:
        superarc = contour_tree.get_superarc(superarc_id)
        
        # Check if already extracted (optimization)
        if superarc.display_list_id is not None:
            continue
            
        # Step 4a: Retrieve Path Seed
        path_seed = superarc.path_seed
        
        # Step 4b: Generate Seed Cell
        # The path seed is typically a grid cell or vertex.
        # Generate a seed cell structure suitable for the continuation method.
        seed_cell = generate_seed_cell(path_seed, scalar_field)
        
        # Step 4c: Extract Contour using Continuation Method
        # Wyvill, McPheeters & Wyvill method
        contour_geometry = wyvill_continuation_method(seed_cell, scalar_field)
        
        # Step 4d: Store in OpenGL Display List
        # Implementation detail: Create GL display list
        display_list_id = create_opengl_display_list(contour_geometry)
        
        # Step 4e: Store ID on Superarc
        superarc.display_list_id = display_list_id

    # Step 5: Render
    for superarc_id, isovalue in active_contours:
        superarc = contour_tree.get_superarc(superarc_id)
        if superarc.display_list_id is not None:
            render_opengl_display_list(superarc.display_list_id, superarc.color)

def generate_seed_cell(path_seed, scalar_field):
    """
    Converts a path seed (e.g., a grid index) into a seed cell for continuation.
    [G] Implementation detail: Typically returns the grid cell containing the seed point
    and the scalar value at that point to initialize the marching/continuation algorithm.
    """
    return {
        'cell_index': path_seed.cell_index,
        'scalar_value': scalar_field[path_seed.cell_index],
        'position': path_seed.position
    }

def wyvill_continuation_method(seed_cell, scalar_field):
    """
    Extracts the contour geometry starting from a seed cell.
    [G] This function implements the surface continuation algorithm.
    It traces the zero-crossing of the isosurface field from the seed.
    """
    # [G] Standard continuation logic:
    # 1. Find intersection points on the edges of the seed cell.
    # 2. Connect these points to form initial segments.
    # 3. Propagate the contour to adjacent cells by finding new intersection points.
    # 4. Continue until the contour closes or terminates.
    # The exact implementation depends on the specific grid topology (e.g., cubic cells).
    # For a cubic grid, this involves checking face intersections and linking segments.
    pass # Placeholder for the specific geometric tracing logic defined by Wyvill et al.
```

**Edge Cases and Pitfalls:**
*   **Contour Tree Layout:** Fixing the y-coordinate of the contour tree vertices by isovalue can cause edges to cross, making the tree hard to display or interpret [S1]. This is a visualization pitfall, not an extraction error, but affects the interactive interface [S1].
*   **Color Ambiguity:** If the number of active contours exceeds the palette size, colors will repeat. The method relies on the user distinguishing contours by position in the contour tree or other means if colors collide [S1].
*   **Topology Changes:** Changing isovales that cause topological changes (merging/splitting) will affect the contour tree structure. The system must handle the update of path seeds and active contours accordingly [S1].

## Operating characteristics
*   **Time Complexity:** The cost of extracting path seeds is constant per edge of the join/split tree and constant per transfer to the contour tree. The asymptotic cost remains $O(n \log n + t \alpha(t))$, where $n$ is the number of grid points and $t$ is the number of critical points [S1]. This is the same complexity as the contour tree algorithm itself [S1].
*   **Memory Complexity:** Path seeds are cheaper to store than minimal seed sets of van Kreveld et al., especially in dimensions higher than two [S1]. Each superarc stores one path seed and one display list ID [S1].
*   **Strengths:**
    *   Allows individual contours to be displayed at different isovalues [S1].
    *   Reduces storage by using one seed per component [S1].
    *   Simplifies isosurface extraction by guaranteeing a 1-1 correspondence between contours, seeds, and superarcs [S1].
    *   Enables visual annotation and tracking of contours as isovalue varies [S1].
*   **Limitations:**
    *   Requires the construction of a contour tree, which can be complex for large datasets.
    *   Visualization of the contour tree itself can be difficult due to edge crossings [S1].
    *   Relies on OpenGL display lists, which may not be suitable for all modern rendering pipelines (though the geometric extraction is general) [S1].

## Background
The Flexible Isosurface method was introduced by Carr, Snoeyink, and Axen in 2003 [S1]. It extends the contour spectrum of Bajaj, Pascucci, and Schikore [S1]. The method builds upon the contour tree algorithm of Carr, Snoeyink, and Axen [S1]. It is used in exploratory visualization to allow users to interactively select and manipulate individual components of an isosurface [S1]. The largest contour segmentation, where separate surfaces are generated for each local maximum, is a special case of this method [S1].

## Sources
[S1] Path Seeds and Flexible Isosurfaces Using Topology for Exploratory Visualization, https://doi.org/10.2312/vissym/vissym03/049-058, fetched 2026-10-30
