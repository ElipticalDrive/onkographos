# Octant-based Black Component Counting

Also called: Octant flooding, Octant occupancy lookup

## Overview
This method efficiently counts the number of black (foreground) components in the 26-neighborhood $N_{26}(P)$ of a voxel $P$ to determine if $P$ is a simple point for topology-preserving thinning [S1, unofficial]. It differs from direct voxel flooding by aggregating voxels into 8 overlapping octants, using a 256-entry lookup table to reject non-simple points immediately and reducing the subsequent flooding search space from 26 voxels to 8 octants [S1, unofficial].

## How it works
**Inputs:** Binary voxel grid; current voxel $P$ at coordinates $(x,y,z)$.
**Outputs:** Integer count of black components in $N_{26}(P)$; boolean indicating if $P$ is simple (count == 1).

**Data Structures:**
1.  **Octant Map:** A 256-entry lookup table (LUT). Each entry corresponds to a unique 8-bit occupancy mask of the 8 octants. The entry stores either `INVALID` (if the configuration implies >1 component) or a flag indicating `NEEDS_FLOODING`.
2.  **Octant Indices:** The 26 neighbors are mapped to 8 overlapping octants. Let the octants be indexed $0$ to $7$. An octant is "occupied" if at least one voxel within it is black.

**Steps:**
1.  **Divide Neighborhood:** Partition $N_{26}(P)$ into 8 overlapping octants. Each octant consists of a subset of the 26 neighbors.
2.  **Compute Occupancy Mask:** For each octant $i \in \{0, \dots, 7\}$, check if any voxel in that octant is black. Construct an 8-bit integer $M$ where the $i$-th bit is 1 if octant $i$ is occupied, and 0 otherwise.
3.  **Lookup Table Check:** Access the LUT at index $M$.
    *   If the LUT entry indicates the configuration implies $>1$ black component, return `count > 1` (voxel $P$ is not simple). No further processing is required [S1, unofficial].
    *   If the LUT entry indicates `NEEDS_FLOODING`, proceed to Step 4.
4.  **Octant Flooding:** Perform a connectivity check on the occupied octants rather than individual voxels.
    *   **Octant Connectivity:** Two octants $W_1$ and $W_2$ are connected neighbors if they share a common face, edge, or vertex, AND at least one voxel in the intersection (face, edge, or vertex) is black [S1, unofficial].
    *   **Transitive Connection:** Octants $W_a$ and $W_b$ are connected if there exists a path of connected neighbor octants between them.
    *   **Counting:** The number of black components equals the number of distinct connected groups of occupied octants. This is computed via a flood-fill or union-find algorithm on the graph of octants [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def count_black_components(voxel_grid, P):
    # 1. Define the 8 octants as sets of neighbor indices relative to P
    # Each octant is a subset of the 26 neighbors.
    # Example mapping (indices 0-25 relative to P):
    octants = [
        {0, 1, 2, 3, 8, 9, 10, 11, 16, 17, 18, 19}, # Octant 0
        {4, 5, 6, 7, 12, 13, 14, 15, 20, 21, 22, 23}, # Octant 1
        # ... define all 8 octants covering the 26 neighbors ...
    ]
    
    # 2. Compute occupancy mask
    mask = 0
    for i, octant_voxels in enumerate(octants):
        is_occupied = False
        for v in octant_voxels:
            if voxel_grid.is_black(v):
                is_occupied = True
                break
        if is_occupied:
            mask |= (1 << i)
            
    # 3. Lookup Table Check
    # LUT_PRECOMPUTED[mask] returns:
    #   -1 if configuration implies >1 component
    #   0 if configuration implies 1 component (simple)
    #   1 if further flooding is needed
    lut_result = LUT_PRECOMPUTED[mask]
    
    if lut_result == -1:
        return 2 # Not simple
    if lut_result == 0:
        return 1 # Simple
        
    # 4. Octant Flooding
    # Build adjacency graph of occupied octants
    occupied_octants = [i for i in range(8) if (mask >> i) & 1]
    if not occupied_octants:
        return 0 # No black components
        
    # Define connectivity between octants
    # Two octants are connected if they share a face/edge/vertex 
    # AND a voxel in that intersection is black.
    def are_octants_connected(o1, o2):
        # Find intersection voxels
        intersection = octants[o1].intersection(octants[o2])
        for v in intersection:
            if voxel_grid.is_black(v):
                return True
        return False
        
    # Count connected components of octants
    visited = set()
    components = 0
    for start_oct in occupied_octants:
        if start_oct in visited:
            continue
        components += 1
        # BFS/DFS
        stack = [start_oct]
        visited.add(start_oct)
        while stack:
            curr = stack.pop()
            for next_oct in occupied_octants:
                if next_oct not in visited and are_octants_connected(curr, next_oct):
                    visited.add(next_oct)
                    stack.append(next_oct)
                    
    return components
```

**Edge Cases and Pitfalls:**
*   **Empty Neighborhood:** If no voxels in $N_{26}(P)$ are black, the count is 0. The LUT should handle the all-zero mask appropriately [S1, unofficial].
*   **Precision:** The method relies on exact boolean occupancy. Floating-point errors are not applicable as input is binary [G].
*   **Boundary Conditions:** The definition of $N_{26}(P)$ assumes $P$ has 26 neighbors. For voxels on the volume boundary, the neighborhood is truncated, and the octant definitions must be adjusted to only include valid neighbors [G].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for the lookup phase. The flooding phase is bounded by the constant size of 8 octants, making it effectively $O(1)$ per voxel, significantly faster than voxel-level flooding which scales with the number of black voxels in the neighborhood [S1, unofficial].
*   **Memory Complexity:** Requires a 256-entry lookup table (constant size) and storage for the 8 octant definitions (constant size) [S1, unofficial].
*   **Strengths:** High efficiency for thinning algorithms where simple point detection is performed on every voxel. Reduces the search space for connectivity checks [S1, unofficial].
*   **Limitations:** Requires precomputation of the LUT based on the specific octant geometry and connectivity rules. Less flexible if connectivity definitions change (e.g., 6-connectivity vs 26-connectivity) [S1, unofficial].

## Background
This method was developed to address the computational cost of determining simple points in 3D image thinning, specifically for applications in die casting part visualization [S1, unofficial]. Traditional methods used modified flooding algorithms to connect voxels directly, which were time-consuming due to the large number of voxels involved [S1, unofficial]. The approach leverages the Euler characteristic and the requirement to count black components in $N_{26}(P)$ [S1, unofficial].

## Sources
[S1] Simple visualization techniques for die casting part and die design. Final report, July 1, 1995--September 30, 1997, https://digital.library.unt.edu/ark:/67531/metadc690873/?q=voxelization triangle mesh, fetched 2026-10-28 (unofficial)
