# Composite Reference Volume Generation

Also called: ALL volume, ALL/ANY Volumes, ANY volume, Intersection/Union Composite Volumes

## Overview
Composite Reference Volume Generation creates two binary composite volumes (ALL and ANY) from a set of reference expert volumes to define the maximal and minimal boundaries of expert performance [S1]. These composites serve as the ground truth against which resident test volumes are compared, accounting for variation in expert performance [S1]. Unlike [Occupancy Ground Truth Computation](18297-occupancy-ground-truth-computation.md), which computes occupancy for arbitrary coordinates via ray casting, this method aggregates a fixed set of discrete reference volumes using set-theoretic operations to establish mandatory and allowed removal zones [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of reference volumes $R = \{V_1, V_2, ..., V_N\}$, where each $V_i$ is a binary volume of identical dimensions [S1].
*   **Output:** Two composite binary volumes: $ALL$ and $ANY$, with the same dimensions as the input volumes [S1].

**Data Structures**
*   **Reference Volume ($V$):** A 3D grid of voxels [G]. Each voxel is binary: occupied (1) or empty (0) [S1].
*   **Composite Volumes ($ALL, ANY$):** 3D grids of the same dimensions as $V$, initialized to empty (0) [G].

**Numbered Steps**
1.  Define the set of reference volumes $R$ [S1].
2.  Initialize the $ALL$ volume grid to all zeros (empty) [G].
3.  Initialize the $ANY$ volume grid to all zeros (empty) [G].
4.  Iterate through every voxel position $(x, y, z)$ in the grid dimensions [G].
5.  Check the occupancy of the voxel at $(x, y, z)$ in all reference volumes $V \in R$ [S1].
6.  Compute the state for the $ALL$ volume: the voxel is occupied if and only if it is occupied in **all** reference volumes in $R$ [S1].
7.  Compute the state for the $ANY$ volume: the voxel is occupied if and only if it is occupied in **any** (at least one) reference volume in $R$ [S1].

**Formulas**
$$ ALL = \bigcap_{V \in R} V, \quad ANY = \bigcup_{V \in R} V $$
where $R$ is the set of reference volumes and $V$ represents a single reference volume [S1]. For binary grids, the intersection $\bigcap$ corresponds to logical AND (all voxels must be 1) and the union $\bigcup$ corresponds to logical OR (at least one voxel must be 1) [S1].

**Parameters**
*   **Reference Set Size ($N$):** The number of reference volumes in $R$. Typical value: 4 [S1].
*   **Volume Type:** Binary [S1].

**Edge Cases and Pitfalls**
*   **Empty Reference Set:** The method requires at least one reference volume to compute intersections and unions [S1].
*   **Dimension Mismatch:** All reference volumes must have identical dimensions to allow voxel-wise comparison [S1].
*   **Binary Assumption:** The method assumes binary inputs; non-binary values will break the logical aggregation [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_composite_volumes(reference_volumes):
    """
    Generates ALL and ANY composite volumes from a set of reference binary volumes.
    
    Args:
        reference_volumes: List of 3D lists representing binary volumes.
                           Each element is a 3D grid of 0s and 1s.
                           All grids must have the same dimensions.
                           
    Returns:
        all_volume: 3D grid representing the intersection of all reference volumes.
        any_volume: 3D grid representing the union of all reference volumes.
    """
    if not reference_volumes:
        raise ValueError("Reference set R cannot be empty.")
        
    # Determine grid dimensions from the first volume
    dim_x = len(reference_volumes[0])
    dim_y = len(reference_volumes[0][0])
    dim_z = len(reference_volumes[0][0][0])
    
    # Initialize output volumes with zeros
    all_volume = [[[0 for _ in range(dim_z)] for _ in range(dim_y)] for _ in range(dim_x)]
    any_volume = [[[0 for _ in range(dim_z)] for _ in range(dim_y)] for _ in range(dim_x)]
    
    for x in range(dim_x):
        for y in range(dim_y):
            for z in range(dim_z):
                # Check occupancy across all reference volumes at this coordinate
                states = [vol[x][y][z] for vol in reference_volumes]
                
                # ALL: occupied if in all reference volumes
                all_volume[x][y][z] = 1 if all(states) else 0
                
                # ANY: occupied if in any reference volume
                any_volume[x][y][z] = 1 if any(states) else 0
                
    return all_volume, any_volume
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S1]
*   **Memory Complexity:** Not given in the sources. [S1]
*   **Strengths:** Simple, deterministic, and computationally inexpensive. It provides a clear, conservative (ALL) and permissive (ANY) boundary for expert performance [S1].
*   **Limitations:** It does not account for the frequency of expert agreement beyond binary presence/absence. It assumes all reference experts are equally valid [G].
*   **When to use:** When a set of expert annotations is available and a binary ground truth range (mandatory vs. allowed) is needed for evaluating test subjects [S1].
*   **When not to use:** Not given in the sources. [S1]

## Background
The method was introduced in the context of virtual mastoidectomy performance evaluation [S1]. It was developed to compare surgical resident performances (test set) against known expert performances (reference set) [S1]. The authors used 130 test volumes and 4 reference volumes to demonstrate the system's utility [S1]. The ALL volume represents mandatory bone removal areas (where all experts removed bone), and the ANY volume represents allowed removal areas (where at least one expert removed bone) [S1]. This approach accounts for variation in expert performance by defining maximal and minimal boundaries of the reference set [S1].

## Sources
[S1] Virtual mastoidectomy performance evaluation through multi-volume analysis., https://doi.org/10.1007/s11548-012-0687-4, fetched 2026-11-08
