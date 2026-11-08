# Direct Manipulation Widgets for Multidimensional TF

Also called: Interactive TF Tools, Spatial-Transfer Function Interaction

## Overview
This method provides a suite of interaction tools to bridge the conceptual gap between the spatial domain (3D geometry/data) and the abstract transfer-function domain, allowing simultaneous interaction in both [S1, book]. It solves the problem where it is difficult to determine which regions of a multidimensional transfer function correspond to features of interest, especially when those regions are small [S1, book]. It sets itself apart from manual parameter tuning by enabling users to probe spatial data and immediately visualize corresponding values in the transfer-function domain, facilitating a natural process of exploration, specification, and refinement [S1, book].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume dataset, a current transfer function (TF), and user interactions via mouse/keyboard.
*   **Output:** A modified transfer function that highlights user-selected features.

**Data Structures:**
*   **Volume Data:** 3D grid of multivariate values (e.g., scalar value, 1st derivative, 2nd derivative).
*   **Transfer Function:** A mapping from input data space to output color/opacity.
*   **Clipping Plane:** A geometric plane defined by normal vector $\mathbf{n}$ and distance $d$ from origin, used to slice the volume.

**Steps:**
1.  **Initialization:** The system presents a volume rendering using a predetermined or default transfer function [S1, book].
2.  **Exploration Phase:**
    *   The user moves a clipping plane through the volume [S1, book].
    *   A slice of the original data is mapped to the clipping plane for inspection [S1, book].
3.  **Probing Phase:**
    *   The user selects a sample position in the spatial domain [S1, book].
    *   The system retrieves the values at the probe point and its neighborhood [S1, book].
    *   These values are visualized in the transfer-function domain [S1, book].
    *   This feedback allows the user to identify regions in the TF corresponding to features of interest [S1, book].
4.  **Specification Phase:**
    *   The user creates a rough draft of the desired TF [S1, book].
    *   The user adds opacity to regions in the TF identified during probing [S1, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def direct_manipulation_workflow(volume, initial_tf):
    """
    Main loop for direct manipulation of multidimensional TFs.
    """
    # Initialize clipping plane at a default position
    clipping_plane = ClippingPlane(normal=(0, 0, 1), distance=0.5)
    
    # Render initial view
    render(volume, initial_tf, clipping_plane)
    
    while not user_quit():
        interaction = get_user_input()
        
        if interaction.type == "MOVE_CLIP":
            # Update clipping plane based on mouse drag
            clipping_plane.update(interaction.delta)
            # Map slice of original data to clipping plane
            slice_data = extract_slice(volume, clipping_plane)
            render(volume, initial_tf, clipping_plane, slice_overlay=slice_data)
            
        elif interaction.type == "PROBE":
            # Get spatial coordinates from mouse click
            spatial_pos = interaction.position
            # Retrieve values at point and neighborhood
            # Note: Neighborhood size is implementation-specific [G]
            neighborhood_values = get_neighborhood_values(volume, spatial_pos, radius=1)
            
            # Map values to TF domain for visualization
            tf_visualization = map_to_tf_domain(neighborhood_values, initial_tf)
            display_tf_feedback(tf_visualization)
            
        elif interaction.type == "EDIT_TF":
            # User selects region in TF domain
            selected_tf_region = interaction.tf_selection
            # Add opacity to the selected region
            modified_tf = add_opacity_to_region(initial_tf, selected_tf_region)
            initial_tf = modified_tf
            render(volume, initial_tf, clipping_plane)
```

**Parameters:**
*   **Clipping Plane Movement:** Controlled by user input.
*   **Neighborhood Radius:** Not given in the sources. [G] A small integer radius (e.g., 1-3 voxels) is typical for sampling local context.

**Edge Cases:**
*   **Small Feature Regions:** The method specifically addresses the difficulty of identifying small regions in the TF [S1, book].
*   **Multivariate Complexity:** Since values are not scalar (e.g., opacity and three colors), the mapping to the TF domain must handle multiple dimensions [S1, book].

## Subtopics

### [Multivariate Transfer Function Design based on User-Selected Samples](9483-multivariate-transfer-function-design-based-on-user-selected.md)
Automates TF creation from spatial samples using KDE, bridging spatial and TF domains without manual parameter adjustment.

## Operating characteristics
*   **Strengths:** Bridges the conceptual gap between spatial and TF domains; allows simultaneous interaction; facilitates natural exploration and specification [S1, book].
*   **Limitations:** Requires user familiarity with both domains; manual specification can be tedious for complex TFs [S1, book].
*   **Performance:** Not given in the sources.

## Background
This method was developed to address the difficulty of determining TF regions corresponding to features of interest, particularly small ones [S1, book]. It is part of a broader effort in multidimensional transfer functions for volume rendering [S1, book].

## Sources
[S1] PDF 9 Multi-Dimensional Transfer Functions for Volume Rendering, https://people.cs.uchicago.edu/~glk/pubs/pdf/Kniss-MultidimensionalTransferFunctions-VH-2005.pdf, fetched 2026-10-27 (book)
