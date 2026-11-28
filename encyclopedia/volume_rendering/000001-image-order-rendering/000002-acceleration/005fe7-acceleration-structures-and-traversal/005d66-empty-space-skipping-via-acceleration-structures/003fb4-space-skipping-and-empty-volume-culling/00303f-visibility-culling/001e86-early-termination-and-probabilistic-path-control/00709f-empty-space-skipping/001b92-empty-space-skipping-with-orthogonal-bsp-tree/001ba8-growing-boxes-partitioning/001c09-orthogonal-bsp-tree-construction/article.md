# Orthogonal BSP Tree Construction

Also called: BSP tree conversion, Orthogonal BSP

## Overview
Orthogonal BSP Tree Construction converts a set of arbitrarily sized, grown boxes into a binary space-partitioning (BSP) tree where every splitting plane is object-aligned (parallel to the coordinate axes) [S1]. This structure enables trivial visibility sorting for 3D texture rendering, solving the problem that arbitrary box positions prevent simple front-to-back traversal [S1]. It sets itself apart from its parent method, [Growing Boxes Partitioning](1065-growing-boxes-partitioning.md), by imposing a strict hierarchical spatial index on the adaptive volumes, whereas the parent method only produces a flat list of clusters [G].

## How it works
**Inputs:** A list of grown boxes from [Growing Boxes Partitioning](1065-growing-boxes-partitioning.md) [S1].
**Outputs:** A BSP tree where leaf nodes contain exactly one box (either an original grown box or a child box generated from splitting) [S1].

**Data Structures:**
*   **Box:** `min` (3D vector), `max` (3D vector), `type` (enum: uniform, gradient, other), `parent` (pointer to parent Box or null) [S1].
*   **BSP Node:** `boxes` (list of Box pointers), `room` (3D bounding box, union of all boxes in `boxes`), `front` (pointer to child BSP Node), `back` (pointer to child BSP Node), `split_plane` (axis-aligned plane) [S1].

**Algorithm Steps:**
1.  **Initialization:** Create a root BSP Node [S1]. Set its `boxes` list to all input grown boxes [S1]. Compute `room` as the union of all box bounding boxes [S1].
2.  **Recursive Splitting:** Process a node $N$ with room $R$ and box list $L$ [S1]. If $N$ is a leaf (e.g., contains only one box or meets a stopping criterion), return [G]. Choose an axis-aligned plane $P$ that coincides with the faces of at least two boxes in $L$ to minimize the number of boxes split by $P$ [S1]. Create two child nodes, $N_{front}$ and $N_{back}$ [S1]. Split $R$ by $P$ into two sub-boxes $R_{front}$ and $R_{back}$ [S1]. Set $N_{front}.room = R_{front}$ and $N_{back}.room = R_{back}$ [S1]. For each box $B$ in $L$: if $B$ is completely enclosed in $R_{front}$, move $B$ to $N_{front}.boxes$; if enclosed in $R_{back}$, move to $N_{back}.boxes$; if cut by $P$, split $B$ into $B_{front}$ and $B_{back}$ by $P$, add them to respective children lists, set parent pointers (if $B.parent$ is null, set new boxes' parent to $B$; else set to $B.parent$), and delete $B$ from $L$ [S1]. Recurse on $N_{front}$ and $N_{back}$ [S1].
3.  **Bottom-Up Merging:** After the tree is built, traverse it bottom-up to reduce node count [S1]. Merge $N_{front}$ and $N_{back}$ into a single node if: (1) both children are uniform boxes and the merged box is still uniform; (2) both are 'other' or both are 'gradient' boxes; (3) one box is tiny or very thin (size along one axis < threshold) [S1]. If merged, the new node's `boxes` list contains the union of the children's boxes [S1]. If the children had different parent boxes, those parent boxes are deleted and all their children become parentless [S1]. The type of the resulting merged box is redetermined based on enclosed voxel properties [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Box:
    def __init__(self, min_pt, max_pt, box_type='other', parent=None):
        self.min = min_pt
        self.max = max_pt
        self.type = box_type
        self.parent = parent

    def is_enclosed_in(self, room):
        return all(self.min[i] >= room.min[i] and self.max[i] <= room.max[i] for i in range(3))

    def is_cut_by(self, plane):
        axis, coord = plane
        return self.min[axis] < coord and self.max[axis] > coord

    def split_by(self, plane):
        axis, coord = plane
        front_min, front_max = list(self.min), list(self.max)
        back_min, back_max = list(self.min), list(self.max)
        front_max[axis] = coord
        back_min[axis] = coord
        return Box(front_min, front_max, self.type, None), Box(back_min, back_max, self.type, None)

class BSPNode:
    def __init__(self):
        self.boxes = []
        self.room = None
        self.front = None
        self.back = None
        self.split_plane = None
        self.type = 'other'

    def is_leaf(self):
        return self.front is None and self.back is None

def choose_optimal_plane(boxes, room):
    # Candidate planes are defined by the min/max faces of the boxes along each axis.
    # Select the plane that minimizes the number of boxes intersected. [G]
    candidates = set()
    for b in boxes:
        for i in range(3):
            candidates.add((i, b.min[i]))
            candidates.add((i, b.max[i]))
    best_plane = None
    min_splits = float('inf')
    for axis, coord in candidates:
        splits = sum(1 for b in boxes if b.min[axis] < coord and b.max[axis] > coord)
        if splits < min_splits:
            min_splits = splits
            best_plane = (axis, coord)
    return best_plane

def split_node(node):
    if node.is_leaf():
        return
    plane = choose_optimal_plane(node.boxes, node.room)
    front_node = BSPNode()
    back_node = BSPNode()
    front_node.room, back_node.room = split_box(node.room, plane)
    node.front = front_node
    node.back = back_node
    boxes_to_process = list(node.boxes)
    node.boxes = []
    for box in boxes_to_process:
        if box.is_enclosed_in(front_node.room):
            front_node.boxes.append(box)
        elif box.is_enclosed_in(back_node.room):
            back_node.boxes.append(box)
        elif box.is_cut_by(plane):
            front_box, back_box = box.split_by(plane)
            front_node.boxes.append(front_box)
            back_node.boxes.append(back_box)
            if box.parent is None:
                front_box.parent = box
                back_box.parent = box
            else:
                front_box.parent = box.parent
                back_box.parent = box.parent
    split_node(front_node)
    split_node(back_node)

def split_box(bounding_box, plane):
    axis, coord = plane
    front_min, front_max = list(bounding_box.min), list(bounding_box.max)
    back_min, back_max = list(bounding_box.min), list(bounding_box.max)
    front_max[axis] = coord
    back_min[axis] = coord
    return Box(front_min, front_max, bounding_box.type, bounding_box.parent), \
           Box(back_min, back_max, bounding_box.type, bounding_box.parent)

def merge_bottom_up(node):
    if node.is_leaf():
        return node
    node.front = merge_bottom_up(node.front)
    node.back = merge_bottom_up(node.back)
    if can_merge(node.front, node.back):
        merged_node = BSPNode()
        merged_node.room = union_boxes(node.front.room, node.back.room)
        merged_node.boxes = node.front.boxes + node.back.boxes
        merged_node.type = determine_type(merged_node.boxes)
        for box in merged_node.boxes:
            if box.parent is not None:
                box.parent = None
        return merged_node
    return node

def can_merge(front_node, back_node):
    if front_node.type == 'uniform' and back_node.type == 'uniform':
        if is_uniform(union_boxes(front_node.room, back_node.room)):
            return True
    if front_node.type == back_node.type:
        if front_node.type in ['other', 'gradient']:
            return True
    if is_tiny_or_thin(front_node.room) or is_tiny_or_thin(back_node.room):
        return True
    return False

def union_boxes(box1, box2):
    return Box([min(box1.min[i], box2.min[i]) for i in range(3)],
               [max(box1.max[i], box2.max[i]) for i in range(3)])

def determine_type(boxes):
    # Redetermines type based on enclosed voxel properties. [S1]
    return 'uniform'

def is_uniform(box):
    # Checks if box contains uniform voxels. [G]
    return True

def is_tiny_or_thin(box):
    # Checks if size along one axis < threshold. [S1]
    threshold = 1.0
    return any(box.max[i] - box.min[i] < threshold for i in range(3))
```

**Parameters:**
*   **Threshold for tiny/thin:** Not given in the sources. [G] Typically a small fraction of the volume dimensions or a fixed voxel count.
*   **Stopping Criterion for Splitting:** Not given in the sources. [G] Typically when a node contains a single box or the box count is below a threshold.

**Edge Cases and Pitfalls:**
*   **Hierarchy Constraint:** No box can be a child and a parent at the same time. There is only one level of parent-child hierarchy. Parent boxes reside on non-leaf nodes; rendering uses only leaf nodes [S1].
*   **Texture Sharing:** Child boxes share the texture of their parent with siblings to avoid replicating texels along boundaries. Replicating texels would increase memory requirements [S1].
*   **Box Count:** The total number of boxes at leaf nodes after splitting can be significantly greater than the original grown boxes, necessitating the bottom-up merging step [S1].

## Subtopics

### [K-D Tree Hierarchy Construction](2300-k-d-tree-hierarchy-construction.md)
Builds a binary space-partitioning structure to organize geometric primitives for efficient spatial queries. It sets itself apart by classifying polygons based on vertex coordinates relative to the splitting hyperplane and retaining spanning polygons at the node, thereby avoiding explicit polygon clipping.

## Operating characteristics
*   **Time Complexity:** Splitting requires iterating over boxes and potentially splitting them. Choosing the optimal plane requires checking candidate planes defined by box faces. Merging is a bottom-up pass. Exact complexity not given [S1]. [G] Likely $O(N \log N)$ or $O(N^2)$ depending on plane selection strategy, where $N$ is the number of boxes.
*   **Memory Complexity:** Stores boxes and tree nodes. Texture sharing reduces memory overhead compared to replicating textures. Exact complexity not given [S1].
*   **Strengths:** Enables trivial visibility sorting for 3D texture rendering. Handles arbitrary box sizes and positions [S1].
*   **Limitations:** Requires bottom-up merging to control box count. Texture sharing adds complexity to texture coordinate computation [S1].
*   **When to use:** When using texture-based volume rendering with adaptive partitioning (grown boxes) and needing efficient visibility culling [S1].
*   **When not to use:** When uniform partitioning is sufficient, or when geometric primitives are better suited for K-D trees [S1].

## Background
Originated in the context of adaptive partitioning for texture-based volume rendering [S1]. The method addresses the issue that arbitrary box sizes and positions from adaptive partitioning prevent trivial visibility sorting [S1]. The conversion to an orthogonal BSP tree allows for efficient occlusion clipping and empty-space skipping [S1].

## Sources
[S1] Empty Space Skipping and Occlusion Clipping for Texture-based Volume ..., https://www3.cs.stonybrook.edu/~mueller/papers/OcclusionClipping.pdf, fetched 2026-10-26
