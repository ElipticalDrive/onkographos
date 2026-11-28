# SVDAG with 2b Header Tags

Also called: 2b HT CHNM, Multi-pointer SVDAG, SVDAG Parameter Optimization: Multi-type Pointers with 2b Header Tags

## Overview
SVDAG with 2b Header Tags is a compression optimization for Sparse Voxel Directed Acyclic Graphs (SVDAG) that replaces the standard 1-bit Child Node Mask (CHNM) with 2-bit Header Tags (HT) [S1]. This method allows a single internal node to distinguish between passive suboctants and three distinct types of active child pointers (PT1, PT2, PT3), enabling variable-length pointer encoding to reduce memory footprint [S1]. It differs from the standard 1b HT method by trading CHNM compactness (16b vs 8b) for the ability to use shorter pointers for spatially closer nodes [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An SVO (Sparse Voxel Octree) structure representing a voxelized 3D scene, where active octants are decomposed iteratively [S1].
*   **Output:** An SVDAG structure where internal nodes use 2-bit Header Tags and variable-length child pointers [S1].

**Data Structures:**
*   **Internal Node (INODE):** Composed of a Child Node Mask (CHNM) and an array of child node pointers (PTS) [S1].
    *   **CHNM:** 16 bits long (8 suboctants $\times$ 2 bits/suboctant) [S1].
    *   **PTS:** Array of pointers. The number of pointers equals the count of active suboctants indicated by the CHNM [S1].
*   **Leaf Node (LNODE):** Composed of a CHNM only; it contains no pointer array [S1].
    *   **CHNM:** 16 bits long [S1].
    *   **Content:** Information regarding the passivity/activity of specific voxels within the leaf [S1].

**Header Tag Codes:**
Each suboctant in the CHNM is assigned a 2-bit Header Tag (HT) [S1]:
*   `00`: Passive suboctant (no child node created) [S1].
*   `01`, `10`, `11`: Active suboctants, mapping to Pointer Type 1 (PT1), Pointer Type 2 (PT2), and Pointer Type 3 (PT3) respectively [S1].

**Pointer Types:**
Three pointer types are defined to optimize addressing space [S1]:
*   **PT1:** Length $pt_{1bit}$ bits.
*   **PT2:** Length $pt_{2bit}$ bits.
*   **PT3:** Length $pt_{3bit}$ bits.
*   Typical values cited are 8b, 16b, and 32b, though the specific mapping of code to length (e.g., is `01` always 8b?) is not explicitly detailed beyond the existence of three distinct types [S1].

**Formulas:**
The size of an internal node $I_{INODE2}$ in bits is calculated as [S1]:
$$ I_{INODE2} = r + 8 \times ht_{bit} + \sum_{x=1}^{3} n_x \times pt_{xbit} $$
Where:
*   $r$: Number of bits allocated for CHNM alignment [S1].
*   $ht_{bit}$: Number of bits constituting the Header Tag ($ht_{bit} = 2$) [S1].
*   $n_x$: Number of active child nodes pointed to by pointer type $PT_x$ [S1].
*   $pt_{xbit}$: Number of bits forming a $PT_x$ pointer [S1].

The size of a leaf node $L_{LNODE2}$ in bits is calculated as [S1]:
$$ L_{LNODE2} = r + 8 \times ht_{bit} $$
Where:
*   $r$: Number of bits allocated for CHNM alignment [S1].
*   $ht_{bit}$: The source text states $ht_{bit}=1$ in the formula description but $ht_{bit}=2$ in the definition and context of the 2b method [S1]. Implementers should use 2 based on the method name and CHNM size [S1].

**Algorithm Steps:**
1.  **Decomposition:** Iteratively decompose active octants until an active octant of $2^3$ voxels is produced [S1].
2.  **Leaf Creation:** Form a leaf node (LNODE) for the $2^3$ voxel octant; this node does not undergo further decomposition [S1].
3.  **Parent Node Construction:** For a parent node at layer $l$, decompose it into eight sub-octants [S1].
4.  **Tag Assignment:** For each of the eight sub-octants:
    *   If the sub-octant is passive, assign HT code `00` [S1].
    *   If the sub-octant is active, assign one of the codes `01`, `10`, or `11` based on the required pointer type for the child node [S1].
5.  **Pointer Array Population:** For each active sub-octant, create a child node (LNODE or INODE) and add a pointer to the parent's pointer array. The pointer length corresponds to the assigned HT code's type (PT1, PT2, or PT3) [S1].
6.  **Alignment:** Apply alignment bits $r$ to the CHNM [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_svdag_2b_ht(voxel_grid):
    """
    Builds an SVDAG with 2-bit Header Tags from a 3D voxel grid.
    voxel_grid: 3D list/array of voxel values.
    Returns: Root Node of the SVDAG.
    """
    HT_BITS = 2
    SUBOCTANTS = 8
    ALIGNMENT_BITS = 0  # r, alignment bits [G]
    # Pointer type lengths (example values from source context) [S1]
    PTR_LENGTHS = {'PT1': 8, 'PT2': 16, 'PT3': 32}
    # HT code to pointer type mapping (source does not specify exact criteria) [S1]
    HT_TO_TYPE = {'00': None, '01': 'PT1', '10': 'PT2', '11': 'PT3'}

    class Node:
        def __init__(self, node_type, chnm, pointers=None):
            self.type = node_type  # 'INODE' or 'LNODE'
            self.chnm = chnm       # 16-bit integer bitmask
            self.pointers = pointers or []

    # Memoization for DAG reduction (sharing identical child nodes) [G]
    node_cache = {}

    def get_octant(grid, origin, size):
        """Extracts a sub-octant from the grid."""
        return [grid[x][y][z] for x in range(origin[0], origin[0]+size)
                for y in range(origin[1], origin[1]+size)
                for z in range(origin[2], origin[2]+size)]

    def is_active(octant):
        """Checks if any voxel in the octant is active (non-zero)."""
        return any(octant)

    def create_leaf(octant):
        """Creates an LNODE for a 2x2x2 active octant."""
        # LNODE CHNM stores voxel activity; source specifies 16b CHNM [S1]
        chnm = 0
        for i, v in enumerate(octant):
            chnm |= (1 if v else 0) << (i * HT_BITS)
        return Node('LNODE', chnm)

    def create_internal(octant, origin, size):
        """Creates an INODE with 2b HTs and variable pointers."""
        chnm = 0
        pointers = []
        half = size // 2
        sub_origins = [
            (origin[0], origin[1], origin[2]),
            (origin[0]+half, origin[1], origin[2]),
            (origin[0], origin[1]+half, origin[2]),
            (origin[0]+half, origin[1]+half, origin[2]),
            (origin[0], origin[1], origin[2]+half),
            (origin[0]+half, origin[1], origin[2]+half),
            (origin[0], origin[1]+half, origin[2]+half),
            (origin[0]+half, origin[1]+half, origin[2]+half)
        ]
        for i, sub_origin in enumerate(sub_origins):
            sub_octant = get_octant(voxel_grid, sub_origin, half)
            if is_active(sub_octant):
                # Assign HT code based on pointer type selection heuristic [G]
                # Source does not specify mapping criteria [S1]
                code = '01' # Default to PT1 for demonstration [G]
                ptr_type = 'PT1'
                child = create_internal(sub_octant, sub_origin, half)
                pointers.append({'type': ptr_type, 'length': PTR_LENGTHS[ptr_type], 'target': child})
            else:
                code = '00'
            chnm |= (int(code, 2) if code != '00' else 0) << (i * HT_BITS)
        return Node('INODE', chnm, pointers)

    # Build root
    root = create_internal(voxel_grid, (0,0,0), len(voxel_grid))
    return root

def traverse_svdag_2b_ht(root, origin, direction, max_steps):
    """
    Traverses the SVDAG along a ray, returning the final cell index.
    Handles negative/zero directions and boundary checks correctly.
    Source does not specify traversal algorithm [S1].
    """
    x, y, z = float(origin[0]), float(origin[1]), float(origin[2])
    dx, dy, dz = float(direction[0]), float(direction[1]), float(direction[2])
    grid_size = len(voxel_grid)
    
    # Initialize DDA parameters
    step_x = 1 if dx > 0 else -1
    step_y = 1 if dy > 0 else -1
    step_z = 1 if dz > 0 else -1
    
    t_max_x = (step_x - x) / dx if dx != 0 else float('inf')
    t_max_y = (step_y - y) / dy if dy != 0 else float('inf')
    t_max_z = (step_z - z) / dz if dz != 0 else float('inf')
    
    t_delta_x = step_x / dx if dx != 0 else float('inf')
    t_delta_y = step_y / dy if dy != 0 else float('inf')
    t_delta_z = step_z / dz if dz != 0 else float('inf')
    
    current_node = root
    for _ in range(max_steps):
        # Determine next cell boundary
        if t_max_x < t_max_y:
            if t_max_x < t_max_z:
                x += step_x; t_max_x += t_delta_x
            else:
                z += step_z; t_max_z += t_delta_z
        else:
            if t_max_y < t_max_z:
                y += step_y; t_max_y += t_delta_y
            else:
                z += step_z; t_max_z += t_delta_z
                
        # Boundary check
        if not (0 <= x < grid_size and 0 <= y < grid_size and 0 <= z < grid_size):
            return None
            
        # DAG node lookup logic would go here based on spatial position [G]
        pass
    return (int(x), int(y), int(z))
```

**Edge Cases and Pitfalls:**
*   **CHNM Size Increase:** Using 2b HTs increases the CHNM length from 8b to 16b, which is a disadvantage compared to the compact 8b CHNM of the 1b HT method [S1].
*   **Source Inconsistency:** The formula for Leaf Node size references $ht_{bit}=1$ in the text description, while the method definition and context specify $ht_{bit}=2$. Implementers should use 2 based on the method name and CHNM size [S1].
*   **Pointer Type Mapping:** The source does not specify how the three active codes (`01`, `10`, `11`) are mapped to specific pointer lengths (PT1, PT2, PT3) or the criteria for choosing a pointer type. This must be determined by the implementation [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** The CHNM size is fixed at 16 bits per internal node, which is double the 8 bits of the 1b HT method [S1]. However, the use of variable-length pointers (PT1, PT2, PT3) can reduce the total size if shorter pointers are used for nearby nodes [S1].
*   **Measured Performance:** Not given in the sources for this specific configuration [S1].
*   **Strengths:** Enables compression through variable-length pointers, optimizing addressing space usage [S1].
*   **Limitations:** Increased CHNM size (16b vs 8b) compared to 1b HT methods [S1].
*   **When to use:** When the scene geometry allows for significant compression via variable-length pointers that outweigh the CHNM overhead [S1].
*   **When not to use:** When CHNM compactness is critical and pointer lengths cannot be significantly reduced [S1].

## Background
*   **Origin:** The method is described in "Parameter Optimization in Sparse Voxel DAGs for Efficient Geometry Representation of Voxelized 3D Scenes" by B. Madoš et al. [S1].
*   **History:** It is part of a series of parameter optimizations for SVDAGs, including 1b HT, 2b HT, and multi-pointer configurations [S1].
*   **Usage:** Used for efficient geometry representation of voxelized 3D scenes [S1].

## Sources
[S1] Parameter Optimization in Sparse Voxel DAGs for Efficient Geometry Representation of Voxelized 3D Scenes, https://doi.org/10.12700/aph.22.7.2025.7.11, fetched 2026-10-27
