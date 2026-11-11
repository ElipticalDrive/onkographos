# Simulation Grid Cell Skipping

Also called: Surface Extraction with Simulation Grid

## Overview
Simulation Grid Cell Skipping is an optimization technique for surface extraction that accelerates iteration over voxel columns by skipping over empty and fully occupied simulation grid cells, processing only partially filled cells [S1]. It sets itself apart from standard voxel traversal by leveraging the coarse-grained status of simulation grid cells (empty, full, or partial) to bypass large volumes of uniform data, building on the THREAD System's ray-marching framework [3743-thread-system.md].

## How it works
**Inputs and Outputs:**
- Input: A simulation grid where each cell is classified as empty, full, or partially filled; a view ray for each voxel column; the current extraction state.
- Output: A list of relevant voxels within partially filled cells for surface extraction; updated ray-marching state.

**Data Structures:**
- Simulation Grid: A regular 3D grid. Each cell stores a status flag: `EMPTY`, `FULL`, or `PARTIAL`.
- Voxel Column: A sequence of voxels along a view ray.
- Packed Machine Words: Bits representing voxel occupancy or cell status, processed in chunks for efficiency.

**Numbered Steps:**
1. For each voxel column, initialize a ray marcher along the view ray corresponding to that column.
2. Iterate over packed machine words rather than single bits, using bit counting intrinsics to quickly determine the next relevant bits to be considered [S1].
3. March along the view ray through the simulation grid.
4. Keep marching through full or empty simulation grid cells until encountering a succession of partially filled cells [S1].
5. Interrupt ray marching to iterate over the corresponding voxels when partially filled cells are encountered [S1].
6. When the iteration reaches the end of the current set of connected partially filled cells, resume ray marching in the simulation grid [S1].

**Formulas:**
- Cell Status Determination: During simulation, determine if a cell is empty, full, or partial based on particle presence [S1].
- Ray Marching Step: Advance ray by cell extent ratio, keeping $\alpha$ constant throughout the perspective grid (logarithmic depth partitioning) [S1].

**Parameters:**
- Cell Status Thresholds: Not given in the sources.
- Bit Counting Intrinsics: Hardware-dependent (e.g., `__builtin_popcount` on GCC/Clang) [S1].

**Edge Cases and Pitfalls:**
- Do not skip cells that have partially filled neighbors to avoid cutting off detailed boundary surfaces potentially formed by particles overlapping cell boundaries; only inner full and empty cells may be skipped [S1].
- Precision issues in ray marching may cause missed intersections if cell boundaries are not handled correctly [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simulate_grid_cell_skipping(simulation_grid, voxel_columns):
    """
    Accelerates surface extraction by skipping empty/full cells.
    
    Args:
        simulation_grid: 3D array of cell statuses (EMPTY, FULL, PARTIAL)
        voxel_columns: List of view rays, one per voxel column
        
    Returns:
        List of voxels to process for surface extraction
    """
    voxels_to_process = []
    
    for ray in voxel_columns:
        # Initialize ray marcher
        current_pos = ray.origin
        direction = ray.direction
        
        while current_pos is within simulation_grid_bounds:
            # Get current simulation grid cell
            cell = get_cell_at_position(simulation_grid, current_pos)
            
            # Check if cell has partially filled neighbors
            has_partial_neighbor = check_neighbors_for_partial(cell, simulation_grid)
            
            if cell.status == EMPTY or cell.status == FULL:
                # Skip if no partial neighbors (inner cell)
                if not has_partial_neighbor:
                    # Advance ray by cell extent
                    current_pos = advance_ray_by_cell(current_pos, direction, cell.extent)
                    continue
                else:
                    # Must process this cell as it borders partial cells
                    pass
            
            # If PARTIAL or (EMPTY/FULL with partial neighbors), process voxels
            voxels = iterate_over_voxels_in_cell(cell, ray)
            voxels_to_process.extend(voxels)
            
            # Advance ray to next cell
            current_pos = advance_ray_by_cell(current_pos, direction, cell.extent)
            
    return voxels_to_process

def check_neighbors_for_partial(cell, simulation_grid):
    """Check if any neighbor of the cell is PARTIAL."""
    neighbors = get_26_neighbors(cell.position)
    for neighbor_pos in neighbors:
        if simulation_grid[neighbor_pos].status == PARTIAL:
            return True
    return False

def iterate_over_voxels_in_cell(cell, ray):
    """Iterate over voxels in a cell along the ray."""
    voxels = []
    # Use bit counting intrinsics for packed words
    # This is a placeholder for the actual bit-packing logic
    # which depends on the specific hardware and data layout
    # Not given in the sources.
    return voxels
```

## Subtopics

### [Masking EB-covered cells in isosurface extraction](12775-masking-eb-covered-cells-in-isosurface-extraction.md)
A technique to exclude fully covered cells from isosurface extraction in simulations using Embedded Boundary (EB) methods, ensuring the surface geometry respects the boundary geometry defined by volume fractions.

## Operating characteristics
- Time Complexity: $O(N)$ where $N$ is the number of voxels, but with a reduced constant factor due to skipping empty/full cells.
- Memory Complexity: $O(1)$ additional memory per voxel column for ray marching state.
- Measured Performance: Not given in the sources.
- Scaling: Scales well with simulation grid size, as larger uniform regions are skipped more effectively.
- Strengths: Significant speedup for simulations with large empty or full regions.
- Limitations: Overhead of checking neighbor status; less effective for highly fragmented partial regions.
- When to Use: SPH simulations with regular grids and distinct empty/full/partial regions [S1].
- When Not to Use: Simulations with highly uniform partial filling or where neighbor checks are too costly.

## Background
Originated in the context of Memory-Efficient On-The-Fly Voxelization of Particle Data [S1]. It leverages the regular grid structure used in SPH simulations for neighbor search [S1].

## Sources
[S1] Memory-Efficient On-The-Fly Voxelization of Particle Data, https://doi.org/10.2312/pgv.20151150, fetched 2026-10-26
