# Composite Template Construction and Synchronization

Also called: Composite Template Construction & Synchronization, Composite Template Method, Composite Template Construction

## Overview
Composite Template Construction and Synchronization is a two-step preprocessing method for compositing and synchronizing Adaptive Mesh Refinement (AMR) data from a series of timesteps to enable query-driven visualization on a common grid hierarchy [S1]. It solves the problem of temporal disparities in spatial refinement, where the disparity of cell refinement between grid hierarchies at a fixed spatial location across timesteps prevents the simultaneous evaluation of data necessary for many visualization algorithms [S1]. Unlike temporally sequential methods that render individual timesteps as an animation, this method presents important features from multiple timesteps in a single, meaningful image (multitemporal visualization) [S1]. It differs from independent grid treatment or physical combination by preserving the greatest refinement from each timestep and providing a consistent basis for mapping across timesteps [S1].

## How it works
The method consists of two distinct phases: Composite Template Construction and Grid Synchronization.

### Inputs and Outputs
**Inputs:**
*   A subset of selected timesteps from a time-varying AMR dataset.
*   Each timestep consists of a unique grid hierarchy, consisting of multiple levels of cell refinement [S1].
*   Grid cells are properly nested, meaning cells at level $l$ do not abut cells at level $l-2$ or lower without intermediate levels [S1].
*   Refinement ratio $r$ (typically $r=2$) [S1].

**Outputs:**
*   A **Composite Template**: A unified reference grid structure that preserves the highest level of refinement from each timestep utilized in its construction [S1].
*   **Synchronized Timesteps**: Each original timestep’s grid hierarchy mapped to the composite template, enabling multitemporal query-driven visualization [S1].

### Data Structures
The composite template is an AMR grid hierarchy. It maintains the same structural properties as the input hierarchies:
*   **Levels:** Integer levels $l$ from $0$ to $l_{max}$.
*   **Cells:** Grid cells at each level, properly nested.
*   **Values:** Cell-centered scalar values associated with each cell [S1].

### Step 1: Composite Template Construction
This step generates a unified grid template from the AMR grid hierarchies of the selected timesteps. The process is refinement-level ordered [S1].

1.  Identify the maximum refinement level $l_{max}$ present across all selected timesteps.
2.  **Initialize** the composite template as empty.
3.  **Iterate** downwards from level $l = l_{max}$ to $l = 0$:
    a. For each timestep in the selected subset:
        i. Identify all grid cells at level $l$.
        ii. For each cell, check if a higher-resolution grid cell (at level $> l$) has already been placed in the same spatial position in the composite template.
        iii. **Condition:** Add the cell to the composite template **if and only if** no higher-resolution grid cell has already been placed at that spatial position [S1].
4.  The resulting template preserves the greatest refinement from each timestep [S1].

### Step 2: Grid Synchronization
This step aligns variable information from each timestep’s grid hierarchy to the composite template.

1.  For each timestep in the selected subset:
    a. **Identify** grid cells from the original timestep hierarchy that map to regions of greater refinement in the composite template. Specifically, identify cells that are coarser than the corresponding cells in the composite template [S1].
    b. **Regrid:** Iteratively divide the original grid cell into a nested group of increasingly refined cells.
        i. The iterative refinement proceeds until the created grid cells are identical in refinement and hierarchical ordering to the group of nested grid cells in the composite template [S1].
    c. **Propagate Values:** Propagate the cell-centered value of the original grid cell to the centers of the newly created grid cells [S1].
2.  After synchronization, every grid cell from any AMR grid hierarchy used to construct the composite template can be mapped to a grid cell of equivalent refinement in the template [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_composite_template(timesteps):
    """
    timesteps: List of objects, each containing a list of levels.
               Each level is a list of Cell objects.
               Cell has attributes: level, spatial_position, value.
    Returns: CompositeTemplate object containing levels and cells.
    """
    composite = CompositeTemplate()
    
    # Find global max level
    l_max = max(cell.level for ts in timesteps for level in ts.levels for cell in level)
    
    # Iterate from highest refinement to lowest
    for l in range(l_max, -1, -1):
        for timestep in timesteps:
            # Get cells at level l from this timestep
            cells_at_l = [cell for cell in timestep.get_cells_at_level(l)]
            
            for cell in cells_at_l:
                # Check if a higher resolution cell already occupies this space
                # This requires checking if any cell in the composite template
                # at level > l overlaps with cell.spatial_position
                if not composite.has_higher_resolution_cell_at(cell.spatial_position):
                    composite.add_cell(cell)
                    
    return composite

def synchronize_timestep(timestep, composite_template):
    """
    Synchronizes a single timestep's grid hierarchy to the composite template.
    Returns a synchronized version of the timestep.
    """
    synchronized_cells = []
    
    # Iterate through all cells in the original timestep
    for cell in timestep.all_cells:
        # Determine the target refinement level in the composite template
        # at the spatial position of this cell
        target_cells = composite_template.get_cells_at_position(cell.spatial_position)
        
        if not target_cells:
            # Cell is outside the domain of the composite template
            continue
            
        # Find the finest cell in the composite template at this position
        finest_target = max(target_cells, key=lambda c: c.level)
        
        if cell.level == finest_target.level:
            # Already matches, just copy value
            synchronized_cells.append(Cell(level=cell.level, 
                                           position=cell.spatial_position, 
                                           value=cell.value))
        else:
            # Cell is coarser than the composite template requires
            # Perform regridding
            new_cells = regrid_cell(cell, finest_target.level)
            synchronized_cells.extend(new_cells)
            
    return SynchronizedTimestep(cells=synchronized_cells)

def regrid_cell(cell, target_level):
    """
    Iteratively divides a cell into a nested group of increasingly refined cells
    until they match the target level.
    """
    # Assuming a standard refinement ratio r=2 for hexahedral/tetrahedral cells
    # The number of subdivisions needed
    levels_to_add = target_level - cell.level
    
    # Generate nested cells
    # Note: In a real implementation, this involves splitting the cell geometry
    # and creating child cells. For pseudocode, we assume a helper exists.
    child_cells = []
    current_cell = cell
    
    for _ in range(levels_to_add):
        # Split current_cell into r^d children (where d is dimension)
        # For r=2, d=3, this creates 8 children.
        # We select the specific child that corresponds to the spatial position
        # of the target composite cell.
        # Since the composite template is properly nested, we can traverse
        # the hierarchy to find the specific leaf cell.
        
        # Simplified: Assume we can directly generate the leaf cells
        # that correspond to the target refinement.
        pass 
        
    # Propagate value to all new cells
    # The value is constant for the parent cell in this synchronization method
    leaf_cells = generate_leaf_cells(cell, target_level)
    for leaf in leaf_cells:
        leaf.value = cell.value
        
    return leaf_cells

def generate_leaf_cells(parent_cell, target_level):
    """
    Generates the leaf cells resulting from refining parent_cell to target_level.
    """
    # Implementation depends on cell type (hex, tet, etc.)
    # For a hexahedron with r=2:
    # Level 0: 1 cell
    # Level 1: 8 cells
    # Level 2: 64 cells
    # ...
    # Level k: 8^k cells
    
    # This function would recursively subdivide the parent cell
    # and return the list of leaf cells at the target level.
    # [G] Placeholder for geometry-specific subdivision logic.
    return []
```

### Parameters
*   **Refinement Ratio ($r$):** Typically $r=2$ [S1].
*   **Selected Timesteps:** The subset of timesteps used to build the template. The choice of timesteps affects storage costs [S1].

### Edge Cases and Pitfalls
*   **Storage Costs:** A large number of compositing steps may cause the template to represent the entire domain at the finest refinement level. While the template itself is not the expensive factor with respect to storage, the synchronization of AMR timesteps with this fully refined template increases storage costs significantly [S1].
*   **Disparity of Refinement:** The method resolves disparities in cell refinement between grid hierarchies at a fixed spatial location across timesteps. If disparities are not resolved, simultaneous evaluation of data for many visualization algorithms is prevented [S1].
*   **Nested Property:** The method assumes properly nested grids, where cells at level $l$ do not abut cells at level $l-2$ or lower [S1].

## Operating characteristics
*   **Time Complexity:** The construction phase requires iterating through all cells of all selected timesteps and checking for spatial overlap with higher-resolution cells. The synchronization phase requires regridding coarser cells to match the composite template's refinement. The complexity is proportional to the total number of cells in the input timesteps and the depth of refinement [S1].
*   **Memory Complexity:** The memory usage is dominated by the composite template and the synchronized timesteps. In the worst-case scenario, where the composite template represents the entire domain at the finest refinement level, storage costs for synchronization are high [S1].
*   **Performance:** By synchronizing all timesteps to a common AMR template, query processing work can be moved to the GPU, greatly accelerating the performance of query-driven visualization (QDV) analysis [S1].
*   **Strengths:** Preserves the greatest refinement from each timestep, ensuring high-fidelity data is used. Provides a consistent basis for mapping across timesteps. Enables multitemporal visualization in a single image [S1].
*   **Limitations:** Potential for high storage costs if many timesteps with disparate refinement patterns are synchronized. The method is a preprocessing step, adding overhead before visualization [S1].

## Background
The method was developed by Luke J. Gosink, John C. Anderson, and Kenneth I. Joy at the Institute for Data Analysis and Visualization (IDAV) at the University of California, Davis [S1]. It was presented in a paper titled "Query-Driven Visualization of Time-Varying Adaptive Mesh Refinement Data," published in IEEE Transactions on Visualization and Computer Graphics (TVCG) in August 2008, posted online on 19 October 2008 [S1]. The work addresses challenges in visualizing time-varying AMR data using query-driven visualization (QDV) approaches, which allow users to process ad-hoc queries over large-scale datasets [S1]. The method integrates with a GPU-based query engine called the Bin-Hash index [S1].

## Sources
[S1] Query-Driven Visualization of Time-Varying Adaptive Mesh Refinement Data, https://digital.library.unt.edu/ark:/67531/metadc897115/?q=amr volume rendering, fetched 2026-10-28
