# Stack-based Octree Traversal with Direction Flipping

Also called: Octree Ray Traversal with Direction Flipping, Stack-based Octree Traversal, TreeTraverser

## Overview
A stack-based algorithm for traversing an octree structure along a ray, handling arbitrary ray directions by flipping the coordinate system and ray parameters to ensure positive traversal direction [S1, unofficial]. It differs from [Ray Traversal of Octree Nodes](1841-ray-traversal-of-octree-nodes.md) by using an explicit stack to manage non-linear traversal paths and direction flipping to simplify child selection logic, and from [IEEE-754 Bit Manipulation Octree Traversal](3907-ieee-754-bit-manipulation-octree-traversal.md) by relying on floating-point comparisons and explicit stack management rather than bit-level integer arithmetic for step determination [G].

## How it works
### Inputs and Outputs
**Inputs:**
*   `Octree`: An octree structure with fields `min`, `max`, `size`, `gridlength`, and a vector of `Node` objects [S1, unofficial]. Each `Node` has methods `hasChild(i)`, `isLeaf()`, and `isNull()` [S1, unofficial].
*   `Ray`: A ray defined by `origin` (vec3) and `direction` (vec3) [S1, unofficial].

**Outputs:**
*   A sequence of visited nodes in traversal order, accessible via the stack state [S1, unofficial].
*   The current node and its associated t-intervals during traversal [S1, unofficial].

### Data Structures
**TraversalNodeInfo_**
A struct representing a node on the traversal stack [S1, unofficial].
*   `const Node* node`: Pointer to the current octree node [S1, unofficial].
*   `vec3 t0`: Entry t-values for the current node's bounding box (x, y, z components) [S1, unofficial].
*   `vec3 t1`: Exit t-values for the current node's bounding box (x, y, z components) [S1, unofficial].
*   `vec3 tm`: Midpoint t-values for the current node's bounding box (x, y, z components) [S1, unofficial].
*   `int nextchild`: Index of the next child to visit [S1, unofficial].
    *   `-1`: First visit to this node; children not yet determined [S1, unofficial].
    *   `0-7`: Index of the next child octant to visit [S1, unofficial].
    *   `8`: All children visited; node is done [S1, unofficial].

**TreeTraverser**
The main traversal class [S1, unofficial].
*   `unsigned char a`: Direction flip mask [S1, unofficial]. Bits 0, 1, 2 correspond to Z, Y, X axes respectively [S1, unofficial]. If bit $i$ is set, the corresponding axis was flipped [S1, unofficial].
*   `Octree const* octree`: Pointer to the octree being traversed [S1, unofficial].
*   `Ray ray`: The modified ray with flipped origin/direction if necessary [S1, unofficial].
*   `Ray original_ray`: The original ray before any flips, used for final position calculation [S1, unofficial].
*   `std::vector<TraversalNodeInfo_> stack`: The stack of nodes to visit [S1, unofficial].
*   `int stepcount`: Counter for traversal steps [S1, unofficial].

### Numbered Steps

#### 1. Initialization (`initTraversal`)
1.  Set `stepcount = 0`, `a = 0` [S1, unofficial].
2.  Store `original_ray` [S1, unofficial].
3.  For each axis $i \in \{0, 1, 2\}$ (corresponding to X, Y, Z):
    *   Check ray direction sign. The source code uses specific logic: if `ray.direction[i] < 0` for X and Y, or `ray.direction[i] > 0` for Z (note: this Z check is non-standard and specific to the source implementation) [S1, unofficial].
    *   If the condition is met:
        *   Flip `ray.origin[i] = octree->size[i] - ray.origin[i]` [S1, unofficial].
        *   Flip `ray.direction[i] = -ray.direction[i]` [S1, unofficial].
        *   Set the corresponding bit in `a`: bit 0 for Z, bit 1 for Y, bit 2 for X [S1, unofficial].
4.  Calculate entry/exit t-values for the octree bounding box using the modified ray:
    *   $tx_0 = (octree->min[0] - ray.origin[0]) \times (1.0f / ray.direction[0])$ [S1, unofficial].
    *   $tx_1 = (octree->max[0] - ray.origin[0]) \times (1.0f / ray.direction[0])$ [S1, unofficial].
    *   Similarly for $ty_0, ty_1$ (Y axis) and $tz_0, tz_1$ (Z axis) [S1, unofficial].
5.  Check if the ray intersects the octree:
    *   $t_{enter} = \max(\max(tx_0, ty_0), tz_0)$ [S1, unofficial].
    *   $t_{exit} = \min(\min(tx_1, ty_1), tz_1)$ [S1, unofficial].
    *   If $t_{enter} < t_{exit}$:
        *   Create `TraversalNodeInfo_` for the root node [S1, unofficial].
        *   Set `t0 = (tx_0, ty_0, tz_0)`, `t1 = (tx_1, ty_1, tz_1)`, `tm = 0.5f * (t0 + t1)` [S1, unofficial].
        *   Set `nextchild = -1` [S1, unofficial].
        *   Push onto `stack` [S1, unofficial].
    *   Else: Stack remains empty; no traversal occurs [S1, unofficial].

#### 2. Step Function (`step`)
Repeatedly call until the stack is empty or traversal is complete [S1, unofficial].
1.  **POP Check**:
    *   If `stack` is empty, return [S1, unofficial].
    *   Let `top = stack.back()` [S1, unofficial].
    *   If `top.nextchild == 8` (all children visited) or `top.node->isLeaf()`, pop `stack` and return [S1, unofficial].
2.  **PUSH (First Visit)**:
    *   If `top.nextchild == -1`:
        *   Calculate `tm = 0.5f * (top.t0 + top.t1)` [S1, unofficial].
        *   Determine the first child to visit using `firstNode(top.t0, tm)` [S1, unofficial].
        *   Set `top.nextchild` to the result of `firstNode` [S1, unofficial].
        *   Update `stack.back()` with the new `nextchild` [S1, unofficial].
3.  **ADVANCE (Visit Child)**:
    *   Let `current_case = top.nextchild` [S1, unofficial].
    *   Calculate `child_index = current_case ^ a` [S1, unofficial]. This maps the logical child index to the physical child index based on the direction flip mask [S1, unofficial].
    *   Check if the child exists: `if (top.node->hasChild(child_index))` [S1, unofficial].
    *   If it exists:
        *   Calculate the new `t0` and `t1` bounds for the child voxel using standard interval subdivision logic [G].
        *   Create `TraversalNodeInfo_` for the child with updated `t0`, `t1`, `tm = 0.5f * (t0 + t1)`, and `nextchild = -1` [S1, unofficial].
        *   Push the new `TraversalNodeInfo_` onto `stack` [S1, unofficial].
        *   Set `top.nextchild` to `current_case + 1` to visit the next child in sequence [S1, unofficial].

#### 3. First Child Selection (`firstNode`)
Given `t0` and `tm` for the current node:
1.  If $tx_0 > ty_0$ and $tx_0 > tz_0$ (entering via YZ plane, i.e., X-axis first):
    *   If $ty_m < tx_0$, set bit 1 [S1, unofficial].
    *   If $tz_m < tx_0$, set bit 0 [S1, unofficial].
    *   Result is the combined bit value [S1, unofficial].
2.  Else if $ty_0 > tz_0$ (entering via XZ plane, i.e., Y-axis first):
    *   If $tx_m < ty_0$, set bit 2 [S1, unofficial].
    *   If $tz_m < ty_0$, set bit 0 [S1, unofficial].
    *   Result is the combined bit value [S1, unofficial].
3.  Else (entering via XY plane, i.e., Z-axis first):
    *   If $tx_m < tz_0$, set bit 2 [S1, unofficial].
    *   If $ty_m < tz_0$, set bit 1 [S1, unofficial].
    *   Result is the combined bit value [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Ray:
    def __init__(self, origin, direction):
        self.origin = list(origin)  # [x, y, z]
        self.direction = list(direction)  # [x, y, z]
    def get_ray_point(self, t):
        return [self.origin[i] + t * self.direction[i] for i in range(3)]

class TraversalNodeInfo:
    def __init__(self, node, t0, t1, tm, nextchild):
        self.node = node
        self.t0 = list(t0)
        self.t1 = list(t1)
        self.tm = list(tm)
        self.nextchild = nextchild

class TreeTraverser:
    def __init__(self, octree, ray):
        self.octree = octree
        self.original_ray = Ray(list(ray.origin), list(ray.direction))
        self.ray = Ray(list(ray.origin), list(ray.direction))
        self.a = 0
        self.stack = []
        self.stepcount = 0

    def init_traversal(self):
        self.stepcount = 0
        self.a = 0
        for i in range(3):
            dir_val = self.ray.direction[i]
            should_flip = False
            if i < 2:  # X or Y
                if dir_val < 0: should_flip = True
            else:  # Z
                if dir_val > 0: should_flip = True
            if should_flip:
                self.ray.origin[i] = self.octree.size[i] - self.ray.origin[i]
                self.ray.direction[i] = -self.ray.direction[i]
                if i == 2: self.a |= 1
                elif i == 1: self.a |= 2
                elif i == 0: self.a |= 4
        inv_dir = [1.0 / self.ray.direction[i] for i in range(3)]
        tx0 = (self.octree.min[0] - self.ray.origin[0]) * inv_dir[0]
        tx1 = (self.octree.max[0] - self.ray.origin[0]) * inv_dir[0]
        ty0 = (self.octree.min[1] - self.ray.origin[1]) * inv_dir[1]
        ty1 = (self.octree.max[1] - self.ray.origin[1]) * inv_dir[1]
        tz0 = (self.octree.min[2] - self.ray.origin[2]) * inv_dir[2]
        tz1 = (self.octree.max[2] - self.ray.origin[2]) * inv_dir[2]
        t0 = [tx0, ty0, tz0]
        t1 = [tx1, ty1, tz1]
        t_enter = max(max(t0[0], t0[1]), t0[2])
        t_exit = min(min(t1[0], t1[1]), t1[2])
        if t_enter < t_exit:
            tm = [0.5 * (t0[i] + t1[i]) for i in range(3)]
            root_node = self.octree.get_root_node()
            info = TraversalNodeInfo(root_node, t0, t1, tm, -1)
            self.stack.append(info)
        else:
            self.stack = []

    def first_node(self, t0, tm):
        tx0, ty0, tz0 = t0
        txm, tym, tzm = tm
        child = 0
        if tx0 > ty0 and tx0 > tz0:
            if tym < tx0: child |= 2
            if tzm < tx0: child |= 1
        elif ty0 > tz0:
            if txm < ty0: child |= 4
            if tzm < ty0: child |= 1
        else:
            if txm < tz0: child |= 4
            if tym < tz0: child |= 2
        return child

    def step(self):
        if not self.stack: return False
        top = self.stack[-1]
        if top.nextchild == 8 or top.node.is_leaf():
            self.stack.pop()
            return True
        if top.nextchild == -1:
            tm = [0.5 * (top.t0[i] + top.t1[i]) for i in range(3)]
            first_child = self.first_node(top.t0, tm)
            top.nextchild = first_child
            self.stack[-1] = top
        current_case = top.nextchild
        child_index = current_case ^ self.a
        if top.node.has_child(child_index):
            new_t0 = list(top.t0)
            new_t1 = list(top.t1)
            for i in range(3):
                bit = (child_index >> i) & 1
                if bit == 0: new_t0[i] = top.tm[i]
                else: new_t1[i] = top.tm[i]
            new_tm = [0.5 * (new_t0[i] + new_t1[i]) for i in range(3)]
            child_node = top.node.get_child(child_index)
            new_info = TraversalNodeInfo(child_node, new_t0, new_t1, new_tm, -1)
            self.stack.append(new_info)
            top.nextchild = current_case + 1
            self.stack[-2] = top
        else:
            top.nextchild = current_case + 1
            self.stack[-1] = top
        return True

    def get_current_position(self):
        if not self.stack: return None
        top = self.stack[-1]
        t = max(top.t0)
        pos = self.original_ray.get_ray_point(t)
        pos[2] = -pos[2]
        return pos
```

### Parameters
*   `a`: Direction flip mask, computed during initialization [S1, unofficial].
*   `stepcount`: Traversal step counter [S1, unofficial].

### Edge Cases and Pitfalls
*   **Ray Direction Handling**: The code assumes specific sign checks for each axis (negative for X and Y, positive for Z) to determine flipping. This is non-standard and must be replicated exactly [S1, unofficial].
*   **Position Calculation**: `getCurrentPosition` uses `original_ray.getRayPoint(t)` where `t = stack.back().t0.max()`, and then negates the z-coordinate: `answer[2] = -answer[2]`. This compensates for the z-flip during traversal [S1, unofficial].
*   **Division by Zero**: If a ray direction component is zero, the t-calculation will result in division by zero. The source code does not explicitly handle this, so it is assumed that rays with zero direction components are either pre-filtered or handled elsewhere [G].
*   **Child Order**: The exact logic for determining the "next" child to visit after the current one is not fully specified in the source. The implementation uses a simple increment, which matches the sequential visitation implied by the stack-based design [G].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of nodes visited along the ray path. Each node is visited once [G].
*   **Memory Complexity**: $O(D)$ where $D$ is the depth of the octree, due to the stack [G].
*   **Measured Performance**: Not given in the sources.
*   **Scaling**: Scales with the depth of the octree and the number of nodes intersected by the ray [G].
*   **Strengths**: Handles arbitrary ray directions efficiently by flipping coordinates. Simple stack-based implementation [S1, unofficial].
*   **Limitations**: Non-standard direction flipping logic for Z-axis. Child order logic is not fully specified [S1, unofficial].
*   **When to Use**: When traversing an octree with arbitrary ray directions and a simple stack-based approach is preferred [G].
*   **When Not to Use**: When bit-manipulation based traversal is preferred for performance, or when the non-standard Z-flip logic is a concern [G].

## Background
The method is derived from an unofficial CPU voxel raycaster implementation [S1, unofficial]. The specific direction flipping logic for the Z-axis (positive direction check) is unique to this implementation and not standard in ray tracing literature [S1, unofficial].

## Sources
[S1] Forceflow/cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
