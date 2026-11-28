# AxisReflection

Also called: Axis Reflection

## Overview
AxisReflection is a geometry transformation filter that applies axis-aligned reflections to a hypertree grid to produce multiple images whose union forms a complete or symmetric shape without duplicating underlying data [S1, unofficial]. It is distinct from real data replication because it leverages the hypertree grid's structure-preserving paradigm to maintain negligible memory increase and execution time, making it suitable for large-scale AMR grids where memory constraints are critical [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ternary tree-based Adaptive Mesh Refinement (AMR) hypertree grid.
*   **Output:** A set of reflected hypertree grid instances. The union of these instances represents the target symmetric shape.

**Data Structures:**
The method operates on the internal structure of the hypertree grid. The grid consists of trees and leaves. The memory footprint is measured in terms of the number of leaves, number of trees, and Maximum Resident Set Size (RSS) in kiB [S1, unofficial].

**Parameters:**
*   **Number of reflections:** Typically 7 for completing a unit ball from a single octant [S1, unofficial].
*   **Reflection planes:** Axis-aligned planes passing through the center of the hypertree grid [S1, unofficial].

**Algorithm Steps:**
1.  **Define Bitmask:** Create a bitmask using a quadratic function to retain only those cells that are within or intersect a target shape (e.g., a truncated octant) [S1, unofficial].
2.  **Initial Octant:** Render or process the initial octant defined by the bitmask.
3.  **Successive Reflections:** Apply successive reflections across the defined axis-aligned planes passing through the grid center. For a 3D cube/octant symmetry, 7 reflections are required to generate the remaining 7 octants from the initial one [S1, unofficial].
4.  **Union:** Union the reflected outputs with the initial octant to form the complete shape [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def axis_reflection(hypertree_grid, num_reflections=7):
    """
    Applies axis-aligned reflections to a hypertree grid to form a symmetric shape.
    
    Args:
        hypertree_grid: The input ternary tree-based AMR hypertree grid.
        num_reflections: The number of reflections to apply (default 7 for 3D octant completion).
        
    Returns:
        A list of hypertree grid instances representing the reflected shapes.
    """
    # Step 1: Define the initial octant using a bitmask
    # The bitmask is defined using a quadratic function to retain cells 
    # within or intersecting a truncated octant.
    initial_octant = apply_bitmask(hypertree_grid, quadratic_function="within_truncated_octant")
    
    # Step 2: Initialize list of reflected grids
    reflected_grids = [initial_octant]
    
    # Step 3: Apply successive reflections
    # Reflection planes are axis-aligned and pass through the center of the grid.
    # For 3D, we reflect across X, Y, and Z axes to generate all 8 octants.
    # 7 reflections are needed to complete the shape from the initial octant.
    for i in range(num_reflections):
        # Determine the reflection plane for this iteration.
        # The specific axis and direction depend on the symmetry group.
        # For a cube, reflections are across planes x=0, y=0, z=0 and their combinations.
        reflection_plane = get_reflection_plane(i, grid_center=hypertree_grid.center)
        
        # Reflect the current grid instance across the plane.
        # This operation is structure-preserving and does not duplicate data.
        reflected_grid = reflect_across_plane(reflected_grids[-1], reflection_plane)
        reflected_grids.append(reflected_grid)
        
    return reflected_grids

def apply_bitmask(grid, quadratic_function):
    """
    Applies a bitmask to the grid based on a quadratic function.
    [G] Implementation detail: This involves evaluating the quadratic function 
    at cell centers or boundaries to determine inclusion.
    """
    # Evaluate quadratic function for each cell
    # Retain cells where function value indicates inclusion in target shape
    return grid.filter_cells(quadratic_function)

def reflect_across_plane(grid, plane):
    """
    Reflects the grid geometry across an axis-aligned plane.
    [G] Implementation detail: This modifies the geometric coordinates of the 
    grid cells without duplicating the topological structure or data values.
    """
    # Modify coordinates of grid cells based on plane equation
    # For axis-aligned plane, this is a simple sign flip or offset adjustment
    return grid.transform_geometry(plane)

def get_reflection_plane(index, grid_center):
    """
    Determines the axis-aligned reflection plane based on the iteration index.
    [G] Specific logic for generating 7 unique reflection planes for 3D octant symmetry.
    """
    # Logic to generate planes x=0, y=0, z=0 and combinations
    # This is a placeholder for the specific plane generation logic
    pass
```

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on the hypertree grid's structure. Precision issues may arise if the reflection planes do not align perfectly with the grid's axis-aligned nature [S1, unofficial].
*   **Boundary Conditions:** The bitmask definition must correctly handle cells intersecting the boundary of the target shape to ensure a complete union [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Execution Time:** Executing 7 reflections takes negligible time compared to octant creation or rendering [S1, unofficial]. This confirms the theoretical prediction that a correctly implemented reflection filter should have negligible execution time [S1, unofficial].
*   **Memory Footprint:** The memory footprint (RSS) for reflections is 45,172 kiB, which is identical to the leaf/tree counts of a single octant but vastly lower than real replication (359,392 kiB for octant*8) [S1, unofficial]. The memory increase is negligible compared to real replication, which scales commensurately with data size [S1, unofficial].

**Performance:**
*   In a test on a single core with a ternary tree-based AMR grid (5x5x6 root cells), the reflections resulted in an AMR mesh 8 times larger (over 1 million cells, 96.3% leaf cells) with negligible time and memory increase [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Structure-preserving, negligible memory increase, negligible execution time, suitable for large-scale AMR grids.
*   **Limitations:** Limited to axis-aligned reflections and symmetric shapes. Requires a hypertree grid structure.

**When to Use:**
*   Use AxisReflection when you need to visualize or analyze symmetric shapes from a partial representation (e.g., an octant) on large-scale AMR grids without incurring the memory cost of full data replication [S1, unofficial].

## Background
**Origin:**
AxisReflection is a native filter in the VTK (Visualization Toolkit) library, specifically designed for hypertree grids [S1, unofficial]. It was introduced as part of the Efficient visualization and analysis of large-scale, tree-based, adaptive mesh refinement simulations with rectilinear geometry research [S1, unofficial].

**History and Development:**
The filter was developed to address the memory and performance limitations of real data replication for symmetric AMR grids. It leverages the structure-preserving paradigm of hypertree grids to achieve efficient geometry transformations [S1, unofficial].

**Usage:**
It is used for structure-preserving geometry transformations on ternary tree-based AMR grids with bitmasks [S1, unofficial]. It is often used in conjunction with other filters like AxisCut and PlaneCutter for comprehensive AMR data analysis [S1, unofficial].

## Sources
[S1] Efficient visualization and analysis of large-scale, tree-based, adaptive mesh refinement simulations with rectilinear geometry, https://cea.hal.science/cea-03501320v1, fetched 2026-10-26 (unofficial)
