# Fast Projection for Octree Nodes


## Overview
Fast Projection for Octree Nodes computes the projected center of an octree node from its parent's projected center using a simple 2D translation [S1]. It preprocesses eight constant 2D translation vectors for every octree level to quickly determine subdivisions, affected pixels, and recursively update occlusion maps (HOMs) [S1]. The same translation approach is used to compute the depth component for fast depth cueing [S1].

## How it works
**Inputs:**
- Parent node's projected center $(x_p, y_p)$ in screen space [S1].
- Octree level $L$ [S1].
- Child octant index $i \in \{0, \dots, 7\}$ [S1].

**Outputs:**
- Child node's projected center $(x_c, y_c)$ [S1].

**Data Structures:**
- `Translations[L][8][2]`: A 3D array storing precomputed 2D translation vectors.
  - `L`: Octree level index.
  - `8`: The eight possible octants (children).
  - `2`: The $(dx, dy)$ translation components.
  - Fields: `level` (int), `octant` (int), `dx` (32-bit int), `dy` (32-bit int).
  - Sizes: $L$ levels, 8 octants, 2 components.
  - Bit packing: Not given in the sources.

**Steps:**
1. **Preprocessing:** For each octree level $L$, compute and store eight constant 2D translation vectors $(dx_{L,i}, dy_{L,i})$ corresponding to the eight children [S1].
2. **Lookup:** Retrieve the translation vector $(dx, dy) = \text{Translations}[L][i]$ for the target octant $i$ [S1].
3. **Translation:** Compute the child's projected center using 32-bit integer addition:
   $$ x_c = x_p + dx $$
   $$ y_c = y_p + dy $$
4. **Recursive Update:** Use $(x_c, y_c)$ to determine subdivisions, assign pixels, and recursively update occlusion maps (HOMs) [S1]. The same $(x_c, y_c)$ is also used to compute the depth component for fast depth cueing [S1].

**Parameters:**
- **Arithmetic Precision:** 32-bit integer arithmetic is used for subdivision determination, pixel assignment, and recursive HOM updates [S1].
- **Precomputed Vectors:** Eight constant 2D translation vectors per octree level [S1].

**Edge Cases and Pitfalls:**
- **Precision:** The method relies on 32-bit integer arithmetic, which may limit maximum screen resolution or octree depth before overflow occurs [S1].
- **Boundary Conditions:** Not given in the sources.
- **Degenerate Input:** Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_child_projected_center(parent_center_x, parent_center_y, level, octant_index, translations):
    """
    Computes the projected center of an octree child node using precomputed 2D translations.
    
    Args:
        parent_center_x (int): X coordinate of the parent's projected center.
        parent_center_y (int): Y coordinate of the parent's projected center.
        level (int): Current octree level.
        octant_index (int): Index of the child octant (0-7).
        translations (list): Precomputed translation vectors [level][octant][2].
        
    Returns:
        tuple: (child_x, child_y) as 32-bit integers.
    """
    # Retrieve precomputed translation vector for this level and octant
    dx, dy = translations[level][octant_index]
    
    # Compute child center using 32-bit integer arithmetic
    child_x = parent_center_x + dx
    child_y = parent_center_y + dy
    
    return child_x, child_y
# Recursive update of HOMs and depth cueing is performed externally using (child_x, child_y) [S1].
```

## Subtopics

### [Sparse Octree Filtering with Node Map](2337-sparse-octree-filtering-with-node-map.md)
Sparse Octree Filtering with Node Map is a thread-scheduling heuristic for MIP-map generation in sparse voxel octrees that reduces redundant computation by terminating duplicate threads. It solves the problem of multiple threads performing identical filtering work on the same octree node, which is frequent at higher MIP-map levels. It differs from standard parallel MIP-map generation by using a 2D auxiliary structure, the node map, to detect common ancestors among threads before computation begins. It is chosen when parallel MIP-map generation causes excessive redundant filtering at deeper levels.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** High speed due to integer arithmetic and precomputation [S1].
- **Limitations:** Limited to scenarios where 32-bit integer precision is sufficient for the screen resolution and octree depth [S1].
- **When to use:** When projecting octree nodes to screen space for occlusion culling or depth cueing in object-order rendering [S1].
- **When not to use:** Not given in the sources.

## Background
The method originates from research on object-order ray casting [S1]. It leverages hierarchical data structures and efficient screen-space mapping to optimize traversal and shading pipelines [S1].

## Sources
[S1] A new object-order ray-casting algorithm, https://doi.org/10.5555/602099.602130, fetched 2026-10-26
