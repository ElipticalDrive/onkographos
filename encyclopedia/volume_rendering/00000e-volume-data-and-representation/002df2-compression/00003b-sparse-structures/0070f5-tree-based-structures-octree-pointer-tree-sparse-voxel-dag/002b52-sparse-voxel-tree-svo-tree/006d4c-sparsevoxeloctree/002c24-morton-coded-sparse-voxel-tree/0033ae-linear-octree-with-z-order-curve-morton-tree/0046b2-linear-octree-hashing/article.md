# Linear Octree Hashing

Also called: Gargantini's linear octree, Linear Octree, Linear Quadtree, Sorted leaf array lookup

## Overview
Linear Octree Hashing is a pointerless sparse data structure that stores only the leaf nodes of an octree in a one-dimensional array, sorted by their Morton (Z-order) code [S2, unofficial]. It solves the problem of memory-efficient storage for sparse volumetric data by eliminating the overhead of internal node pointers and empty space indicators [S2, unofficial]. Unlike its parent [Linear Octree with Z-order curve (Morton tree)](7054-linear-octree-with-z-order-curve-morton-tree.md), which may store internal structure or rely on implicit traversal, this method relies exclusively on binary search over the sorted leaf array for point location [S1, unofficial][S2, unofficial].

## How it works
**Inputs:** A set of non-zero voxel values in a 3D grid.
**Outputs:** A sorted array of leaf nodes enabling $O(\log L)$ point location.

**Data Structures:**
The structure consists of a single contiguous array of leaf nodes [S2, unofficial].
*   **Leaf Node Layout:** Each entry stores the full Morton code (locational code) as the key. The Morton code is an interleaved base-8 integer representing the 3D coordinates $(x, y, z)$ [S2, unofficial].
*   **Sorting:** The array is sorted in ascending order of the decimal value of the Morton codes [S1, unofficial][S2, unofficial].
*   **Implicit Geometry:** The Morton code implicitly defines the voxel's position and size (level) within the octree hierarchy. A code at level $l$ covers a spatial interval $[M_{min}, M_{max})$ [S3, unofficial].

**Algorithm: Point Location**
To retrieve the value associated with a query point $q$:

1.  **Encode:** Compute the finest-level Morton code $q_M$ for the query coordinates [S1, unofficial][S3, unofficial].
2.  **Search:** Perform a binary search on the sorted leaf array to find the position of $q_M$.
3.  **Identify Neighbors:** Let $L_{prev}$ be the leaf node immediately preceding the insertion point (predecessor) and $L_{next}$ be the leaf node at the insertion point or immediately following (successor) [S3, unofficial].
4.  **Check Exact Hit:** If $L_{next}$ exists and its Morton code equals $q_M$, return $L_{next}$ as the occupied voxel [S3, unofficial].
5.  **Check Predecessor Coverage:** If $L_{prev}$ exists, calculate the finest-level Morton interval $[M_{min}(L_{prev}), M_{max}(L_{prev}))$ covered by $L_{prev}$. If $q_M$ falls within this interval ($M_{min}(L_{prev}) \le q_M < M_{max}(L_{prev})$), return $L_{prev}$ as the occupied voxel [S3, unofficial].
6.  **Check Empty Space:** If no occupied voxel contains the point, iterate from level $l = 0$ to the maximum depth $L$. For each level, compute the coarser Morton code $q_{M,l} = \text{MortonEncode}(q, l)$. Check if the interval of the successor leaf at the finest level is contained within the interval of the current level's predecessor and successor leaves. If so, return the empty voxel at level $l$ [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_octree_lookup(leaf_array, query_coords, max_level):
    """
    leaf_array: List of tuples (morton_code, value), sorted by morton_code.
    query_coords: Tuple (x, y, z).
    max_level: Maximum depth of the octree.
    Returns: (morton_code, value, status) where status 1=occupied, 0=empty.
    """
    # 1. Encode query to finest level
    q_M = morton_encode(query_coords, max_level)
    
    # 2. Binary search for insertion point
    # bisect_right returns index i such that all e in leaf_array[:i] have e.key <= q_M
    # and all e in leaf_array[i:] have e.key > q_M
    import bisect
    keys = [leaf.morton_code for leaf in leaf_array]
    idx = bisect.bisect_right(keys, q_M)
    
    # 3. Identify neighbors
    # Successor is at idx (if idx < len)
    # Predecessor is at idx - 1 (if idx > 0)
    
    # 4. Check Exact Hit
    if idx < len(leaf_array) and leaf_array[idx].morton_code == q_M:
        return leaf_array[idx].morton_code, leaf_array[idx].value, 1
        
    # 5. Check Predecessor Coverage
    if idx > 0:
        prev_leaf = leaf_array[idx - 1]
        prev_code = prev_leaf.morton_code
        # Determine the level of the predecessor from its code or stored level
        # Assuming we can derive level from code or it's stored. 
        # If not stored, we must infer or store it. 
        # For this implementation, we assume we can compute the interval bounds.
        # The finest-level interval of a leaf at level l with code C is:
        # [C * 2^(3*(max_level-l)), (C+1) * 2^(3*(max_level-l)))
        # However, standard Morton intervals are defined by the code itself.
        # A simpler check often used: if the query code falls within the 
        # spatial extent of the predecessor.
        
        # Calculate bounds for predecessor at its specific level l_prev
        # Note: The source [S3, unofficial] implies checking if q_M is in the interval 
        # covered by the predecessor leaf.
        l_prev = get_level_from_morton(prev_code) # Helper to extract level
        step = 3 * (max_level - l_prev)
        min_prev = prev_code << step
        max_prev = (prev_code + 1) << step
        
        if min_prev <= q_M < max_prev:
            return prev_leaf.morton_code, prev_leaf.value, 1

    # 6. Check Empty Space (Largest Empty Voxel)
    # Iterate levels to find the largest empty voxel containing q
    for l in range(max_level + 1):
        q_M_l = morton_encode(query_coords, l)
        
        # Get predecessor and successor at level l
        # We need to find leaves that bound the interval of q_M_l
        # This requires searching for the interval [q_M_l * 2^(3*(max_level-l)), (q_M_l+1)*2^(3*(max_level-l)))
        
        # Simplified logic from [S3, unofficial]: Check if the successor leaf's finest interval
        # is contained within the predecessor/successor intervals at level l.
        # This part is complex to implement without storing level info explicitly.
        # A robust implementation stores (morton_code, level, value).
        
        # Placeholder for complex interval containment check as per [S3, unofficial]
        # if is_empty_voxel_containing(leaf_array, q_M_l, l):
        #     return q_M_l, None, 0
        pass

    return None, None, 0 # Not found or error

def morton_encode(coords, level):
    # Standard Morton encoding for x,y,z truncated to 'level' bits
    x, y, z = coords
    # Mask to keep only 'level' bits
    mask = (1 << level) - 1
    x &= mask
    y &= mask
    z &= mask
    
    m = 0
    for i in range(level):
        m |= (x & 1) << (2*i)
        m |= (y & 1) << (2*i + 1)
        m |= (z & 1) << (2*i + 2)
    return m

def get_level_from_morton(code):
    # In a pure linear octree, the level is often implicit or stored.
    # If not stored, one might need to store it. 
    # [S2, unofficial] notes only leaf nodes are stored.
    # If level is not stored, this helper is ambiguous without additional context.
    # [G] Assuming level is stored or derivable. If not, the structure is incomplete.
    return 0 # Placeholder
```

**Parameters:**
*   **Max Level:** The maximum depth of the octree, determining the resolution of the finest voxels.
*   **Leaf Array:** The sorted list of occupied voxels.

**Edge Cases and Pitfalls:**
*   **No Interior Nodes:** The structure does not store interior nodes or pointers, making it unsuitable for applications requiring traversal of non-leaf structures or neighbor finding without additional data [S2, unofficial].
*   **Empty Space:** Determining the "largest empty voxel" requires iterating levels and checking interval containment, which is more complex than simple point location [S3, unofficial].
*   **Precision:** Morton codes are integers. Precision issues may arise if coordinates are not aligned to the grid or if the max level exceeds integer bit-widths.

## Subtopics

### [Linear Level Octree (LLO)](16467-linear-level-octree-llo.md)
Linear Level Octree (LLO) is a hierarchical data structure for volume data that stores only leaf nodes, implicitly encoding location, size, and path from root to nodes to reduce memory and processing cost. It extends the linear octree scheme by using a specific labeling system that allows efficient estimation of spatial relations between points and nodes. Unlike its parent [Linear Octree Hashing](11794-linear-octree-hashing.md), which relies on hash tables for lookup, LLO uses a deterministic coordinate-based key system that facilitates direct geometric calculations without hashing overhead.

### [Ray-walking with Linear-Octree and DDA](23332-ray-walking-with-linear-octree-and-dda.md)
This method performs ray traversal on an Adaptive Mesh Refinement (AMR) grid stored in a linear-octree structure, combining DDA (Digital Differential Analyzer) stepping with Morton code indexing to locate leaf cells[unofficial]. It solves the problem of efficiently sampling sparse, multi-resolution volumetric data by restricting traversal only to cells actually intersected by the ray, avoiding the overhead of pointer-based tree traversal or uniform grid sampling[unofficial]. It differs from sibling methods like [Linear Level Octree (LLO)](16467-linear-level-octree-llo.md) by relying on a flat array of leaf nodes sorted by Morton code rather than explicit level-based indexing structures.

### Sparse Octree Query via Morton Encoding
Sparse Octree Query via Morton Encoding is a method to query a sparse voxel field stored in a flat, sorted array of Morton codes. It maps a query point to its finest-level Morton code and uses binary search to find the occupied voxel containing the point or the largest empty voxel interval to skip [S3, unofficial]. The algorithm checks for an exact hit, then checks if the query falls within the predecessor leaf's interval, and finally iterates levels to find the largest empty voxel containing the query point [S3, unofficial].

## Operating characteristics
**Time Complexity:**
*   **Lookup:** $O(\log L)$, where $L$ is the number of leaf nodes [S1, unofficial][S2, unofficial].
*   **Comparison:** Slower to hash than pointer octrees ($O(\log_2 L)$ vs $O(\log_8 N)$) except in highly vertical trees with few leaf nodes relative to interior nodes [S2, unofficial].

**Memory Complexity:**
*   **Storage:** Stores only leaf nodes, eliminating pointers and internal node data [S2, unofficial].
*   **Efficiency:** Lower memory consumption compared to pointer-based quadtrees/octrees [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** High memory efficiency for sparse data; simple contiguous memory layout; no pointer chasing overhead [S2, unofficial].
*   **Limitations:** Higher runtime for lookups compared to pointer-based methods in many cases; no support for efficient neighbor finding or internal node traversal [S1, unofficial][S2, unofficial].
*   **When to Use:** Applications where storage is of utmost importance and only leaf node access is required [S2, unofficial].
*   **When Not to Use:** Applications requiring frequent traversal of internal nodes or neighbor queries [S2, unofficial].

## Background
The linear octree was introduced by Gargantini [S1, unofficial]. It is an extension of the linear quadtree to three dimensions [S1, unofficial]. The linear quadtree was originally proposed by Gargantini for quadtrees [S2, unofficial]. The structure uses an interleaved base-8 code similar to that proposed by Morton for traditional octrees [S2, unofficial]. It is an intriguing structure for its potential compression abilities in volume rendering [S2, unofficial].

## Sources
[S1] Eine adaptive, hybride Datenstruktur für Volumen Datensätze auf der GPU, https://doi.org/10.34726/hss.2016.26562, fetched 2026-10-26 (unofficial)
[S2] PDF A Survey of Octree Volume Rendering Methods, https://www.sci.utah.edu/~knolla/octsurvey.pdf, fetched 2026-10-29 (unofficial)
[S3] SpaRV: Sparse Radiance Voxels for Scalable Geometrically Consistent LiDAR-Visual Mapping and Rendering, https://doi.org/10.48550/arxiv.2610.25665, fetched 2026-10-26 (unofficial)
