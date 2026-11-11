# Restart Trail

Also called: Generalized Restart Trail, Restart Trail for Wide BVHs, Short Stack BVH Traversal, Short stack traversal with restart trail, Wide BVH Traversal with Short Stack, Wide BVH Traversal with a Short Stack

## Overview
Restart Trail is a stackless or short-stack traversal algorithm for Bounding Volume Hierarchies (BVHs) that enables traversal restarts from the root by storing one bit of data per hierarchy level to track which parts of the tree have been processed [S2, unofficial]. It sets itself apart from standard short-stack traversal by allowing the stack to be completely exhausted without losing traversal state, using a compact bit-array (the "trail") to resume from the correct point, whereas methods like [Stack Culling](9874-stack-culling-parent-node-re-intersection.md) trade a small increase in traversal steps for reduced primitive intersections using only 1 bit of extra storage per stack entry [S1, unofficial]. It solves the problem of memory-constrained traversal (e.g., in hardware or GPUs) where a full stack is too large, by replacing the stack with a fixed-size short stack and a restart mechanism that avoids re-entering already processed subtrees.

## How it works
**Inputs and Outputs:**
- Input: A ray, an N-wide BVH (where $N$ is the number of children per internal node), and a short stack of fixed size $S$ (typically 5 entries).
- Output: The closest intersection (for closest-hit) or all intersections (for any-hit), traversing the BVH.

**Data Structures:**
1.  **Restart Trail (`trail`):** An array of bits (or counters for wide BVHs) with one entry per hierarchy level.
    -   For binary BVHs (Laine [S2, unofficial]): `trail` is a single register (`trailReg`) of 32 or 64 bits. `trail[0]` is a sentinel. `trail[i]` corresponds to the node at level $i$.
    -   For N-wide BVHs (Scheuermann et al. [S1, unofficial]): `trail[i]` is a counter ranging from $0$ to $N-1$. `trail[i]` stores the number of children at level $i$ already processed. $N-1$ indicates the current subtree is the last child at that level.
2.  **Level Register (`levelReg`):** A register with exactly one bit set, corresponding to the bit position in `trailReg` pointing to the parent of the current node. Initialized to `10000...` (1 followed by zeros) for the sentinel [S2, unofficial].
3.  **Short Stack:** A fixed-size array of node references. Each entry may include a reserved bit to mark if the entry corresponds to the last child of its parent node [S1, unofficial].
4.  **Pop Level (`popLevel`):** An integer tracking the level where the last stack pop would have taken the traversal. Initialized to NONE [S2, unofficial].

**Parameters:**
-   $N$: The width of the BVH (number of children per internal node).
-   Short Stack Size: A small fixed number of entries (e.g., 5) stored in on-chip memory [S1, unofficial].

**Steps:**

1.  **Initialization:**
    -   Set `node` to `ROOT`.
    -   Set `trail` to all zeros (or counters to 0).
    -   Set `level` to 0.
    -   Set `shortstack` to empty.
    -   Set `popLevel` to NONE [S1, unofficial][S2, unofficial].

2.  **Traversal Loop:**
    While the current `node` is not a leaf:
    a.  **Intersect Children:** Intersect the ray against all children of `node`. Let $H$ be the list of intersecting children.
    b.  **Sort:** Sort $H$ by increasing hit distance to obtain sorted list $S$ [S1, unofficial].
    c.  **Handle Multiple Hits (Binary BVH Logic [S2, unofficial]):**
        -   If both children are intersected:
            -   Identify `near` child (smaller $t_{min}$) and `far` child (greater $t_{min}$).
            -   Increment `level` (shift `levelReg` right by 1).
            -   If `trail[level]` is 1 (bit set): Set `node` to `far`. (The near child has been fully processed or skipped).
            -   Else: Set `node` to `near`. Push `far` onto the short stack.
        -   If only one child is intersected:
            -   Increment `level`.
            -   If `level` $\neq$ `popLevel`: Set `trail[level]` to 1. Set `node` to the intersected child.
            -   Else (`level` == `popLevel`): Invoke `POP()`. This handles the case where the far child was culled during the processing of the near child [S2, unofficial].
        -   If no children are intersected:
            -   Invoke `POP()` [S2, unofficial].
    d.  **Handle Multiple Hits (N-wide BVH Logic [S1, unofficial]):**
        -   Let $k = \text{trail}[\text{level}]$.
        -   If $k = N-1$: Remove all but the last entry in $S$ to form $Q$. (Current subtree is the last one).
        -   Else: Remove the first $k$ entries in $S$ to form $Q$. (Skip already processed children).
        -   If $Q$ is empty: Call `POP(trail, level)`.
        -   Else:
            -   Set `node` to the first entry in $Q$.
            -   Remove the first entry from $Q$.
            -   If $Q$ is empty: Set `trail[level]` to $N-1$.
            -   Else: Mark the last entry in $Q$ as the "last child" and push remaining entries in $Q$ onto the short stack (back-to-front order) [S1, unofficial].

3.  **Leaf Processing:**
    -   Intersect the ray against primitives in the leaf `node`.
    -   If a closer primitive is found, shorten the ray at the end.
    -   Invoke `POP()` [S1, unofficial][S2, unofficial].

4.  **POP() Operation:**
    -   **Find Parent Level:** Scan the restart trail upwards from the current `level`. Skip levels with the max count value ($N-1$ for wide BVHs, or bit=1 for binary BVHs where it indicates completion). Find the largest $i \le \text{level}$ such that `trail[i]` is 0 (binary) or $< N-1$ (wide). If no such level exists (all scanned levels are maxed), the stack is empty and traversal is complete.
    -   **Update Trail:**
        -   Increment `trail[parentLevel]` (wide BVH) or set `trail[parentLevel]` to 1 (binary BVH).
        -   Clear all counters/bits below `parentLevel` to 0.
    -   **Update State:**
        -   If short stack is empty: Set `node` to `ROOT`, `level` to 0. Restart traversal.
        -   Else: Pop from short stack.
            -   If popped node is tagged as "last child": Set `trail[parentLevel]` to $N-1$ (wide BVH).
            -   If popped node is tagged as "parent node" (Stack Culling extension): Set `level` to `parentLevel`.
            -   Else: Set `level` to `parentLevel` + 1.
    -   **Save Pop Level:** If a node's near child is marked as fully processed (bit set in trail for binary, or counter incremented for wide), save the level of that node into `popLevel` [S2, unofficial].

**Pseudocode (Binary BVH, combining [S2, unofficial] and [S1, unofficial] logic):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def restart_trail_traverse(ray, root, short_stack_size=5):
    # Initialization
    node = root
    trail_reg = 0  # All zeros
    level_reg = 1 << 31  # Sentinel bit set (assuming 32-bit register, MSB is sentinel)
    pop_level_reg = 0  # NONE
    short_stack = []
    
    # Helper to get level index from level_reg
    def get_level_index(reg):
        # Returns the index of the set bit. 
        # For MSB sentinel at bit 31, index 0 corresponds to bit 31.
        return 31 - (reg.bit_length() - 1)

    while True:
        # Traverse down
        while not node.is_leaf():
            # Intersect children
            hits = []
            for child in node.children:
                t = ray.intersect(child.bvh_node)
                if t is not None:
                    hits.append((t, child))
            
            hits.sort(key=lambda x: x[0]) # Sort by hit distance
            
            if len(hits) == 2:
                near_child, far_child = hits[0][1], hits[1][1]
                
                # Increment level
                level_reg >>= 1
                
                # Check trail
                if trail_reg & level_reg:
                    # Trail bit is 1: skip near, go to far
                    node = far_child
                else:
                    # Trail bit is 0: go to near, push far
                    node = near_child
                    if len(short_stack) < short_stack_size:
                        short_stack.append(far_child)
                    else:
                        # Stack full, handle overflow? 
                        # In strict restart trail, stack is short. 
                        # If full, we might need to pop or restart. 
                        # Standard implementation assumes short stack fits.
                        pass 
            elif len(hits) == 1:
                child = hits[0][1]
                level_reg >>= 1
                
                if level_reg != pop_level_reg:
                    # Set trail bit for this level
                    trail_reg |= level_reg
                    node = child
                else:
                    # Far child was culled, invoke POP
                    node, level_reg, pop_level_reg, short_stack, trail_reg = pop(trail_reg, level_reg, pop_level_reg, short_stack, node, get_level_index(level_reg))
            else:
                # No hits
                node, level_reg, pop_level_reg, short_stack, trail_reg = pop(trail_reg, level_reg, pop_level_reg, short_stack, node, get_level_index(level_reg))
        
        # Leaf processing
        t_leaf, primitive = ray.intersect_leaf(node)
        if t_leaf is not None:
            # Update closest hit
            if t_leaf < ray.t_max:
                ray.t_max = t_leaf
                # Record hit
        
        # Pop after leaf
        node, level_reg, pop_level_reg, short_stack, trail_reg = pop(trail_reg, level_reg, pop_level_reg, short_stack, node, get_level_index(level_reg))
        
        # Check termination
        if trail_reg & (1 << 31): # Sentinel bit set
            break

def pop(trail_reg, level_reg, pop_level_reg, short_stack, node, current_level_idx):
    # Find largest i <= current_level_idx where trail bit is 0
    # In binary trail, we look for the next zero bit above or at level_reg
    
    # Efficient bit manipulation for finding next zero bit
    # trailReg &= -levelReg; trailReg += levelReg;
    # This sets the lowest zero bit at or above levelReg to 1, and clears bits below.
    
    # However, we need to find the index first to update pop_level
    # Let's simulate the bit logic:
    
    # 1. Clear bits below current level
    # 2. Find next zero bit
    
    # Using the formula from [S2, unofficial]:
    # trailReg &= -levelReg
    # trailReg += levelReg
    
    # But we also need to track popLevel. 
    # popLevel is set when we traverse a node with 2 hits and set trail bit.
    
    # Let's implement the bit ops directly on integers
    # level_reg has one bit set.
    
    # Step 1: Find parent level (next zero bit)
    # We can iterate or use bit hacks. For clarity:
    temp = trail_reg & level_reg
    if temp == 0:
        # No zero bit at or above level_reg? 
        # This shouldn't happen if logic is correct, or means we are done?
        # Actually, we need to find the HIGHEST zero bit <= current level?
        # No, POP finds the largest i <= level where trail[i] == 0.
        
        # Let's use a loop for correctness in pseudocode
        i = current_level_idx
        while i >= 0:
            bit = 1 << (31 - i)
            if not (trail_reg & bit):
                parent_level_idx = i
                parent_level_reg = bit
                break
            i -= 1
        else:
            # No zero bit found, stack empty, traversal complete
            return None, 0, 0, [], 0 # Termination signal
            
    else:
        # There is a zero bit at or above level_reg?
        # The logic in [S2, unofficial] says: "Find the largest i such that i <= level and trail[i] = 0"
        # So we scan downwards from current level.
        parent_level_idx = current_level_idx
        while parent_level_idx >= 0:
            bit = 1 << (31 - parent_level_idx)
            if not (trail_reg & bit):
                break
            parent_level_idx -= 1
        else:
            # No zero bit found
            return None, 0, 0, [], 0
            
    parent_level_reg = 1 << (31 - parent_level_idx)
    
    # Update trail: set the bit at parent_level
    trail_reg |= parent_level_reg
    
    # Clear bits below parent_level
    # Create mask for bits below parent_level_reg
    # mask = parent_level_reg - 1
    # trail_reg &= ~mask
    # But we also need to clear bits above? No, only below.
    # Actually, the formula `trailReg &= -levelReg` clears bits below levelReg.
    # Here we want to clear bits below parent_level_reg.
    
    # Let's just clear bits below parent_level_reg
    mask = parent_level_reg - 1
    trail_reg &= ~mask
    
    # Update pop_level_reg
    pop_level_reg = parent_level_reg
    
    # Check sentinel
    if trail_reg & (1 << 31):
        return None, 0, 0, [], 0 # Termination
        
    # Pop from short stack
    if short_stack:
        next_node = short_stack.pop()
        # Determine new level
        # If next_node was a "last child" marker, we might need to adjust trail?
        # In binary BVH, the stack just holds nodes. The trail tracks progress.
        # The level is determined by the trail.
        # The new level is parent_level_idx + 1?
        # No, the level register points to the PARENT of the current node.
        # When we pop, we go up to parent_level. The next node is a child of that parent.
        # So the new level should point to the child of parent_level.
        
        # If we popped a node, it is a child of the node at parent_level_idx.
        # So the new level_reg should be parent_level_reg >> 1.
        
        new_level_reg = parent_level_reg >> 1
        return next_node, new_level_reg, pop_level_reg, short_stack, trail_reg
    else:
        # Stack empty, restart from root
        return root, (1 << 31), pop_level_reg, [], trail_reg
```

**Edge Cases and Pitfalls:**
-   **Traversal Order:** Must be front-to-back (closest first) for closest-hit searches. If a child is culled because a closer hit was found, it must not be counted as 'processed' in the restart trail counter, otherwise the restart will skip it incorrectly [S1, unofficial].
-   **Culled Far Child:** When a node has two intersecting children, but the far child becomes culled during the processing of the near child (due to ray shortening), the algorithm must detect this via `popLevel` to avoid re-entering the near branch. If `level` equals `popLevel` in the single-child branch, it indicates the far child was culled, and `POP()` must be invoked [S2, unofficial].
-   **Precision:** Care must be taken to ensure that equal $t$ values after a restart do not cause re-entering already processed branches, leading to infinite loops [S2, unofficial].

## Subtopics

### [Stack Culling (Parent Node Re-intersection)](9874-stack-culling-parent-node-re-intersection.md)
A technique that pushes the parent node onto the stack instead of multiple children when multiple children intersect and at least one is a leaf. The parent is re-intersected later to cull children with hit distances greater than the current closest hit. It sets itself apart from standard short-stack traversal by trading a small increase in traversal steps (1-7%) for a reduction in primitive intersections (2-14%), using only 1 bit of extra storage per stack entry instead of storing hit distances for every entry.

## Operating characteristics
-   **Time Complexity:** Traversal steps increase by 1-10% for closest-hit queries and 0-5% for any-hit queries compared to a full stack [S1, unofficial]. Stackless traversal (no short stack) visits approximately 2.2–2.4 times as many nodes as ordinary traversal with a full stack [S2, unofficial]. A short stack with just one entry reduces this multiplier to 1.3–1.4 [S2, unofficial].
-   **Memory Complexity:** Uses ~3x less memory than a full stack. For a short stack of 5 entries, storage is < 44 bytes (Root: 64 bits, Current Node: 32 bits, Restart Trail: 3x32 bits, Current Depth: 5 bits, Short Stack: 5x32 bits). A full stack for a max depth of 32 requires 136 bytes [S1, unofficial].
-   **Performance:** Preliminary tests on NVIDIA G80 and GT200 hardware indicate that memory bandwidth benefits of short stacks may not outweigh costs of restarts, SIMD divergence, and extra instructions [S2, unofficial]. However, it is well-suited for hardware hierarchy traversal units where registers are limited [S2, unofficial].
-   **Strengths:** Compact memory footprint, no need for parent pointers or hash lookups in BVH nodes, simpler implementation than stackless backtracking methods [S1, unofficial].
-   **Limitations:** Restart overhead increases with smaller stack sizes. Slightly higher extraneous node visits compared to kd-trees due to BVH child overlap causing more stack pushes [S2, unofficial].

## Background
Restart Trail was introduced by Tero Laine in 2010 [S2, unofficial]. It generalizes the restart trail algorithm for binary BVHs to BVHs of arbitrary widths, as proposed by Scheuermann et al. [S1, unofficial]. It is used in real-time ray tracing, particularly in hardware acceleration units and GPU ray cast kernels where memory bandwidth and on-chip storage are critical constraints [S1, unofficial][S2, unofficial].

## Sources
[S1] Wide BVH Traversal with a Short Stack, https://doi.org/10.2312/hpg.20191190, fetched 2026-10-24 (unofficial)
[S2] PDF Restart Trail for Stackless BVH Traversal - NVIDIA, https://research.nvidia.com/sites/default/files/pubs/2010-06_Restart-Trail-for/laine2010hpg_paper.pdf, fetched 2026-10-26 (unofficial)
