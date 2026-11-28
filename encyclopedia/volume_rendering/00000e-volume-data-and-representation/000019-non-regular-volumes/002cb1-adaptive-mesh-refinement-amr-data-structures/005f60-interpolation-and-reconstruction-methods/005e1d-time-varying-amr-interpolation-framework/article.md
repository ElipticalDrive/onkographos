# Time-varying AMR Interpolation Framework

Also called: Kaehler et al. (2005) method, Kaehler et al. framework, Time-Varying AMR Interpolation

## Overview
The Time-varying AMR Interpolation Framework is a method for visualizing adaptive mesh refinement (AMR) data that changes over time [S1, unofficial][S2, unofficial]. It solves the problem of inconsistent refinement configurations between consecutive time steps, where finer patches are often updated more frequently than coarse ones [S1, unofficial][S2, unofficial]. Unlike static AMR interpolation, this framework ensures temporal consistency by synchronizing the refinement topology before computing intermediate values, allowing for smooth visualization of features that split, merge, or appear and disappear [S1, unofficial].

## How it works
**Inputs:** Two consecutive time steps of AMR data, $T_i$ and $T_{i+1}$ [S1, unofficial].
**Outputs:** A temporally consistent dataset where both time steps share the same refinement configuration, with interpolated values for cells present in one step but not the other [S1, unofficial].

**Data Structures:** The framework operates on the existing AMR data structure (e.g., octree or hierarchical grid) without defining new complex structures [S1, unofficial].

**Steps:**
1. **Synchronize Refinement Configuration:** For two subsequent time steps, ensure both have the same refinement configuration, meaning each cell that is refined in one time step is also refined in the other time step [S1, unofficial][S2, unofficial].
2. **Interpolate Missing Values:** Obtain values for cells that are not refined in the current time step but are refined in the other via interpolation [S1, unofficial].
3. **Compute Intermediate Values:** Define an interpolation scheme to compute intermediate values in regions that are covered by coarser levels and thus updated less frequently [S1, unofficial][S2, unofficial].
4. **Remote Data Access:** Automatically handle remote data access and compute interpolated values on the machine that also runs the simulation [S1, unofficial][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def synchronize_and_interpolate(time_step_i, time_step_i_plus_1):
    # Step 1: Synchronize refinement configuration
    active_cells = set(time_step_i.refined_cells) | set(time_step_i_plus_1.refined_cells)
    
    synchronized_i = {}
    synchronized_i_plus_1 = {}
    
    # Step 2: Interpolate values for cells not refined in current step but refined in other
    for cell in active_cells:
        if cell in time_step_i.refined_cells:
            synchronized_i[cell] = time_step_i.values[cell]
        else:
            # [G] Standard spatial interpolation from coarser parent or neighboring cells
            synchronized_i[cell] = interpolate_spatial(time_step_i, cell)
            
        if cell in time_step_i_plus_1.refined_cells:
            synchronized_i_plus_1[cell] = time_step_i_plus_1.values[cell]
        else:
            synchronized_i_plus_1[cell] = interpolate_spatial(time_step_i_plus_1, cell)
            
    # Step 3: Compute intermediate values for regions covered by coarser levels
    intermediate_values = compute_temporal_interpolation(synchronized_i, synchronized_i_plus_1)
    
    # Step 4: Remote data access is handled automatically by the framework
    return synchronized_i, synchronized_i_plus_1, intermediate_values

def interpolate_spatial(amr_data, cell):
    # [G] Trilinear or weighted average from available parent or neighbor cells
    parent = get_parent_cell(cell)
    if parent and parent in amr_data.refined_cells:
        return amr_data.values[parent]
    neighbors = get_neighbor_cells(cell)
    valid_vals = [amr_data.values[n] for n in neighbors if n in amr_data.refined_cells]
    return sum(valid_vals) / len(valid_vals) if valid_vals else 0.0

def compute_temporal_interpolation(data_i, data_i_plus_1):
    # [G] Linear interpolation across the time interval
    return {k: 0.5 * data_i[k] + 0.5 * data_i_plus_1[k] for k in data_i}

def get_parent_cell(cell):
    # [G] Returns the coarser grid cell containing the given cell
    return cell.parent

def get_neighbor_cells(cell):
    # [G] Returns the 6 adjacent cells in the grid
    return [cell.offset(dx, dy, dz) for dx, dy, dz in [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]]
```

**Parameters:**
- **Interpolation Kernel:** Not given in the sources. Must handle regions covered by coarser levels [S1, unofficial][S2, unofficial].
- **Remote Access:** Enabled by default; computations occur on the simulation machine [S1, unofficial][S2, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Handling:** Cells at the domain boundary may lack neighbors for interpolation [G].
- **Degenerate Input:** If a cell is refined in one step but its parent is not available in the other, interpolation may require fallback strategies [G].

## Subtopics

### AMR Basis Reconstruction Filter
A sibling method that focuses on basis function reconstruction for static AMR data [G]. It does not address temporal synchronization [G]. Choose this for static data visualization. [Link](5581-amr-basis-reconstruction-filter.md)

## Operating characteristics
- **Time Complexity:** Not given in the sources. Depends on the number of cells and the interpolation kernel [S1, unofficial][S2, unofficial].
- **Memory Complexity:** Not given in the sources. Requires storing interpolated values for synchronized cells [S1, unofficial][S2, unofficial].
- **Measured Performance:** Not given in the sources [S1, unofficial][S2, unofficial].
- **Strengths:** Ensures temporal consistency in AMR visualization and handles remote data access automatically [S1, unofficial][S2, unofficial].
- **Limitations:** Requires synchronization of refinement configurations, which may be computationally expensive for large datasets [S1, unofficial][S2, unofficial].
- **When to Use:** When visualizing time-varying AMR data with inconsistent refinement frequencies [S1, unofficial][S2, unofficial].
- **When Not to Use:** For static AMR data or when temporal consistency is not required [G].

## Background
The Time-varying AMR Interpolation Framework was described by Kaehler et al. in 2005 [S1, unofficial][S2, unofficial]. It addresses the challenge of visualizing AMR data where finer patches are updated more frequently than coarse ones [S1, unofficial][S2, unofficial]. The framework ensures that both time steps have the same refinement configuration, allowing for consistent interpolation and visualization [S1, unofficial][S2, unofficial]. It is used in conjunction with AMR visualization tools to handle remote data access and compute interpolated values on the simulation machine [S1, unofficial][S2, unofficial].

## Sources
[S1] Visualization Tools for Adaptive Mesh Refinement Data, https://digital.library.unt.edu/ark:/67531/metadc894061/, fetched 2026-10-26 (unofficial)
[S2] Visualization of Scalar Adaptive Mesh Refinement Data, https://digital.library.unt.edu/ark:/67531/metadc901373/?q=adaptive mesh refinement visualization ray casting, fetched 2026-10-28 (unofficial)
