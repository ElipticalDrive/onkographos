# Octree with Neighbor Pointers

Also called: Octree Construction with X-Neighbor Pointers, Octree with neighbors structure

## Overview
A spatial index structure that augments a branch-on-need octree with links to allow direct traversal to adjacent nodes, used to accelerate ray traversal in volume rendering [S2]. It differs from full 26-neighbor structures by maintaining only x-neighbor pointers, reducing memory overhead while enabling efficient skipping of empty space and boundary crossing [S1].

## How it works
**Inputs:** A view-dependent branch-on-need octree.
**Outputs:** A 3D texture acting as a spatial index, encoded with tagged pointers and neighbor links.

**Data Structures:**
The structure is encoded into an 8-bit RGBA 3D texture [S2]. Each octree node is encoded in 8 consecutive texels arranged in the x-direction [S2]. The layout uses a tagged pointer scheme where the Alpha (A) component determines the node kind:
*   $A=1.0$: Inner node.
*   $A=0.5$: Data node (leaf with data).
*   $A=0.0$: Empty node [S2].

**Inner Node Layout:**
The RGB components point to the first of 8 children, which are arranged consecutively in the x-direction [S2].

**Leaf Node Layout:**
The RGB components point to leaf info, consisting of:
1.  A data pointer (if not empty).
2.  6 consecutive texels storing neighbor pointers for the 6 faces [S2].
The last leaf data texel contains a NULL pointer [S2]. Neighbor pointers use their A value to encode the octree level of the neighbor [S2].

**Construction Steps:**
1.  **Top-Level Initialization:** Create all nodes at the top level $n$ (typically two levels below the root to speed up traversal) and establish x-neighbor pointers for all adjacent nodes at this level [S1].
2.  **Pointer Propagation:** Propagate x-neighbor pointers from nodes in level $i$ to their child nodes in level $i-1$. Specifically, link adjacent level-$(i-1)$ nodes that have different parent nodes [S1].
3.  **Texture Encoding:** Encode the resulting structure into a 3D texture. Update the GPU texture using `glTexSubImage` [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_octree_with_neighbors(octree):
    # Step 1: Initialize top level
    top_level = octree.get_level(n) # n is typically root_level - 2
    for node in top_level:
        node.neighbors_x = find_x_neighbors(node, top_level)
        
    # Step 2: Propagate pointers down
    for level in range(n, 0, -1):
        current_level_nodes = octree.get_level(level)
        child_level_nodes = octree.get_level(level - 1)
        
        for parent in current_level_nodes:
            children = parent.get_children()
            # Link children that share a boundary but have different parents
            for i, child in enumerate(children):
                # Find adjacent children in the same level
                # Logic depends on Morton ordering or spatial adjacency
                adjacent_children = find_adjacent_children(child, child_level_nodes)
                for adj_child in adjacent_children:
                    if adj_child.parent != parent:
                        child.set_neighbor(adj_child)
                        
    # Step 3: Encode to 3D texture
    texture = create_3d_texture(width, height, depth)
    for node in octree.all_nodes():
        encode_node_to_texture(node, texture)
        
    return texture

def find_x_neighbors(node, level_nodes):
    # Returns pointer to the node immediately adjacent in the +x direction
    # at the same level
    pass

def encode_node_to_texture(node, texture):
    # Determine node type
    if node.is_empty:
        alpha = 0.0
        rgb = NULL
    elif node.is_leaf:
        alpha = 0.5
        # RGB points to data pointer
        # Next 6 texels store neighbor pointers
        # Last texel is NULL
    else:
        alpha = 1.0
        # RGB points to first child (8 children follow in x)
    pass
```

**Edge Cases:**
*   **Boundary Nodes:** Neighbor pointers link to the smallest node enclosing all adjacent nodes if there are multiple ones [S2].
*   **Depth Limit:** Traversal stops before reaching a single root node at the top level to optimize performance [S1].

## Operating characteristics
*   **Memory:** Uses a 3D texture with 8-bit RGBA components. Each node occupies 8 texels. The addressing space can potentially address $16 \times M$ nodes, which is several orders of magnitude larger than needed [S2].
*   **Performance:** On-the-fly construction at each frame allows for view-dependent optimization [S2].
*   **Comparison:** Differs from Zhou et al. [2010] by storing different payload (mesh vs point cloud) and maintaining only x-neighbor pointers instead of pointers to all 26 neighbors [S1].

## Background
Originated in the context of GPU-based volume rendering and voxelization. The structure was introduced to accelerate ray casting by providing direct access to adjacent nodes, avoiding backtracking [S2]. It shares similarities with bottom-up breadth-first construction using Morton ordering [S1].

## Sources
[S1] Fast Parallel Surface and Solid Voxelization on GPUs, https://www.michael-schwarz.com/research/publ/files/vox-siga10.pdf, fetched 2026-10-27
[S2] A single-pass GPU ray casting framework for interactive out-of-core ..., https://www.crs4.it/vic/data/papers/cgi2008-movr.pdf, fetched 2026-10-29
