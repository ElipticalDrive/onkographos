# Indirect Neighbor Search

Also called: Indirect neighbor computation, second-step neighbor search

## Overview
Indirect Neighbor Search computes the set of neighbors for the three faces of a cell $c$ that are not covered by direct neighbors. These neighbors are separated from $c$ by splitting planes located deeper in the octree hierarchy. While Direct Neighbor Search handles the immediate vicinity resulting from the last three splitting steps, Indirect Neighbor Search handles the remaining faces by traversing the tree structure to find cells or planes on the opposite side of bounding splitting planes. It is the second phase of the complete neighborhood computation in generalized octrees [S1].

## How it works
**Inputs:**
- Cell index $b_d$ (binary representation).
- The set of direct neighbors $N_{direct}(c)$ computed in the previous step.
- The octree structure (implicitly defined by the binary indices of nodes).

**Outputs:**
- The set of indirect neighbors $N_{indirect}(c)$.
- The complete neighborhood $N(c) = N_{direct}(c) \cup N_{indirect}(c)$.

**Data Structures:**
- Cell index $b_d$: A binary integer where every third bit corresponds to a dimension ($x_0, x_1, x_2$). The least significant bits correspond to the root level, and higher bits to deeper levels.
- $N(c)$: A set of indices (integers) representing splitting planes and leaf cells.

**Algorithm Steps:**

1.  **Identify Bounding Splitting Planes:**
    For each dimension $d \in \{0, 1, 2\}$, determine the splitting plane that bounds cell $c$ on the face not covered by direct neighbors.
    -   Extract the bits of $b_d$ corresponding to dimension $d$. This is done by considering every third bit starting from the least significant bit (LSB) for that dimension.
    -   Scan these bits from back to front (LSB to MSB, i.e., from root to leaf).
    -   Find the first bit swap (transition from 0 to 1 or 1 to 0) in this sequence.
    -   If the first swap occurs at bit position $j$ (where $j=0$ is the LSB), the index of the bounding splitting plane is $b_{d_j} = b_d \gg j$ (right shift by $j$).
    -   If no bit swap occurs in a dimension, the cell lies entirely on one side of all division planes in that dimension relative to the root, and there is no bounding plane in that direction for indirect neighbors [S1].

2.  **Recursive Search for Indirect Neighbors:**
    For each identified bounding plane $p$ found in Step 1:
    -   Perform a recursive search in the subtree starting from $p$ to find cells or planes on the opposite side of $p$ relative to $c$.
    -   The recursion logic for a `current_node` (which is a splitting plane) is as follows:
        1.  If `current_node` splits in the same dimension as $p$:
            -   Continue with the subtree of `current_node` that is adjacent to $c$ (the side containing $c$).
        2.  Else (if `current_node` splits in a different dimension):
            -   Check if `current_node` has a common point with $c$. This check requires comparing the coordinates of $c$ against the pivot value of the splitting plane.
            -   If they have a common point:
                -   Add `current_node` to $N(c)$.
                -   Continue with **both** subtrees of `current_node`.
            -   If they do not have a common point:
                -   Continue with the subtree of `current_node` that is adjacent to $c$.
    -   The recursion stops when leaf nodes (cells) are reached. These leaf cells are added to $N(c)$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def find_indirect_neighbors(cell_index_b, direct_neighbors):
    """
    Computes indirect neighbors for cell c with binary index b.
    Assumes direct_neighbors N(c) is already computed.
    """
    indirect_neighbors = set()
    
    # Step 1: Find bounding planes for the 3 dimensions
    # Dimensions are interleaved in b: bits 0,3,6... for dim 0; 1,4,7... for dim 1; 2,5,8... for dim 2
    bounding_planes = []
    
    for dim in range(3):
        # Extract bits for this dimension
        # We scan from LSB (bit 0 of the dimension's subsequence) upwards
        # The actual bit index in b is dim + 3*k
        current_b = cell_index_b
        swap_found = False
        shift_amount = 0
        
        # We need to find the first bit swap in the sequence of bits for this dimension
        # Let's extract the subsequence bits first
        dim_bits = []
        temp_b = cell_index_b
        while temp_b > 0:
            dim_bits.append((temp_b >> dim) & 1)
            temp_b >>= 3
            
        # Scan from LSB (index 0) to MSB
        first_swap_idx = -1
        for k in range(1, len(dim_bits)):
            if dim_bits[k] != dim_bits[k-1]:
                first_swap_idx = k
                break
        
        if first_swap_idx != -1:
            # The plane index is the cell index right-shifted by (3 * first_swap_idx)
            # Note: The source says "right shift by j" where j is the bit index in the subsequence.
            # In the full binary index, this corresponds to shifting by 3*j.
            plane_index = cell_index_b >> (3 * first_swap_idx)
            bounding_planes.append(plane_index)
        else:
            # No swap means no bounding plane in this direction for indirect neighbors
            bounding_planes.append(None)

    # Step 2: Recursive search for each bounding plane
    for plane in bounding_planes:
        if plane is None:
            continue
        
        # Start recursion from the plane itself. 
        # The source implies starting the search in the subtree defined by the plane.
        # We assume a function search_subtree(plane_node, cell_c)
        search_subtree(plane, cell_index_b, indirect_neighbors)

    return indirect_neighbors

def search_subtree(current_node, cell_index_b, neighbor_set):
    """
    Recursive search for indirect neighbors.
    current_node: index of a splitting plane.
    cell_index_b: binary index of the target cell c.
    neighbor_set: set to add found neighbors to.
    """
    # Base case: if current_node is a leaf (cell), add it.
    # In this binary encoding, leaves are typically distinguished or handled by the tree structure.
    # The source says "recursion stops in leaf nodes that represent cells belonging and being added to N(c)".
    # We need to know if current_node is a leaf. 
    # In the context of the source, splitting planes are internal nodes. 
    # If the tree structure is implicit, we might need a separate check or the caller handles it.
    # For the purpose of this algorithm description, we assume we can distinguish planes from cells.
    # However, the source focuses on planes. Let's assume current_node is a plane.
    
    # Determine the dimension of the split for current_node
    # The dimension is determined by the least significant bit that is 1 in the "dimension mask"
    # Or more simply, based on the bit position of the node in the global index.
    # The source says: "if (current plane splits in the same dimension as p)"
    # We need to know the dimension of 'current_node' and 'p'.
    
    # Let's define a helper to get the split dimension of a plane index.
    # The split dimension is determined by the position of the least significant bit 
    # that is part of the "plane index" structure. 
    # Actually, the source uses the property that the index determines the position.
    # A plane index $b_{d_j}$ has a specific structure.
    # Let's assume we have a function get_split_dimension(plane_index)
    
    dim_current = get_split_dimension(current_node)
    dim_p = get_split_dimension(p) # p is the initial bounding plane passed to this recursion context
    
    # Note: The source recursion logic is applied relative to the initial plane p of the face.
    # But the code snippet in the source is generic: "if (current plane splits in the same dimension as p)"
    # Here 'p' refers to the specific bounding plane for the face being processed.
    
    if dim_current == dim_p:
        # Continue with subtree next to c
        # We need to determine which child of current_node is next to c.
        # This depends on the bit of c in the dimension of current_node.
        # However, the source says "continue with subtree of current node next to c".
        # This implies we traverse down the side containing c.
        # But wait, we are looking for neighbors on the OPPOSITE side of p.
        # The source says: "Exploiting these properties we recursively search in the subtree starting from p."
        # And: "if (current plane splits in the same dimension as p) continue with subtree of current node next to c"
        # This seems to be pruning the search space.
        
        # To implement "subtree next to c", we need to know the bit of c in dim_current.
        # But we are searching for neighbors. The recursion logic in the source is:
        # 1. Same dim as p: go to child next to c.
        # 2. Diff dim: check common point.
        
        # Let's assume we have a function get_child_next_to_c(node, cell_index)
        next_node = get_child_next_to_c(current_node, cell_index_b)
        if next_node is not None:
            search_subtree(next_node, cell_index_b, neighbor_set)
            
    else:
        # Different dimension
        # Check if current plane has a common point with c
        if has_common_point(current_node, cell_index_b):
            neighbor_set.add(current_node)
            # Continue with both subtrees
            left_child = get_left_child(current_node)
            right_child = get_right_child(current_node)
            if left_child is not None:
                search_subtree(left_child, cell_index_b, neighbor_set)
            if right_child is not None:
                search_subtree(right_child, cell_index_b, neighbor_set)
        else:
            # Continue with subtree next to c
            next_node = get_child_next_to_c(current_node, cell_index_b)
            if next_node is not None:
                search_subtree(next_node, cell_index_b, neighbor_set)

def get_split_dimension(plane_index):
    """
    Determines the dimension (0, 1, or 2) split by the plane.
    Based on the binary index structure where every 3rd bit is a dimension.
    The dimension is determined by the least significant bit position modulo 3.
    """
    # Find the least significant bit that is 1
    # In the context of the source, the plane index is a prefix of the cell index.
    # The dimension is determined by the position of the bit that caused the split.
    # For a plane index $b_{d_j}$, the split dimension is determined by the bit position $j$ in the subsequence.
    # The actual bit position in the full index is $dim + 3*j$.
    # The dimension is simply $(bit_position) % 3$.
    # We need the position of the LSB of the plane index? 
    # No, the plane index represents a node. The split dimension of a node is fixed by its depth and position.
    # In the binary encoding, the dimension is determined by the least significant bit of the index?
    # Let's look at the example: $b_d = 1001000011001_d$.
    # The source says "considering every third bit".
    # The dimension of a split at bit $k$ (in the subsequence) is $k \% 3$? No.
    # The bits are interleaved: $b_0, b_1, b_2, b_3, b_4, b_5...$
    # $b_0$ is dim 0, $b_1$ is dim 1, $b_2$ is dim 2, $b_3$ is dim 0, etc.
    # So the dimension of the split at bit position $P$ in the full index is $P \% 3$.
    # The plane index is a prefix. The split dimension is determined by the LSB of the plane index?
    # No, the plane index is the index of the node. The node's split dimension is determined by its depth.
    # Depth $D$. The split dimension is $D \% 3$.
    # How to get depth from index?
    # The source doesn't give a direct formula for depth from index, but implies it via the bit scan.
    # For the purpose of the algorithm, we assume `get_split_dimension` is available.
    pass

def has_common_point(plane_index, cell_index_b):
    """
    Checks if the cell c has a common point with the splitting plane.
    This test uses the coordinates of c.
    """
    # This requires coordinate values. The source states:
    # "Only the test for which side of a plane cell c lies uses coordinates."
    # So this function must take coordinates.
    # Let's assume we have coordinates of c: (x, y, z)
    # And the plane's pivot value in its split dimension.
    # If the cell's coordinate in the split dimension is equal to the pivot, they share a point.
    # Or if the cell spans the pivot.
    # Since these are discrete cells, "common point" usually means the cell boundary touches the plane.
    pass

def get_child_next_to_c(node_index, cell_index_b):
    """
    Returns the child node of 'node_index' that is on the same side as cell c.
    """
    # Determine split dimension of node
    dim = get_split_dimension(node_index)
    # Get the bit of c in this dimension
    # The bit position in the full index for this dimension at this depth needs to be known.
    # This requires knowing the depth of the node.
    pass
```

**Parameters:**
- None explicitly defined. The algorithm is deterministic based on the tree structure.

**Edge Cases and Pitfalls:**
-   **No Bit Swap:** If no bit swap is found in a dimension during Step 1, there is no bounding plane in that direction, and no indirect neighbors are searched for that face [S1].
-   **Coordinate Dependency:** Only the test for whether a plane has a common point with $c$ uses the coordinates of $c$. All other operations (bit scanning, XOR, recursion path selection) use the binary representation of indices for speed [S1].
-   **Worst Case Complexity:** The worst-case number of indirect neighbors is $O(\sqrt[3]{n})$, where $n$ is the number of cells. This occurs only for a small number of cells [S1].
-   **Average Case:** In 3D, the average case has up to 9 neighbors per face, leading to 27 indirect neighbors per cell [S1].

## Subtopics

### Direct Neighbor Search
Computes the set of direct neighbors $N(c)$ for a cell $c$ with index $b_d$. Direct neighbors include splitting planes and cells resulting from the last three splitting steps. It identifies direct splitting planes at indices $b_{d1}, b_{d2}, b_{d3}$ (right shifts by 1, 2, 3 bits) and adjacent cells using bitwise XOR with $1_d, 10_d, 100_d$. It also checks four additional splitting planes beyond the direct ones using specific criteria to complete the neighborhood. This method covers half of the faces of $c$ and includes at least three and at most seven cells plus the respective splitting planes in between [S1].

## Operating characteristics
-   **Time Complexity:** The search performance is similar to nearest-neighbor search in kd-trees due to the heavy use of binary operations [S1]. The worst-case complexity for indirect neighbors is $O(\sqrt[3]{n})$ [S1].
-   **Memory Complexity:** Not given in the sources.
-   **Strengths:** Fast execution due to bitwise operations; avoids coordinate comparisons except for the final geometric test [S1].
-   **Limitations:** Worst-case complexity can be high for specific cell configurations, though rare [S1].
-   **When to use:** When a complete neighborhood of a cell in a generalized octree is required for isosurface extraction or other volume rendering techniques [S1].

## Background
The method was introduced by P. Rosenthal and L. Linsen in their 2006 paper "Direct Isosurface Extraction from Scattered Volume Data" [S1]. It is part of a two-step process for computing the neighborhood of a cell in a generalized octree, which is used for direct isosurface extraction from scattered volume data.

## Sources
[S1] Direct Isosurface Extraction from Scattered Volume Data, https://doi.org/10.2312/vissym/eurovis06/099-106, fetched 2026-10-26
