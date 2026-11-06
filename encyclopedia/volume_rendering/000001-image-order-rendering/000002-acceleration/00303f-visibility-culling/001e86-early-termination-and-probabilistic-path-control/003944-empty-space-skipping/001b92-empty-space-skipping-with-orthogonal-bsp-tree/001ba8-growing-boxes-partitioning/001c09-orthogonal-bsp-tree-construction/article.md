# Orthogonal BSP Tree Construction

Also called: BSP tree conversion, Orthogonal BSP

## Overview
Orthogonal BSP Tree Construction converts a set of arbitrarily positioned and sized "grown boxes" into an orthogonal Binary Space Partitioning (BSP) tree where every splitting plane is object-aligned [S1]. This conversion solves the problem of non-trivial visibility sorting that arises when box sizes and positions are arbitrary, potentially creating cyclic visibility orders from specific viewpoints [S1]. By enforcing object-aligned splits, it enables trivial depth sorting for 3D texture rendering [S1]. It sets itself apart from its sibling [Growing Boxes Partitioning](1065-growing-boxes-partitioning.md) by taking the adaptive box set as input and structuring it into a strict hierarchy for traversal, and from [K-D Tree Hierarchy Construction](2300-k-d-tree-hierarchy-construction.md) by restricting split planes to coincide with object faces rather than allowing arbitrary positions [S1, S1].

## How it works
**Inputs:** A set of grown boxes, each defined by its bounding box and type (uniform, gradient, or other).
**Outputs:** An orthogonal BSP tree where leaf nodes contain exactly one box (either an original grown box or a child box generated from splitting) [S1].

**Data Structures:**
*   **Box:** Defined by its spatial bounds. It has a `type` field (uniform, gradient, other) and a `parent` pointer.
*   **BSP Node:** Contains:
    *   `room`: A bounding box representing the union of all boxes in the node's list.
    *   `box_list`: A list of boxes associated with this node.
    *   `front`: Pointer to the front child node.
    *   `back`: Pointer to the back child node.
    *   `parent_box`: If this node is a non-leaf node, it may hold a "parent box" that was split to create this node. [S1]

**Algorithm Steps:**

1.  **Initialization:** Create a root BSP node. Set its `room` to the union of all input grown boxes. Add all input grown boxes to the root's `box_list`. [S1]

2.  **Node Splitting:** Recursively split nodes until leaf constraints are met. For a given node:
    *   **Termination:** If the node's `box_list` contains exactly one box, it is a leaf node. Stop splitting. [S1]
    *   **Plane Selection:** Choose a partitioning plane aligned with the faces of at least two boxes in the `box_list` to minimize the number of boxes that need to be split. The plane must split the current `room` into two sub-boxes. [S1]
    *   **Child Creation:** Create two child nodes: `front` and `back`.
    *   **Room Assignment:** Split the current node's `room` using the chosen plane. Assign the resulting front sub-box as the `room` of the `front` child, and the back sub-box as the `room` of the `back` child. [S1]
    *   **Box Redistribution:** Iterate through each box in the current node's `box_list`:
        *   If the box is completely enclosed in the `front` room, move it to the `front` child's `box_list`.
        *   If the box is completely enclosed in the `back` room, move it to the `back` child's `box_list`.
        *   If the box is cut by the partitioning plane:
            *   Split the box into two new boxes along the plane.
            *   Add the front part to the `front` child's `box_list`.
            *   Add the back part to the `back` child's `box_list`.
            *   **Parent-Child Hierarchy:**
                *   If the original box has no parent (it is an original grown box), set the original box as the `parent_box` of the current node (or associate it with the split). [S1]
                *   If the original box has a parent, set that parent as the parent of the two new child boxes. [S1]
                *   Delete the original box from the current node's list. [S1]
    *   **Recursion:** Recursively call the split procedure on the `front` and `back` children.

3.  **Bottom-Up Merging:** After the initial tree construction, reduce the tree size by merging child nodes. For a pair of sibling nodes (`front` and `back`):
    *   Merge if any of the following conditions are true:
        1.  Both children contain uniform boxes, and the union of their boxes forms a uniform box. [S1]
        2.  Both children contain "other" boxes or both contain "gradient" boxes. [S1]
        3.  One of the boxes is tiny or very thin (size along one axis is smaller than a threshold). [S1]
    *   **Merge Action:**
        *   Create a new merged box representing the union of the two child boxes.
        *   Determine the type of the merged box based on the enclosed voxel properties. [S1]
        *   If the merged node replaces the two children, update the parent's reference.
        *   If the two children had different parent boxes, delete those parent boxes and make their children parentless (if they are not leaves). [S1]
    *   Repeat merging until no more merges are possible. [S1]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Box:
    def __init__(self, bounds, box_type, parent=None):
        self.bounds = bounds  # AABB
        self.type = box_type  # 'uniform', 'gradient', 'other'
        self.parent = parent

class BSPNode:
    def __init__(self, room, box_list=None):
        self.room = room      # AABB
        self.box_list = box_list if box_list else []
        self.front = None
        self.back = None
        self.parent_box = None # Box that was split to create this node

def split_node(node):
    if len(node.box_list) == 1:
        return # Leaf node

    # 1. Choose partitioning plane
    # Criterion: Minimize splits, plane coincides with faces of at least two boxes
    plane = choose_partitioning_plane(node.box_list, node.room)
    
    # 2. Create children and split room
    front_room, back_room = split_aabb(node.room, plane)
    front_node = BSPNode(front_room)
    back_node = BSPNode(back_room)
    node.front = front_node
    node.back = back_node

    # 3. Process boxes
    boxes_to_process = list(node.box_list)
    node.box_list = [] # Clear current list as boxes are moved/split

    for box in boxes_to_process:
        if box.is_enclosed_in(front_room):
            front_node.box_list.append(box)
        elif box.is_enclosed_in(back_room):
            back_node.box_list.append(box)
        else:
            # Box is cut by plane
            front_part, back_part = split_box(box, plane)
            
            # Set parent hierarchy
            if box.parent is None:
                # Original grown box becomes parent of this split
                node.parent_box = box
            else:
                # Existing parent applies to new children
                front_part.parent = box.parent
                back_part.parent = box.parent
            
            front_node.box_list.append(front_part)
            back_node.box_list.append(back_part)

    # Recurse
    split_node(front_node)
    split_node(back_node)

def merge_nodes(front_node, back_node):
    # Check merge conditions
    if not can_merge(front_node, back_node):
        return False

    # Create merged box
    merged_bounds = union_aabb(front_node.room, back_node.room)
    merged_type = determine_merged_type(front_node, back_node)
    merged_box = Box(merged_bounds, merged_type)

    # Handle parent boxes
    if front_node.parent_box != back_node.parent_box:
        # Delete conflicting parent boxes if they are no longer needed
        # Note: Logic depends on specific implementation of "parentless" children
        pass
    
    # Replace children with merged node in parent
    # This requires access to the parent of front_node and back_node
    return True

def build_orthogonal_bsp(grown_boxes):
    root_room = union_aabb([b.bounds for b in grown_boxes])
    root = BSPNode(root_room, grown_boxes)
    
    # Initial Splitting
    split_node(root)
    
    # Bottom-Up Merging
    # Traverse tree bottom-up and attempt merges
    merge_all(root)
    
    return root
```

**Parameters:**
*   **Threshold for tiny/thin boxes:** Not given in the sources. [G] (Typically a small fraction of the volume extent or voxel size).

**Edge Cases and Pitfalls:**
*   **Hierarchy Constraint:** No box can be both a child and a parent at the same time. There is only one level of parent-child hierarchy. Parent boxes reside on non-leaf nodes; rendering uses only leaf nodes. [S1]
*   **Texture Sharing:** Child boxes share the texture of their parent with siblings to avoid replicating texels along boundaries. Replicating texels would increase memory requirements. [S1]
*   **Cyclic Visibility:** The conversion is necessary because arbitrary grown boxes may have cyclic visibility orders from specific viewpoints, which trivial sorting cannot handle. [S1]

## Subtopics

### [K-D Tree Hierarchy Construction](2300-k-d-tree-hierarchy-construction.md)
Allows splitting planes to be positioned arbitrarily within a node's bounds (often via bisection) rather than being strictly object-aligned, and associates polygons with the smallest node region that completely contains them, thereby avoiding polygon subdivision. Choose when arbitrary splits are acceptable or beneficial for balance, and when avoiding polygon splitting is critical.

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S1]
*   **Memory Complexity:** Not given in the sources. [S1]
*   **Strengths:** Enables trivial visibility sorting for 3D texture rendering by ensuring object-aligned splits. Keeps the total number of boxes comparable to the original set after merging. [S1]
*   **Limitations:** The total number of boxes at leaf nodes after splitting can be significantly greater than the original set, necessitating the merging step. [S1]
*   **When to use:** When using adaptive partitioning (grown boxes) where arbitrary box sizes and positions would otherwise prevent trivial visibility sorting. [S1]
*   **When not to use:** When uniform partitioning is sufficient, or when arbitrary split planes (K-D tree) are preferred for balancing. [S1]

## Background
This method is part of the "Empty Space Skipping and Occlusion Clipping for Texture-based Volume Rendering" framework. It addresses the limitations of adaptive partitioning where box sizes and positions are arbitrary, leading to non-trivial visibility sorting. The conversion to an orthogonal BSP tree allows the system to leverage the efficiency of texture-based rendering with proper depth sorting. [S1]

## Sources
[S1] Empty Space Skipping and Occlusion Clipping for Texture-based Volume Rendering, https://www3.cs.stonybrook.edu/~mueller/papers/OcclusionClipping.pdf, fetched 2026-10-26
