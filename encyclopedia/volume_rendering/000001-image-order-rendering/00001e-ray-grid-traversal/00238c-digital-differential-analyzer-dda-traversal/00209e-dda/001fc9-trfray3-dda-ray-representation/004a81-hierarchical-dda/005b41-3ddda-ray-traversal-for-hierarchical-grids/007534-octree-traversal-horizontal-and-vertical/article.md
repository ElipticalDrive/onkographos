# Octree Traversal (Horizontal and Vertical)
Also called: 3DDDA Octree Traversal

## Overview
This method defines the traversal logic for hierarchical octree structures using the 3DDDA ray representation. It solves the problem of efficiently navigating a sparse, multi-resolution volume representation by distinguishing between movement within a single node (horizontal traversal) and movement between nodes of different levels (vertical traversal). It sets itself apart from simple cell decomposition by introducing vertical descent and ascent mechanisms that utilize 3DDDA state variables, while reusing the horizontal traversal logic unchanged.

## How it works
The method relies on the 3DDDA ray representation [Parent](16833-3ddda-ray-traversal-for-hierarchical-grids.md). It operates on an octree where nodes are classified as HETERO-BRANCH, HETERO-LEAF, or HOMO.

### Data Structures
*   **Node Numbering**: Octants within a node are ordered systematically [S1]. Each digit of the binary node number corresponds to a cell index in the cell decomposition. For a single node, there are exactly two octants in each principal direction ($x, y, z$). The indices satisfy $0 \leq i, j, k \leq 1$ [S1].
*   **Pointer Array**: An array `POINTER` stores pointers to child nodes [S1].
*   **Working Array**: A short working array of length equal to the number of levels in the tree. It stores pointers recorded during descent to facilitate ascent [S1].
*   **3DDDA State Variables**:
    *   Control terms (typically related to distance to next grid plane).
    *   Inverse slopes (reciprocals of ray direction components).
    *   Octant size (current voxel size at the current level) [S1].

### Algorithm Steps

1.  **Horizontal Traversal (Intra-node)**:
    *   Traverse octants within the current node exactly as in a simple cell-decomposed structure [S1].
    *   When moving from one cell to an adjacent cell, update exactly one of the three indices ($i, j, k$) [S1].
    *   The new octant number is derived directly from the updated indices [S1].
    *   **Termination**: Horizontal traversal terminates when any index overflows or underflows (i.e., becomes $<0$ or $>1$), indicating the ray has left the current node [S1].

2.  **Vertical Traversal (Descent)**:
    *   Triggered when 3DDDA identifies a HETERO-BRANCH node [S1].
    *   Suspend 3DDDA execution [S1].
    *   Traverse downwards until a HETERO-LEAF or HOMO node is reached [S1].
    *   Determine the child node to enter using the current 3DDDA byproducts: control terms, inverse slopes, and octant size [S1].
    *   **State Update**: On each vertical movement from one level to the next, multiply or divide the relevant state variables by two. This is performed using fixed-point arithmetic (bit shifts) [S1].
    *   Record the pointer from the `POINTER` array into the working array at the current level index [S1].

3.  **Vertical Traversal (Ascent)**:
    *   Triggered when horizontal traversal terminates due to index overflow/underflow [S1].
    *   Ascend the tree to identify the adjacent node [S1].
    *   Use the working array (populated during descent) to retrieve the correct parent pointers. The pointers are not linked backwards in the tree structure itself, so the working array is essential [S1].
    *   Resume 3DDDA execution in the new node [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_octree(ray_origin, ray_direction, root_node, tree_levels):
    # Initialize 3DDDA state variables based on ray and root node size
    # These include control terms, inverse slopes, and current octant size
    dda_state = init_3ddda(ray_origin, ray_direction, root_node.size)
    
    # Working array to store pointers during descent
    # Length equals number of levels in the tree
    pointer_stack = [None] * tree_levels
    current_level = 0
    
    current_node = root_node
    
    while True:
        # --- Horizontal Traversal ---
        # Indices for the 2x2x2 octants within the current node
        i, j, k = 0, 0, 0
        
        while True:
            # Process current octant (i, j, k)
            octant_index = (i << 2) | (j << 1) | k # Example mapping, see Yamaguchi order
            process_voxel(current_node, octant_index, dda_state)
            
            # Determine next octant in horizontal sequence
            # Update one index at a time based on systematic order
            next_i, next_j, next_k = get_next_octant_indices(i, j, k)
            
            # Check for boundary exit (overflow/underflow)
            if next_i < 0 or next_i > 1 or next_j < 0 or next_j > 1 or next_k < 0 or next_k > 1:
                break # Horizontal traversal terminated
            
            i, j, k = next_i, next_j, next_k
        
        # --- Vertical Traversal (Ascent or Descent Decision) ---
        # If we exited horizontally, we must ascend to find the adjacent node
        # If we hit a HETERO-BRANCH during processing, we must descend
        
        # Note: The specific trigger for descent vs ascent depends on the 
        # node type encountered at the exit point or during processing.
        
        if node_is_hetero_branch(current_node):
            # --- Descent ---
            # Suspend 3DDDA
            suspended_state = dda_state
            
            # Record pointer for ascent later
            pointer_stack[current_level] = current_node.pointer_to_child
            
            # Determine which child to enter using 3DDDA byproducts
            child_index = determine_child(dda_state)
            
            # Move to child
            current_node = current_node.children[child_index]
            current_level += 1
            
            # Update state variables for new level (fixed-point mult/div by 2)
            dda_state = update_state_for_level(dda_state, descent=True)
            
            # Resume 3DDDA in new node
            # Loop continues to process horizontal traversal in new node
            
        elif horizontal_exit_occurred:
            # --- Ascent ---
            # Use pointer_stack to find parent
            if current_level == 0:
                break # Reached root, ray is done
            
            # Retrieve parent pointer from stack
            parent_pointer = pointer_stack[current_level - 1]
            
            # Move up
            current_node = parent_pointer.parent
            current_level -= 1
            
            # Update state variables for new level (fixed-point mult/div by 2)
            dda_state = update_state_for_level(dda_state, descent=False)
            
            # Determine which sibling to enter next
            # This requires re-evaluating the exit position relative to the parent
            next_sibling = find_adjacent_sibling(current_node, dda_state)
            current_node = next_sibling
            
            # Resume 3DDDA
```

### Parameters and Edge Cases
*   **Fixed-Point Arithmetic**: Multiplication/division by two during vertical traversal is implemented as bit shifts to avoid floating-point operations [S1].
*   **Precision**: The method avoids floating-point division or multiplication during horizontal traversal [S1].
*   **Node Types**: The algorithm distinguishes between HETERO-BRANCH (requires descent), HETERO-LEAF (leaf with data), and HOMO (uniform) nodes [S1].

## Operating characteristics
*   **Complexity**: Horizontal traversal is identical to simple cell decomposition, requiring no modifications to 3DDDA [S1]. Vertical traversal adds overhead for descent/ascent but avoids expensive floating-point operations by using fixed-point arithmetic [S1].
*   **Performance**: Initial experiments on a VAX 11/750 showed significant optimization potential by removing subroutine calls and using integer logic [S1].
*   **Strengths**: Efficient traversal of hierarchical structures; avoids floating-point division/multiplication in the main loop [S1].
*   **Limitations**: Requires maintaining a working array for ascent; complex logic for determining child nodes during descent [S1].

## Background
The method was proposed by Yamaguchi et al. [S1]. It was implemented in FORTRAN 77 and tested on a Digital Equipment Corporation VAX 11/750 [S1]. The systematic ordering of octants was adopted from Yamaguchi et al. [S1].

## Sources
[S1] Arts: Accelerated ray-tracing system, https://ieeexplore.ieee.org/abstract/document/4056861/, fetched 2026-10-25
