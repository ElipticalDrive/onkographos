# Stack-based Octree Traversal with Direction Flipping

Also called: Octree Ray Traversal with Direction Flipping, Stack-based Octree Traversal, TreeTraverser

## Overview
A stack-based algorithm for traversing an octree structure along a ray, handling arbitrary ray directions by flipping the coordinate system and ray parameters to ensure positive traversal direction [S1, unofficial]. It differs from [Ray Traversal of Octree Nodes](1841-ray-traversal-of-octree-nodes.md) by using an explicit stack to manage backtracking and state, rather than relying solely on mid-plane intersection logic for control flow, and from [Ray-Octree Traversal (Gargantini and Atkinson variant)](1890-ray-octree-traversal-gargantini-and-atkinson-variant.md) by normalizing ray direction to simplify child index calculation via XOR masking [S1, unofficial].

## How it works
**Inputs:** An octree structure (`Octree`), a ray (`Ray`) defined by origin and direction, and the octree bounding box (`min`, `max`) [S1, unofficial].
**Outputs:** A sequence of visited octree nodes (leaf or internal) intersected by the ray, or a termination signal when the ray exits the octree or all children are visited [S1, unofficial].

**Data Structures:**
1.  `Node`: Represents an octree node.
    *   `hasChild(unsigned int i) const`: Returns true if child `i` exists.
    *   `isLeaf() const`: Returns true if the node has no children.
    *   `children_offset[8]`: Array storing offsets to children. `NOCHILD` indicates absence.
    *   `children_base`: Base index for child data.
2.  `TraversalNodeInfo_`: Stack element.
    *   `const Node* node`: Pointer to the current octree node.
    *   `vec3 t0`: Entry t-values for the current voxel bounds (x, y, z components).
    *   `vec3 t1`: Exit t-values for the current voxel bounds (x, y, z components).
    *   `vec3 tm`: Midpoint t-values for the current voxel bounds (x, y, z components).
    *   `int nextchild`: Index of the next child to visit. `-1` indicates first visit (unvisited). `0-7` indicates specific child index. `8` indicates all children visited [S1, unofficial].
3.  `TreeTraverser`: Main traversal class.
    *   `unsigned char a`: Direction flip mask (3 bits, bit 0 for Z, bit 1 for Y, bit 2 for X) [S1, unofficial].
    *   `Octree const* octree`: Pointer to the octree.
    *   `Ray ray`: Modified ray with flipped direction/origin.
    *   `Ray original_ray`: Original ray for position calculation.
    *   `std::vector<TraversalNodeInfo_> stack`: Stack for traversal state.
    *   `int stepcount`: Counter for steps.

**Initialization (`initTraversal`):**
1.  Set `stepcount = 0`, `a = 0` [S1, unofficial].
2.  Store `original_ray` [S1, unofficial].
3.  For each axis `i` (0=X, 1=Y, 2=Z):
    *   If `ray.direction[i] < 0` for `i=0,1` (X, Y) OR `ray.direction[i] > 0` for `i=2` (Z) [G: Specific sign check logic from source]:
        *   Flip `ray.origin[i] = octree->size[i] - ray.origin[i]` [S1, unofficial].
        *   Flip `ray.direction[i] = -ray.direction[i]` [S1, unofficial].
        *   Set bit `i` in `a` (bit 0 for Z, bit 1 for Y, bit 2 for X) [S1, unofficial].
4.  Calculate entry/exit t-values for the octree bounding box:
    *   `tx0 = (octree->min[0] - ray.origin[0]) * (1.0f / ray.direction[0])` [S1, unofficial].
    *   `tx1 = (octree->max[0] - ray.origin[0]) * (1.0f / ray.direction[0])` [S1, unofficial].
    *   `ty0 = (octree->min[1] - ray.origin[1]) * (1.0f / ray.direction[1])` [S1, unofficial].
    *   `ty1 = (octree->max[1] - ray.origin[1]) * (1.0f / ray.direction[1])` [S1, unofficial].
    *   `tz0 = (octree->min[2] - ray.origin[2]) * (1.0f / ray.direction[2])` [S1, unofficial].
    *   `tz1 = (octree->max[2] - ray.origin[2]) * (1.0f / ray.direction[2])` [S1, unofficial].
5.  Check intersection: If `max(max(tx0, ty0), tz0) < min(min(tx1, ty1), tz1)`:
    *   Create `TraversalNodeInfo_` for root:
        *   `node = octree->getRootNode()`
        *   `t0 = vec3(tx0, ty0, tz0)`
        *   `t1 = vec3(tx1, ty1, tz1)`
        *   `tm = vec3(0.5f * (tx0 + tx1), 0.5f * (ty0 + ty1), 0.5f * (tz0 + tz1))` [S1, unofficial].
        *   `nextchild = -1`
    *   Push onto stack.
    *   Else, stack remains empty.

**Step Function:**
1.  **POP:** If stack is empty, return. If `stack.back().nextchild == 8` (all children visited) or `stack.back().node->isLeaf()`, pop stack and return [S1, unofficial].
2.  **PUSH (First Visit):** If `stack.back().nextchild == -1`:
    *   Calculate `tm` (already computed in init for root, or derived from `t0` and `t1` for children) [S1, unofficial].
    *   Determine `first_child` using `firstNode(t0, tm)` [S1, unofficial].
    *   Set `stack.back().nextchild = first_child` [S1, unofficial].
3.  **ADVANCE:** Switch on `stack.back().nextchild` (0-7):
    *   Calculate `child_index = current_case ^ a` [S1, unofficial].
    *   Check if `stack.back().node->hasChild(child_index)` [S1, unofficial].
    *   If exists:
        *   Calculate new `t0_child`, `t1_child` bounds for the child voxel based on the child's spatial extent relative to `t0`, `t1`, `tm` [S1, unofficial].
        *   Create new `TraversalNodeInfo_`:
            *   `node = stack.back().node->getChild(child_index)` [S1, unofficial].
            *   `t0 = new_t0`
            *   `t1 = new_t1`
            *   `tm = 0.5f * (t0 + t1)`
            *   `nextchild = -1`
        *   Push new node onto stack.
        *   Set `stack.back().nextchild` to the next child to visit after this one [S1, unofficial].
    *   If not exists, continue to next case (or handle empty child) [S1, unofficial].

**First Child Selection (`firstNode`):**
Given `t0` and `tm`:
1.  If `tx0 > ty0` and `tx0 > tz0` (entering via YZ plane, i.e., X is the last axis to exit):
    *   Check if `tym < tx0` (set bit 1 for Y) [S1, unofficial].
    *   Check if `tzm < tx0` (set bit 0 for Z) [S1, unofficial].
    *   Result: `bit1 | bit0`.
2.  Else if `ty0 > tz0` (entering via XZ plane, i.e., Y is the last axis to exit):
    *   Check if `txm < ty0` (set bit 2 for X) [S1, unofficial].
    *   Check if `tzm < ty0` (set bit 0 for Z) [S1, unofficial].
    *   Result: `bit2 | bit0`.
3.  Else (entering via XY plane, i.e., Z is the last axis to exit):
    *   Check if `txm < tz0` (set bit 2 for X) [S1, unofficial].
    *   Check if `tym < tz0` (set bit 1 for Y) [S1, unofficial].
    *   Result: `bit2 | bit1`.

**Position Calculation (`getCurrentPosition`):**
1.  `t = stack.back().t0.max()` (maximum of x, y, z components of t0) [S1, unofficial].
2.  `answer = original_ray.getRayPoint(t)` [S1, unofficial].
3.  `answer[2] = -answer[2]` (negate Z to compensate for Z-flip) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vec3:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z
    def __getitem__(self, i):
        return (self.x, self.y, self.z)[i]
    def __setitem__(self, i, v):
        vals = [self.x, self.y, self.z]
        vals[i] = v
        self.x, self.y, self.z = vals
    def max(self):
        return max(self.x, self.y, self.z)

class Ray:
    def __init__(self, origin, direction):
        self.origin = origin  # Vec3
        self.direction = direction  # Vec3
    def get_point(self, t):
        return Vec3(self.origin.x + t * self.direction.x,
                    self.origin.y + t * self.direction.y,
                    self.origin.z + t * self.direction.z)

class Node:
    def __init__(self, is_leaf, has_children_func, get_child_func):
        self._is_leaf = is_leaf
        self._has_children_func = has_children_func
        self._get_child_func = get_child_func
    def is_leaf(self):
        return self._is_leaf
    def has_child(self, index):
        return self._has_children_func(index)
    def get_child(self, index):
        return self._get_child_func(index)

class TraversalNodeInfo:
    def __init__(self, node, t0, t1, tm, nextchild):
        self.node = node
        self.t0 = t0  # Vec3
        self.t1 = t1  # Vec3
        self.tm = tm  # Vec3
        self.nextchild = nextchild  # int

class TreeTraverser:
    def __init__(self, octree, ray):
        self.octree = octree
        self.original_ray = ray
        self.ray = Ray(Vec3(*ray.origin), Vec3(*ray.direction))
        self.a = 0
        self.stack = []
        self.stepcount = 0
        self.init_traversal()

    def init_traversal(self):
        self.stepcount = 0
        self.a = 0
        # Flip ray direction and origin based on specific sign checks [S1, unofficial]
        for i in range(3):
            if (i < 2 and self.ray.direction[i] < 0) or (i == 2 and self.ray.direction[i] > 0):
                self.ray.origin[i] = self.octree.size[i] - self.ray.origin[i]
                self.ray.direction[i] = -self.ray.direction[i]
                self.a |= (1 << i)

        # Calculate t0, t1 for bounding box [S1, unofficial]
        tx0 = (self.octree.min[0] - self.ray.origin[0]) / self.ray.direction[0]
        tx1 = (self.octree.max[0] - self.ray.origin[0]) / self.ray.direction[0]
        ty0 = (self.octree.min[1] - self.ray.origin[1]) / self.ray.direction[1]
        ty1 = (self.octree.max[1] - self.ray.origin[1]) / self.ray.direction[1]
        tz0 = (self.octree.min[2] - self.ray.origin[2]) / self.ray.direction[2]
        tz1 = (self.octree.max[2] - self.ray.origin[2]) / self.ray.direction[2]

        # Check intersection [S1, unofficial]
        t_enter = max(tx0, ty0, tz0)
        t_exit = min(tx1, ty1, tz1)

        if t_enter < t_exit:
            t0 = Vec3(tx0, ty0, tz0)
            t1 = Vec3(tx1, ty1, tz1)
            tm = Vec3(0.5 * (tx0 + tx1), 0.5 * (ty0 + ty1), 0.5 * (tz0 + tz1))
            root = self.octree.getRootNode()
            self.stack.append(TraversalNodeInfo(root, t0, t1, tm, -1))
        else:
            self.stack = []

    def step(self):
        if not self.stack:
            return False

        top = self.stack[-1]

        # POP condition [S1, unofficial]
        if top.nextchild == 8 or top.node.is_leaf():
            self.stack.pop()
            return False

        # PUSH condition (First visit) [S1, unofficial]
        if top.nextchild == -1:
            tm = top.tm
            first_child = self.first_node(top.t0, tm)
            top.nextchild = first_child

        # ADVANCE condition
        current_case = top.nextchild
        child_index = current_case ^ self.a  # Map logical to physical index [S1, unofficial]

        if top.node.has_child(child_index):
            # Calculate child bounds based on spatial extent [S1, unofficial]
            # Bit 0=Z, Bit 1=Y, Bit 2=X
            child_t0 = Vec3(
                top.t0.x if (child_index & 4) == 0 else top.tm.x,
                top.t0.y if (child_index & 2) == 0 else top.tm.y,
                top.t0.z if (child_index & 1) == 0 else top.tm.z
            )
            child_t1 = Vec3(
                top.tm.x if (child_index & 4) == 0 else top.t1.x,
                top.tm.y if (child_index & 2) == 0 else top.t1.y,
                top.tm.z if (child_index & 1) == 0 else top.t1.z
            )
            child_tm = Vec3(
                0.5 * (child_t0.x + child_t1.x),
                0.5 * (child_t0.y + child_t1.y),
                0.5 * (child_t0.z + child_t1.z)
            )

            child_node = top.node.get_child(child_index)
            self.stack.append(TraversalNodeInfo(child_node, child_t0, child_t1, child_tm, -1))

            # Update parent's nextchild to next sibling [G: Standard traversal logic]
            top.nextchild = current_case + 1
        else:
            # Child does not exist, move to next sibling
            top.nextchild = current_case + 1

        return True

    def first_node(self, t0, tm):
        tx0, ty0, tz0 = t0.x, t0.y, t0.z
        txm, tym, tzm = tm.x, tm.y, tm.z

        if tx0 > ty0 and tx0 > tz0:
            # Entering via YZ plane [S1, unofficial]
            bit1 = 1 if tym < tx0 else 0
            bit0 = 1 if tzm < tx0 else 0
            return bit1 | bit0
        elif ty0 > tz0:
            # Entering via XZ plane [S1, unofficial]
            bit2 = 1 if txm < ty0 else 0
            bit0 = 1 if tzm < ty0 else 0
            return bit2 | bit0
        else:
            # Entering via XY plane [S1, unofficial]
            bit2 = 1 if txm < tz0 else 0
            bit1 = 1 if tym < tz0 else 0
            return bit2 | bit1

    def get_current_position(self):
        if not self.stack:
            return None
        top = self.stack[-1]
        t = top.t0.max()
        pos = self.original_ray.get_point(t)
        pos.z = -pos.z  # Compensate for Z-flip [S1, unofficial]
        return pos
```

## Subtopics

### [Ray-Octree Traversal with Stack-Based Backtracking](13237-ray-octree-traversal-with-stack-based-backtracking.md)
A ray casting algorithm that traverses an octree structure to locate leaf voxels intersected by a ray. It uses a stack to store parent context for backtracking when a ray exits a voxel or skips empty space, allowing it to resume traversal at a higher level. It builds on [Stack-based Octree Traversal with Direction Flipping](5007-stack-based-octree-traversal-with-direction-flipping.md) by explicitly managing the traversal state via a stack to handle arbitrary ray directions and empty space skipping.

## Operating characteristics
Not given in the sources.

## Background
Origin: Unofficial source from GitHub repository `cpu_voxel_raycaster` by Forceflow [S1, unofficial]. The code is part of a CPU-based voxel raycaster implementation [S1, unofficial].

## Sources
[S1] Forceflow/cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
