# Indirect Neighbor Search


## Overview
Indirect Neighbor Search computes the set of neighbors for the three faces of a cell $c$ that are not covered by direct neighbors [S1]. These neighbors are separated from $c$ by splitting planes located deeper in the octree hierarchy [S1]. While Direct Neighbor Search handles the immediate vicinity resulting from the last three splitting steps, Indirect Neighbor Search handles the remaining faces by traversing the tree structure to find cells or planes on the opposite side of bounding splitting planes [S1]. It is the second phase of the complete neighborhood computation in generalized octrees [S1].

## How it works
**Inputs:**
- Cell index $b_d$ (binary representation) [S1].
- The set of direct neighbors $N_{direct}(c)$ computed in the previous step [S1].
- Cell coordinates $(x, y, z)$ for geometric tests [G].

**Outputs:**
- The set of indirect neighbors $N_{indirect}(c)$ [S1].
- The complete neighborhood $N(c) = N_{direct}(c) \cup N_{indirect}(c)$ [S1].

**Data Structures:**
- Cell index $b_d$: A binary integer where every third bit corresponds to a dimension ($x_0, x_1, x_2$) [S1]. The least significant bits correspond to the root level, and higher bits to deeper levels [S1].
- $N(c)$: A set of indices (integers) representing splitting planes and leaf cells [S1].

**Algorithm Steps:**

1. **Identify Bounding Splitting Planes:**
   For each dimension $d \in \{0, 1, 2\}$, determine the splitting plane that bounds cell $c$ on the face not covered by direct neighbors [S1].
   - Extract the bits of $b_d$ corresponding to dimension $d$ by considering every third bit starting from the least significant bit (LSB) for that dimension [S1].
   - Scan these bits from LSB to MSB (root to leaf) [S1].
   - Find the first bit swap (transition from 0 to 1 or 1 to 0) in this sequence [S1].
   - If the first swap occurs at subsequence index $j$ (where $j=0$ is the LSB), the index of the bounding splitting plane is $b_{d_j} = b_d \gg (3j)$ (right shift by $3j$) [S1].
   - If no bit swap occurs in a dimension, the cell lies entirely on one side of all division planes in that dimension relative to the root, and there is no bounding plane in that direction for indirect neighbors [S1].

2. **Recursive Search for Indirect Neighbors:**
   For each identified bounding plane $p$ found in Step 1:
   - Perform a recursive search in the subtree starting from $p$ to find cells or planes on the opposite side of $p$ relative to $c$ [S1].
   - The recursion logic for a `current_node` (which is a splitting plane) is as follows [S1]:
     1. If `current_node` splits in the same dimension as $p$:
        - Continue with the subtree of `current_node` that is adjacent to $c$ (the side containing $c$) [S1].
     2. Else (if `current_node` splits in a different dimension):
        - Check if `current_node` has a common point with $c$. This check requires comparing the coordinates of $c$ against the pivot value of the splitting plane [S1].
        - If they have a common point:
          - Add `current_node` to $N(c)$ [S1].
          - Continue with **both** subtrees of `current_node` [S1].
        - If they do not have a common point:
          - Continue with the subtree of `current_node` that is adjacent to $c$ [S1].
   - The recursion stops when leaf nodes (cells) are reached. These leaf cells are added to $N(c)$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def find_indirect_neighbors(cell_index, cell_coords, direct_neighbors):
    """
    Computes indirect neighbors for cell c with binary index b.
    Assumes direct_neighbors N(c) is already computed.
    """
    indirect_neighbors = set()
    
    # Dimensions are interleaved in Morton code: bits 0,3,6... for dim 0; 1,4,7... for dim 1; 2,5,8... for dim 2
    for dim in range(3):
        # Extract bits for this dimension
        bits = []
        temp = cell_index
        while temp > 0:
            bits.append((temp >> dim) & 1)
            temp >>= 3
            
        # Scan from LSB (index 0) to MSB to find first swap
        first_swap_idx = -1
        for k in range(1, len(bits)):
            if bits[k] != bits[k-1]:
                first_swap_idx = k
                break
                
        if first_swap_idx != -1:
            # The plane index is the cell index right-shifted by 3*j
            plane_index = cell_index >> (3 * first_swap_idx)
            # Step 2: Recursive search
            search_subtree(plane_index, cell_index, cell_coords, indirect_neighbors, dim)
            
    return indirect_neighbors

def search_subtree(current_plane, cell_index, cell_coords, neighbor_set, dim_p):
    """
    Recursive search for indirect neighbors.
    current_plane: index of a splitting plane.
    cell_index: binary index of the target cell c.
    cell_coords: (x, y, z) coordinates of c.
    neighbor_set: set to add found neighbors to.
    dim_p: split dimension of the initial bounding plane p.
    """
    # [G] Leaf detection: check tree-specific flag or mask indicating a leaf node
    if is_leaf(current_plane):
        neighbor_set.add(current_plane)
        return

    # Determine split dimension of current_plane (Morton code: dim 0 at bit 0, dim 1 at bit 1, dim 2 at bit 2)
    dim_current = current_plane & 3
    
    if dim_current == dim_p:
        # Continue with subtree next to c
        next_node = get_child_towards_c(current_plane, cell_coords)
        if next_node is not None:
            search_subtree(next_node, cell_index, cell_coords, neighbor_set, dim_p)
    else:
        # Different dimension: check if current plane has a common point with c
        if has_common_point(current_plane, cell_coords):
            neighbor_set.add(current_plane)
            # Continue with both subtrees
            left_child = current_plane << 1
            right_child = (current_plane << 1) | 1
            search_subtree(left_child, cell_index, cell_coords, neighbor_set, dim_p)
            search_subtree(right_child, cell_index, cell_coords, neighbor_set, dim_p)
        else:
            # Continue with subtree next to c
            next_node = get_child_towards_c(current_plane, cell_coords)
            if next_node is not None:
                search_subtree(next_node, cell_index, cell_coords, neighbor_set, dim_p)

def get_child_towards_c(node_index, cell_coords):
    """
    Returns the child node of 'node_index' that is on the same side as cell c.
    Uses coordinate comparison as specified by the source [S1].
    """
    dim = node_index & 3
    coord = cell_coords[dim]
    # [G] Retrieve pivot value for the splitting plane; implementation depends on tree storage
    pivot = get_pivot(node_index)
    if coord < pivot:
        return node_index << 1
    else:
        return (node_index << 1) | 1

def has_common_point(plane_index, cell_coords):
    """
    Checks if the cell c has a common point with the splitting plane.
    This test uses the coordinates of c when comparing it to the pivot value of the splitting plane [S1].
    """
    dim = plane_index & 3
    coord = cell_coords[dim]
    pivot = get_pivot(plane_index)
    # [G] Exact boundary check
    return abs(coord - pivot) < 1e-9
```

**Parameters:**
- None explicitly defined. The algorithm is deterministic based on the tree structure [S1].

**Edge Cases and Pitfalls:**
- **No Bit Swap:** If no bit swap is found in a dimension during Step 1, there is no bounding plane in that direction, and no indirect neighbors are searched for that face [S1].
- **Coordinate Dependency:** Only the test for whether a plane has a common point with $c$ uses the coordinates of $c$. All other operations (bit scanning, XOR, recursion path selection) use the binary representation of indices for speed [S1].
- **Worst Case Complexity:** The worst-case number of indirect neighbors is $O(\sqrt[3]{n^2})$, where $n$ is the number of cells. This occurs only for a small number of cells [S1].
- **Average Case:** In 3D, the average case has up to 9 neighbors per face, leading to 27 indirect neighbors per cell [S1].

## Subtopics

### Direct Neighbor Search
Computes the set of direct neighbors $N(c)$ for a cell $c$ with index $b_d$ [S1]. Direct neighbors include splitting planes and cells resulting from the last three splitting steps [S1]. It identifies direct splitting planes at indices $b_{d1}, b_{d2}, b_{d3}$ (right shifts by 1, 2, 3 bits) and adjacent cells using bitwise XOR with $1_d, 10_d, 100_d$ [S1]. It also checks four additional splitting planes beyond the direct ones using specific criteria to complete the neighborhood [S1]. This method covers half of the faces of $c$ and includes at least three and at most seven cells plus the respective splitting planes in between [S1].

## Operating characteristics
- **Time Complexity:** The search performance is similar to nearest-neighbor search in kd-trees due to the heavy use of binary operations [S1]. The worst-case complexity for indirect neighbors is $O(\sqrt[3]{n^2})$ [S1].
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Fast execution due to bitwise operations; avoids coordinate comparisons except for the final geometric test [S1].
- **Limitations:** Worst-case complexity can be high for specific cell configurations, though rare [S1].
- **When to use:** When a complete neighborhood of a cell in a generalized octree is required for isosurface extraction or other volume rendering techniques [S1].

## Background
The method was introduced by P. Rosenthal and L. Linsen in their 2006 paper "Direct Isosurface Extraction from Scattered Volume Data" [S1]. It is part of a two-step process for computing the neighborhood of a cell in a generalized octree, which is used for direct isosurface extraction from scattered volume data [S1].

## Sources
[S1] Direct Isosurface Extraction from Scattered Volume Data, https://doi.org/10.2312/vissym/eurovis06/099-106, fetched 2026-10-26
