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
*   $ht_{bit}$: The source text states $ht_{bit}=1$ in the formula description but $ht_{bit}=2$ in the definition and context of the 2b method. Given the method name "2b Header Tags", the value should be 2, but the source text contains an inconsistency [S1].

**Algorithm Steps:**
1.  **Decomposition:** Iteratively decompose active octants until an active octant of $2^3$ voxels is produced [S1].
2.  **Leaf Creation:** Form a leaf node (LNODE) for the $2^3$ voxel octant. This node does not undergo further decomposition [S1].
3.  **Parent Node Construction:** For a parent node at layer $l$, decompose it into eight sub-octants [S1].
4.  **Tag Assignment:** For each of the eight sub-octants:
    *   If the sub-octant is passive, assign HT code `00` [S1].
    *   If the sub-octant is active, assign one of the codes `01`, `10`, or `11` based on the required pointer type for the child node [S1].
5.  **Pointer Array Population:** For each active sub-octant, create a child node (LNODE or INODE) and add a pointer to the parent's pointer array. The pointer length corresponds to the assigned HT code's type (PT1, PT2, or PT3) [S1].
6.  **Alignment:** Apply alignment bits $r$ to the CHNM [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_svdag_2b_ht(scene_voxels):
    """
    Builds an SVDAG with 2-bit Header Tags.
    scene_voxels: 3D array of voxel data.
    Returns: Root node of the SVDAG.
    """
    # Constants
    HT_BITS = 2
    SUBOCTANTS = 8
    # Typical pointer lengths as per source context, though exact mapping of code to length
    # (e.g. 01->8b) is not specified in the provided text.
    # We assume a generic structure where pointer type determines length.
    POINTER_LENGTHS = {
        'PT1': 8,
        'PT2': 16,
        'PT3': 32
    }
    # Mapping of HT codes to pointer types is not explicitly defined in the text.
    # We assume a generic mapping for implementation purposes.
    HT_TO_TYPE = {
        '00': None, # Passive
        '01': 'PT1',
        '10': 'PT2',
        '11': 'PT3'
    }

    def is_active(octant):
        """Check if any voxel in the octant is active."""
        # Implementation depends on scene_voxels structure [G]
        return any(octant)

    def create_leaf_node(octant):
        """Create a leaf node for a 2x2x2 active octant."""
        # CHNM is 16 bits, all active or mixed, but no pointers [S1]
        chnm = 0xFFFF # Placeholder for voxel activity mask [G]
        return {'type': 'LNODE', 'chnm': chnm, 'pointers': []}

    def create_internal_node(octant):
        """Create an internal node with 2b HTs."""
        chnm_bits = []
        pointers = []
        
        # Decompose into 8 sub-octants
        sub_octants = decompose(octant)
        
        for i, sub in enumerate(sub_octants):
            if is_active(sub):
                # Determine pointer type based on some heuristic or fixed mapping
                # Source does not specify how PT1/PT2/PT3 are chosen for active nodes
                # We assume a default mapping for the sake of completeness [G]
                # In practice, this might depend on distance or layer [G]
                ptr_type = 'PT1' 
                code = '01' # Example mapping
            else:
                ptr_type = None
                code = '00'
            
            chnm_bits.append(code)
            
            if ptr_type:
                child_node = create_node(sub) # Recursive call
                ptr_len = POINTER_LENGTHS[ptr_type]
                # Store pointer with type info
                pointers.append({'type': ptr_type, 'length': ptr_len, 'target': child_node})
        
        # Construct CHNM from bits
        chnm = int(''.join(chnm_bits), 2)
        
        return {'type': 'INODE', 'chnm': chnm, 'pointers': pointers}

    def create_node(octant):
        if is_leaf_size(octant):
            return create_leaf_node(octant)
        else:
            return create_internal_node(octant)

    root = create_node(scene_voxels)
    return root

def decompose(octant):
    """Split octant into 8 sub-octants."""
    # Implementation depends on voxel grid indexing [G]
    pass

def is_leaf_size(octant):
    """Check if octant is 2x2x2."""
    # Implementation depends on octant size tracking [G]
    pass
```

**Edge Cases and Pitfalls:**
*   **CHNM Size Increase:** Using 2b HTs increases the CHNM length from 8b to 16b, which is a disadvantage compared to the compact 8b CHNM of the 1b HT method [S1].
*   **Source Inconsistency:** The formula for Leaf Node size references $ht_{bit}=1$ in the text description, while the method definition and context specify $ht_{bit}=2$. Implementers should use 2 based on the method name and CHNM size [S1].
*   **Pointer Type Mapping:** The source does not specify how the three active codes (`01`, `10`, `11`) are mapped to specific pointer lengths (PT1, PT2, PT3) or the criteria for choosing a pointer type. This must be determined by the implementation [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** The CHNM size is fixed at 16 bits per internal node, which is double the 8 bits of the 1b HT method [S1]. However, the use of variable-length pointers (PT1, PT2, PT3) can reduce the total size if shorter pointers are used for nearby nodes [S1].
*   **Measured Performance:** Not given in the sources for this specific configuration.
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
