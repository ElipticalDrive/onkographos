# Time-varying AMR Interpolation Framework

Also called: Kaehler et al. (2005) method, Kaehler et al. framework, Time-Varying AMR Interpolation

## Overview
The Time-varying AMR Interpolation Framework is a method for visualizing adaptive mesh refinement (AMR) data that changes over time [S1, S2, unofficial]. It solves the problem of inconsistent refinement configurations between consecutive time steps, where finer patches are often updated more frequently than coarse ones [S1, S2, unofficial]. Unlike static AMR interpolation, this framework ensures temporal consistency by synchronizing the refinement topology before computing intermediate values, allowing for smooth visualization of features that split, merge, or appear/disappear [S1, unofficial]. It builds on [Interpolation and reconstruction methods](17874-interpolation-and-reconstruction-methods.md) by adding a temporal synchronization layer.

## How it works
**Inputs:** Two consecutive time steps of AMR data, $T_i$ and $T_{i+1}$, each consisting of a hierarchy of patches with varying refinement levels.
**Outputs:** A temporally consistent dataset where both time steps share the same refinement configuration, with interpolated values for cells present in one step but not the other.

**Data Structures:**
The framework operates on the existing AMR data structure (e.g., octree or hierarchical grid). No new complex data structures are defined in the sources; instead, it modifies the interpretation of existing cells.
- **Refinement Configuration:** A boolean mask or set of active cells for each time step.
- **Interpolated Values:** Scalar values computed for cells that exist in the target configuration but lack data in the source time step.

**Steps:**
1. **Synchronize Refinement Configuration:**
   For two subsequent time steps $T_i$ and $T_{i+1}$, determine the union of their refinement configurations. Ensure that every cell refined in $T_i$ is also considered refined in $T_{i+1}$, and vice versa [S1, S2, unofficial]. This creates a common topological basis for interpolation.

2. **Interpolate Missing Values:**
   Identify cells that are refined in one time step but not the other. For these cells, compute values using an interpolation scheme [S1, S2, unofficial].
   - If a cell is refined in $T_{i+1}$ but not $T_i$, interpolate its value from the coarser parent or neighboring cells in $T_i$.
   - If a cell is refined in $T_i$ but not $T_{i+1}$, interpolate its value from the coarser parent or neighboring cells in $T_{i+1}$.
   The specific interpolation kernel is not detailed in the sources, but it must handle regions covered by coarser levels which are updated less frequently [S1, S2, unofficial].

3. **Compute Intermediate Values:**
   Once the refinement configurations are aligned, define an interpolation scheme to compute intermediate values for the time interval between $T_i$ and $T_{i+1}$ [S1, S2, unofficial]. This allows for smooth temporal transitions in visualization.

4. **Remote Data Access:**
   The framework automatically handles remote data access. Interpolated values are computed on the machine running the simulation, ensuring that the visualization client receives consistent data [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def synchronize_and_interpolate(time_step_i, time_step_i_plus_1):
    """
    Synchronizes refinement configurations between two time steps and interpolates missing values.
    
    Args:
        time_step_i: AMR data structure for time t
        time_step_i_plus_1: AMR data structure for time t+1
        
    Returns:
        synchronized_data_i: Data for time t with interpolated values for missing cells
        synchronized_data_i_plus_1: Data for time t+1 with interpolated values for missing cells
    """
    # Step 1: Determine the union of refinement configurations
    # A cell is "active" if it is refined in either time step
    active_cells = union(time_step_i.refined_cells, time_step_i_plus_1.refined_cells)
    
    # Step 2: Interpolate values for cells missing in time_step_i
    synchronized_data_i = copy(time_step_i)
    for cell in active_cells:
        if cell not in time_step_i.refined_cells:
            # Interpolate value from coarser parent or neighbors in time_step_i
            synchronized_data_i.values[cell] = interpolate_from_coarser(time_step_i, cell)
            
    # Step 3: Interpolate values for cells missing in time_step_i_plus_1
    synchronized_data_i_plus_1 = copy(time_step_i_plus_1)
    for cell in active_cells:
        if cell not in time_step_i_plus_1.refined_cells:
            # Interpolate value from coarser parent or neighbors in time_step_i_plus_1
            synchronized_data_i_plus_1.values[cell] = interpolate_from_coarser(time_step_i_plus_1, cell)
            
    # Step 4: Compute intermediate values for visualization
    # This step allows for smooth temporal transitions
    intermediate_values = compute_temporal_interpolation(synchronized_data_i, synchronized_data_i_plus_1)
    
    return synchronized_data_i, synchronized_data_i_plus_1, intermediate_values

def interpolate_from_coarser(amr_data, cell):
    """
    Interpolates value for a cell not refined in the given AMR data.
    Uses coarser parent or neighboring cells.
    Note: Specific interpolation kernel is not defined in sources [S1, S2, unofficial].
    """
    # [G] Implementation detail: Typically involves trilinear interpolation from parent cells
    # or weighted average of neighbors based on distance.
    parent = get_parent_cell(cell)
    if parent is not None and parent in amr_data.refined_cells:
        return amr_data.values[parent]
    else:
        # Fallback to neighbor interpolation
        neighbors = get_neighbors(cell)
        return weighted_average(amr_data.values[n] for n in neighbors if n in amr_data.refined_cells)
```

**Parameters:**
- **Interpolation Kernel:** Not specified in the sources. Must handle regions covered by coarser levels [S1, S2, unofficial].
- **Remote Access:** Enabled by default; computations occur on the simulation machine [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Interpolation may introduce artifacts if the coarser level does not capture fine-scale features [S1, S2, unofficial].
- **Boundaries:** Cells at the domain boundary may lack neighbors for interpolation.
- **Degenerate Input:** If a cell is refined in one step but its parent is not available in the other, interpolation may fail. The framework assumes a valid hierarchy [S1, S2, unofficial].

## Subtopics

### AMR Basis Reconstruction Filter
A sibling method that focuses on basis function reconstruction for static AMR data. It does not address temporal synchronization. Choose this for static data visualization. [Link](5581-amr-basis-reconstruction-filter.md)

## Operating characteristics
- **Time Complexity:** Not given in the sources. Depends on the number of cells and the interpolation kernel.
- **Memory Complexity:** Not given in the sources. Requires storing interpolated values for synchronized cells.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Ensures temporal consistency in AMR visualization; handles remote data access automatically.
- **Limitations:** Requires synchronization of refinement configurations, which may be computationally expensive for large datasets.
- **When to Use:** When visualizing time-varying AMR data with inconsistent refinement frequencies.
- **When Not to Use:** For static AMR data or when temporal consistency is not required.

## Background
The Time-varying AMR Interpolation Framework was described by Kaehler et al. in 2005 [S1, S2, unofficial]. It addresses the challenge of visualizing AMR data where finer patches are updated more frequently than coarse ones. The framework ensures that both time steps have the same refinement configuration, allowing for consistent interpolation and visualization. It is used in conjunction with AMR visualization tools to handle remote data access and compute interpolated values on the simulation machine [S1, S2, unofficial].

## Sources
[S1] Visualization Tools for Adaptive Mesh Refinement Data, https://digital.library.unt.edu/ark:/67531/metadc894061/, fetched 2026-10-26 (unofficial)
[S2] Visualization of Scalar Adaptive Mesh Refinement Data, https://digital.library.unt.edu/ark:/67531/metadc901373/?q=adaptive mesh refinement visualization ray casting, fetched 2026-10-28 (unofficial)
