# Sparse Voxel Octree Ray Traversal (State Machine)
Also called: Fixed-Point SVO Traversal, SVO Raycasting

## Overview
A deterministic software ray traversal algorithm for Sparse Voxel Octrees (SVO) implemented as a finite state machine using fixed-point arithmetic, originally designed to emulate an FPGA hardware raycaster [S1, unofficial]. It differs from recursive or stack-based traversals by using explicit goto-based control flow and a circular buffer stack to manage backtracking, ensuring predictable execution paths suitable for hardware synthesis or deterministic software rendering [S1, unofficial].

## How it works
**Inputs:** Ray origin $r.o$, ray direction $r.d$, SVO data structure (compact nodes), traversal parameters.
**Outputs:** Hit point, depth, or traversal status.

**Data Structures:**
*   **CNode (Compact Node):** 32-bit packed structure.
    *   Bits 0-7: `valid_mask` (8 bits, child validity).
    *   Bits 8-15: `leaf_mask` (8 bits, child leaf status).
    *   Bits 16-30: `child_ptr` (15 bits, offset to children).
    *   Bit 31: `far_ptr` flag (1 bit, indicates if children are at a far address).
    *   *Note:* The source defines `UInt8` and `UInt16` fields within a union with `UInt32 far_ptr` [S1, unofficial].
*   **OctreeCursor:** Contains pointer to `CNode` data, current `CNode* node`, current address `adr`, current `level`, and a `Cache` pointer [S1, unofficial].
*   **StackItem:**
    *   `cur`: OctreeCursor.
    *   `tx0, ty0, tz0, tx1, ty1, tz1`: Fixed-point intersection times (tnum).
    *   `idx`: Current child index being processed.
*   **Stack:** Circular buffer of `StackItem` with `top` index, `length`, and size `ss`. Push wraps `top` modulo `ss` and increments `length` (capped at `ss`). Pop decrements `top` (wrapping if negative) and decrements `length` [S1, unofficial].

**Parameters:**
*   `terminLevel`: Maximum depth before forcing leaf termination. Default: 10 [S1, unofficial].
*   `tnum_scale`: Fixed-point scaling factor. Value: 65536 ($2^{16}$) [S1, unofficial].
*   `tnum_max`, `tnum_min`: Bounds for fixed-point clipping. Clamping offset: 1000 [S1, unofficial].

**Algorithm Steps:**

1.  **Ray Direction Normalization:**
    *   If any component of $r.d$ is zero, add 0.001 to that component to prevent division by zero [S1, unofficial].
    *   If any component of $r.d$ is negative, flip the ray origin component ($r.o.x \leftarrow -r.o.x$) and negate the direction component ($r.d.x \leftarrow -r.d.x$). Record these flips in `dir_mask` (bit 4 for X, bit 2 for Y, bit 1 for Z) [S1, unofficial].

2.  **Initial Bounding Box Intersection:**
    *   Compute intersection times with the unit cube $[-1, 1]^3$:
        $$tx0 = \frac{-1 - r.o.x}{r.d.x}, \quad tx1 = \frac{1 - r.o.x}{r.d.x}$$
        $$ty0 = \frac{-1 - r.o.y}{r.d.y}, \quad ty1 = \frac{1 - r.o.y}{r.d.y}$$
        $$tz0 = \frac{-1 - r.o.z}{r.d.z}, \quad tz1 = \frac{1 - r.o.z}{r.d.z}$$
    *   Calculate entry and exit times:
        $$t_{enter} = \max(tx0, \max(ty0, tz0))$$
        $$t_{exit} = \min(tx1, \min(ty1, tz1))$$
    *   If $t_{enter} \ge t_{exit}$, the ray misses the bounding box; terminate [S1, unofficial].

3.  **Fixed-Point Conversion:**
    *   Scale times by `tnum_scale`: $tnum = t \times 65536$.
    *   Clip values to $[tnum\_min + 1000, tnum\_max - 1000]$ to prevent overflow [S1, unofficial].

4.  **State Machine Loop:**
    The traversal uses explicit states (S_START, S_NEXT_IDX, S_EXIT, S_FINISH) [S1, unofficial].

    *   **S_START:**
        *   Initialize `cur` to root. `level = 0`.
        *   Compute node midpoints: $txm = (tx0+tx1)/2$, $tym = (ty0+ty1)/2$, $tzm = (tz0+tz1)/2$.
        *   Set $t_{enter} = \max(tx0, \max(ty0, tz0))$.
        *   Select initial child index: $idx = (txm < t_{enter}) \ll 2 \mid (tym < t_{enter}) \ll 1 \mid (tzm < t_{enter})$.
        *   Jump to S_NEXT_IDX.

    *   **S_NEXT_IDX:**
        *   Compute child bounds based on `idx` and current bounds ($tx0, tx1$, etc.):
            $$tx0_{child} = (idx \& 4) ? txm : tx0, \quad tx1_{child} = (idx \& 4) ? tx1 : txm$$
            (Similarly for Y and Z)
        *   Calculate child exit time: $t_{exit\_child} = \min(tx1_{child}, \min(ty1_{child}, tz1_{child}))$.
        *   Flip index for direction: $idx_{flip} = idx \oplus dir\_mask$.
        *   **Check Exit:** If $t_{exit\_child} \le t_{start}$ (current traversal start time), skip this node. Jump to S_NEXT_IDX with $idx \leftarrow idx + 1$. If $idx > 7$, jump to S_EXIT.
        *   **Check Leaf/Termination:** If `cur.childIsFilled(idx_flip)` OR `level == terminLevel`:
            *   Mark as leaf hit.
            *   Jump to S_FINISH.
        *   **Check Validity:** If `cur.childIsValid(idx_flip)`:
            *   Push current state (`cur`, bounds, `idx`) to Stack.
            *   Update `cur` to child: `cur = cur.getChild(idx_flip)`.
            *   Update bounds to child bounds.
            *   Increment `level`.
            *   Jump to S_NEXT_IDX.
        *   **Else:** Skip to next index. Jump to S_NEXT_IDX with $idx \leftarrow idx + 1$. If $idx > 7$, jump to S_EXIT.

    *   **S_EXIT:**
        *   If `level == 0`: Jump to S_FINISH.
        *   If Stack is empty: Restart traversal from $t_{exit\_child}$ (global restart). Jump to S_START.
        *   Pop Stack into `cur`, bounds, and `idx`.
        *   Decrement `level`.
        *   Jump to S_NEXT_IDX.

    *   **S_FINISH:**
        *   Return hit result.

**Pseudocode:**

```python
def traverse_svo_state_machine(ray, svo_data, termin_level=10, tnum_scale=65536):
    # 1. Handle Ray Direction
    dir_mask = 0
    r_o = ray.origin.copy()
    r_d = ray.direction.copy()
    
    for i, axis in enumerate(['x', 'y', 'z']):
        if r_d[axis] == 0:
            r_d[axis] = 0.001
        if r_d[axis] < 0:
            r_o[axis] = -r_o[axis]
            r_d[axis] = -r_d[axis]
            # Bit mapping: X=4, Y=2, Z=1
            if axis == 'x': dir_mask |= 4
            elif axis == 'y': dir_mask |= 2
            elif axis == 'z': dir_mask |= 1

    # 2. Initial Bounding Box Intersection
    tx0 = (-1.0 - r_o.x) / r_d.x
    tx1 = (1.0 - r_o.x) / r_d.x
    ty0 = (-1.0 - r_o.y) / r_d.y
    ty1 = (1.0 - r_o.y) / r_d.y
    tz0 = (-1.0 - r_o.z) / r_d.z
    tz1 = (1.0 - r_o.z) / r_d.z

    t_enter = max(tx0, max(ty0, tz0))
    t_exit = min(tx1, min(ty1, tz1))

    if t_enter >= t_exit:
        return None # Miss

    # 3. Fixed Point Conversion
    def to_tnum(t):
        val = t * tnum_scale
        # Clip to prevent overflow (values approximated from context)
        tnum_max = 2**31 - 1 # Assuming 32-bit signed int for tnum
        tnum_min = -2**31
        return max(tnum_min + 1000, min(tnum_max - 1000, int(val)))

    tx0 = to_tnum(tx0); tx1 = to_tnum(tx1)
    ty0 = to_tnum(ty0); ty1 = to_tnum(ty1)
    tz0 = to_tnum(tz0); tz1 = to_tnum(tz1)

    # Data Structures
    class StackItem:
        def __init__(self, cur, tx0, ty0, tz0, tx1, ty1, tz1, idx):
            self.cur = cur
            self.tx0 = tx0; self.ty0 = ty0; self.tz0 = tz0
            self.tx1 = tx1; self.ty1 = ty1; self.tz1 = tz1
            self.idx = idx

    stack = []
    stack_top = 0
    stack_len = 0
    stack_size = 256 # Example size, not given in sources [G]
    stack_store = [None] * stack_size

    def push_stack(item):
        nonlocal stack_top, stack_len
        stack_store[stack_top] = item
        stack_top = (stack_top + 1) % stack_size
        if stack_len < stack_size:
            stack_len += 1

    def pop_stack():
        nonlocal stack_top, stack_len
        if stack_len == 0:
            return None
        stack_len -= 1
        stack_top = (stack_top - 1) % stack_size
        if stack_top < 0: stack_top += stack_size
        return stack_store[stack_top]

    # Cursor Helper (Simplified)
    def get_child_cursor(cur, child_idx):
        # Logic from OctreeCursor::getChild
        # Assumes cur.node is current CNode
        # Counts valid children to find actual index
        valid_count = 0
        for i in range(child_idx):
            if cur.node.valid_mask & (1 << i):
                valid_count += 1
        
        # Get child pointer
        child_ptr = cur.node.child_ptr & ~(1 << 15)
        if cur.node.child_ptr & (1 << 15):
            far_adr = cur.adr + child_ptr
            # Cache request logic omitted for brevity [G]
            child_node_addr = far_adr
        else:
            child_node_addr = cur.adr + child_ptr # Relative offset logic [G]
            
        # Create new cursor
        new_cur = OctreeCursor()
        new_cur.data = cur.data
        new_cur.dataSize = cur.dataSize
        new_cur.cache = cur.cache
        new_cur.level = cur.level + 1
        new_cur.adr = child_node_addr
        new_cur.node = cur.data[child_node_addr] # Assuming array access
        return new_cur

    class OctreeCursor:
        def __init__(self):
            self.data = None
            self.dataSize = 0
            self.node = None
            self.level = 0
            self.adr = 0
            self.cache = None

    # Initialize Root
    cur = OctreeCursor()
    cur.data = svo_data.nodes
    cur.dataSize = len(svo_data.nodes)
    cur.node = svo_data.root
    cur.level = 0
    cur.adr = 0
    cur.cache = svo_data.cache

    t_start = 0 # Global start time for restart

    while True:
        # S_START / S_NEXT_IDX logic
        # Compute midpoints
        txm = (tx0 + tx1) // 2
        tym = (ty0 + ty1) // 2
        tzm = (tz0 + tz1) // 2
        
        # Re-calculate t_enter for current bounds
        t_enter_curr = max(tx0, max(ty0, tz0))
        
        # If first node, idx is determined by midpoints vs t_enter
        # Otherwise, idx is passed in from stack or incremented
        # This state machine logic is complex to map to pure Python without goto
        # We simulate the loop structure
        
        # For the first iteration, idx is calculated
        if not 'idx' in locals() or 'idx' not in dir():
            idx = ((txm < t_enter_curr) << 2) | ((tym < t_enter_curr) << 1) | (tzm < t_enter_curr)
        else:
            idx = idx # From previous iteration or stack

        while idx < 8:
            # Compute child bounds
            # idx bit 2 (4) is X, bit 1 (2) is Y, bit 0 (1) is Z
            if idx & 4:
                c_tx0, c_tx1 = txm, tx1
            else:
                c_tx0, c_tx1 = tx0, txm
            
            if idx & 2:
                c_ty0, c_ty1 = tym, ty1
            else:
                c_ty0, c_ty1 = ty0, ty1
                
            if idx & 1:
                c_tz0, c_tz1 = tzm, tz1
            else:
                c_tz0, c_tz1 = tz0, tz1

            t_exit_child = min(c_tx1, min(c_ty1, c_tz1))
            
            idx_flip = idx ^ dir_mask
            
            # Check if child is behind start
            if t_exit_child <= t_start:
                idx += 1
                continue
            
            # Check leaf/termination
            # Note: childIsFilled and childIsValid depend on specific SVO layout
            # Assuming standard bitmasks
            is_filled = (cur.node.leaf_mask >> idx_flip) & 1
            is_valid = (cur.node.valid_mask >> idx_flip) & 1
            
            if is_filled or cur.level == termin_level:
                return {"hit": True, "depth": t_exit_child, "node": cur.node}
            
            if is_valid:
                # Push state
                item = StackItem(cur, tx0, ty0, tz0, tx1, ty1, tz1, idx)
                push_stack(item)
                
                # Move to child
                cur = get_child_cursor(cur, idx_flip)
                tx0, ty0, tz0 = c_tx0, c_ty0, c_tz0
                tx1, ty1, tz1 = c_tx1, c_ty1, c_tz1
                idx = 0 # Reset idx for new node
                break # Restart loop for new node
            else:
                idx += 1
        else:
            # S_EXIT: All children of current node processed
            if cur.level == 0:
                return None # Done
            
            if stack_len == 0:
                # Restart from t_exit_child
                t_start = t_exit_child
                # Reset to root
                cur = OctreeCursor()
                cur.data = svo_data.nodes
                cur.dataSize = len(svo_data.nodes)
                cur.node = svo_data.root
                cur.level = 0
                cur.adr = 0
                cur.cache = svo_data.cache
                tx0, ty0, tz0 = to_tnum(-1.0 - r_o.x)/r_d.x, ... # Recompute initial bounds? 
                # Actually, restart implies re-evaluating from t_exit_child
                # Simplified: just reset and let loop handle t_start check
                # But bounds need to be reset to root bounds
                tx0 = to_tnum((-1.0 - r_o.x)/r_d.x)
                tx1 = to_tnum((1.0 - r_o.x)/r_d.x)
                ty0 = to_tnum((-1.0 - r_o.y)/r_d.y)
                ty1 = to_tnum((1.0 - r_o.y)/r_d.y)
                tz0 = to_tnum((-1.0 - r_o.z)/r_d.z)
                tz1 = to_tnum((1.0 - r_o.z)/r_d.z)
                idx = 0
                continue
            
            popped = pop_stack()
            cur = popped.cur
            tx0, ty0, tz0 = popped.tx0, popped.ty0, popped.tz0
            tx1, ty1, tz1 = popped.tx1, popped.ty1, popped.tz1
            idx = popped.idx + 1
            cur.level -= 1

## Operating characteristics
*   **Time Complexity:** Proportional to the number of nodes visited. The state machine overhead is minimal, making it efficient for hardware implementation.
*   **Memory Complexity:** Uses a fixed-size circular buffer stack. Memory usage is $O(S)$ where $S$ is the stack size, independent of tree depth.
*   **Strengths:** Deterministic execution, low memory footprint, suitable for FPGA and embedded systems. Fixed-point arithmetic avoids floating-point unit dependencies.
*   **Limitations:** Fixed-point precision limits accuracy for very deep trees or large scenes. Requires careful handling of edge cases (zero direction, overflow).
*   **When to use:** Real-time rendering on constrained hardware, FPGA-based ray tracing, or when deterministic software behavior is required.
*   **When not to use:** High-precision offline rendering where floating-point accuracy is critical, or scenes with extremely deep octrees exceeding stack limits.

## Background
The method originates from the `svo-workbench` project by skyfex, which implements a software ray tracer for Sparse Voxel Octrees [S1, unofficial]. It was specifically designed to model a Verilog-based hardware raycaster, allowing the same traversal logic to be used in both software and FPGA implementations [S1, unofficial]. The project demonstrates caching strategies and rendering into a shared framebuffer [S1, unofficial].

## Sources
[S1] skyfex/svo-workbench, https://github.com/skyfex/svo-workbench, fetched 2026-10-26 (unofficial)
