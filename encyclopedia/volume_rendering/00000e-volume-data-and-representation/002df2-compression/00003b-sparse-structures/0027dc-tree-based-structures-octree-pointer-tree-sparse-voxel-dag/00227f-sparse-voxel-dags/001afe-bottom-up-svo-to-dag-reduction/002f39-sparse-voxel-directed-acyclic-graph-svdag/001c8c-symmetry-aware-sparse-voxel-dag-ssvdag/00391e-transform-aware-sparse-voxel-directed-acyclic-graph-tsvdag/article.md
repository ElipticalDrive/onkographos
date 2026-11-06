# Transform-Aware Sparse Voxel Directed Acyclic Graph (TSVDAG)

Also called: Destructive Translation Matching, Destructive Translations, Translation-aware SVDAG

## Overview
TSVDAG extends Sparse Voxel Directed Acyclic Graphs (SVDAG) by detecting and encoding repeating geometry that aligns with node boundaries through destructive translations, mirror symmetries, and axis permutations [S1]. It solves the limitation of standard SVDAGs, which only merge subtrees that are identical in position and orientation, by allowing voxels to be translated out of a subtree (disappearing) to match translated counterparts [S1]. This enables significantly higher compression ratios for scenes with repetitive structures, at the cost of increased preprocessing complexity and rendering overhead due to variable-length pointer encoding and additional traversal state [S1].

## How it works
**Inputs:** A Sparse Voxel Octree (SVO) or a pre-computed SVDAG/SSVDAG structure.
**Outputs:** A TSVDAG structure with transformed pointers and a compressed memory layout.

### Data Structures and Layout

**1. Node Header**
Each node begins with a 16-bit mask. This mask represents eight 2-bit integers, where each integer specifies the length (in bits) of the corresponding child pointer. A length of 0 indicates an empty child [S1].
*   **Field:** `mask` (16 bits)
*   **Layout:** 8 x 2-bit fields.

**2. Child Pointers**
Pointers are variable-length: 16, 32, or 48 bits. They are sorted by frequency so that the most referenced nodes have the shortest pointers [S1].
*   **16-bit pointers:** Always encode an index into a Level-specific Look-Up Table (LUT).
*   **32-bit pointers:** The most significant bit (MSB) indicates if it is a LUT index (MSB=1) or a direct pointer (MSB=0). If direct, it points to a 16-bit aligned address.
*   **48-bit pointers:** Always direct pointers.

**3. Transform ID**
Stored in the least-significant bits of the pointer (before the address). It combines:
*   **Child-Reorder Transformation ID:** Identifies symmetry or axis permutation.
*   **Translation Mask:** A 3-bit mask indicating zero translation along each of the primary axes (X, Y, Z). If an axis has non-zero translation, its corresponding bit is 0; if zero, it is 1 [S1].
*   **Encoding:** The Transform ID is Huffman-coded to minimize bits for frequent transforms [S1].

**4. Translation Data**
If the Translation Mask indicates non-zero translation for an axis, the translation distance is stored in the next significant bits.
*   **Size:** 5 bits per axis with non-zero translation [S1].

**5. Memory Address**
The remaining most-significant bits store the memory address of the child node. The address space is indexed at 16-bit intervals [S1].

**6. Look-Up Table (LUT)**
One LUT is constructed per SVDAG level.
*   **Entry Size:** Fixed 64 bits.
*   **Content:** Stores the full pointer (Transform ID + Translation + Address) for frequently used pointers.
*   **Construction:** Greedy algorithm. Sort pointers by frequency. Iterate and insert into LUT if `Cost(Direct) > Cost(LUT)`.
    *   `Cost(Direct) = Rounded_Pointer_Length * Occurrences`
    *   `Cost(LUT) = 64 * Occurrences` [S1].

### Algorithm Steps

#### 1. Fast Translation Search
To find destructive translation matches between subtrees (flattened to $(2^L)^3$ voxel grids, with $L=4$):
1.  Compute a hash for each voxel describing the area to the top/right.
2.  Compute hashes horizontally, then vertically [S1].
3.  A match between the lower/left voxel of grid B and a voxel in grid A indicates B is constructable from A by a translation of $(-1, -1)$ in the hash space.
4.  Test for symmetry simultaneously: A voxel in direction $(+x, +y, -z)$ matching a corner voxel in $(+x, +y, +z)$ indicates a transformation by translation and z-axis symmetry [S1].
5.  Complexity: $O(NV)$, where $N$ is the number of grids and $V=(2^L)^3$ [S1].

#### 2. Dependency Graph Construction
1.  Flatten subtrees into voxel grids.
2.  Identify parent-child relationships where a parent subtree can be transformed into a child subtree.
3.  If multiple parents exist, store the one with the highest memory address to encourage clustering [S1].
4.  Sort subtrees by the number of occupied voxels to favor destructive translations [S1].
5.  Limit translation search to subtrees of up to $16^3$ voxels ($L=4$). Higher levels have diminishing returns and require more bits for the translation range $[-2^L+1, +2^L-1]^3$ [S1].

#### 3. Pointer Encoding and LUT Construction
1.  Sort nodes such that the most referenced ones appear first in memory [S1].
2.  For each level, collect all child pointers.
3.  Sort pointers by frequency.
4.  Iterate through sorted pointers:
    *   Calculate direct cost: Length in bits (rounded up to 32 or 48) $\times$ occurrences.
    *   Calculate LUT cost: 64 bits $\times$ occurrences.
    *   If LUT cost is lower, add pointer to LUT.
5.  Encode pointers:
    *   If in LUT: Encode as 16-bit index.
    *   Else: Encode as 32-bit (with MSB flag) or 48-bit direct pointer.
6.  Apply Huffman coding to the Transform ID field [S1].

#### 4. Rendering (Ray Traversal)
1.  **Ray-Node Intersection:** Compute intersection points between the ray and the three primary planes passing through the node center. Create a bitmask of intersected children.
2.  **Traversal Order:** Determine order by sign bits of the ray direction [S1].
3.  **Child Reordering:**
    *   Store transformation state on a traversal stack.
    *   Use a GPU constant memory LUT to apply permutations and pre-composed transformations ($T = T_i \circ T_j$) [S1].
4.  **Translation Handling:**
    *   Use a second stack for levels containing translations (up to $16^3$).
    *   Stack entry stores: Accumulated translation, ray entry distance ($t_{min}$), and ray exit distance ($t_{max}$).
    *   Translate child node centers using accumulated translation.
    *   Limit intersection bounds to remove voxels moved outside subtree bounds [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_hash(grid):
    """Compute hash for each voxel describing area to top/right."""
    rows, cols = grid.shape
    hash_grid = zeros_like(grid)
    
    # Horizontal pass
    for y in range(rows):
        current_hash = 0
        for x in range(cols):
            # Hash function combining voxel value and previous hash
            current_hash = hash_function(grid[y, x], current_hash)
            hash_grid[y, x] = current_hash
            
    # Vertical pass
    final_hash_grid = zeros_like(grid)
    for x in range(cols):
        current_hash = 0
        for y in range(rows):
            current_hash = hash_function(hash_grid[y, x], current_hash)
            final_hash_grid[y, x] = current_hash
            
    return final_hash_grid

def find_translation_matches(grid_A, grid_B):
    """
    Check if grid_B can be constructed from grid_A by destructive translation.
    Returns translation vector if match found, else None.
    """
    hash_A = compute_hash(grid_A)
    hash_B = compute_hash(grid_B)
    
    # Check for match at lower/left voxel of B against A
    # This indicates translation of (-1, -1) in hash space
    # In 3D, this generalizes to checking offsets in x, y, z
    
    # Simplified 2D logic for demonstration of hash matching
    # In 3D, iterate over possible translation vectors within range [-2^L+1, 2^L-1]
    
    for tx in range(-15, 16):
        for ty in range(-15, 16):
            for tz in range(-15, 16):
                if check_translation_match(hash_A, hash_B, tx, ty, tz):
                    return (tx, ty, tz)
    return None

def check_translation_match(hash_A, hash_B, tx, ty, tz):
    """Check if shifting hash_B by (tx, ty, tz) matches hash_A."""
    # Implementation details depend on specific hash function
    # Compare voxels in B shifted by (tx, ty, tz) against A
    # Handle boundary conditions (destructive translation)
    pass

def build_pointer_encoding(nodes, level):
    """
    Build variable-length pointers and LUT for a specific level.
    """
    # Count pointer frequencies
    pointer_counts = count_frequencies(nodes)
    
    # Sort pointers by frequency
    sorted_pointers = sort_by_frequency(pointer_counts)
    
    lut = {}
    direct_pointers = []
    
    for pointer, count in sorted_pointers:
        # Calculate costs
        # Direct cost: rounded length (32 or 48) * count
        direct_cost = calculate_direct_cost(pointer) * count
        # LUT cost: 64 * count
        lut_cost = 64 * count
        
        if lut_cost < direct_cost:
            # Add to LUT
            if pointer not in lut:
                lut[pointer] = len(lut)
            direct_pointers.append((pointer, 'lut', lut[pointer]))
        else:
            direct_pointers.append((pointer, 'direct', None))
            
    return lut, direct_pointers

def ray_traverse_tsvdag(ray, node, transform_stack, translation_stack):
    """
    Traverse TSVDAG node.
    """
    # 1. Compute intersection with node center planes
    intersection_mask = compute_intersection_mask(ray, node)
    
    # 2. Determine traversal order based on ray direction signs
    order = get_traversal_order(intersection_mask, ray.direction)
    
    # 3. For each intersected child
    for child_idx in order:
        if not is_child_intersected(intersection_mask, child_idx):
            continue
            
        # Get pointer for child
        pointer = get_child_pointer(node, child_idx)
        
        # Decode pointer
        transform_id = decode_transform_id(pointer)
        translation = decode_translation(pointer)
        address = decode_address(pointer)
        
        # Handle Transform (Symmetry/Permutation)
        current_transform = transform_stack.top()
        new_transform = apply_transform(current_transform, transform_id)
        transform_stack.push(new_transform)
        
        # Handle Translation
        if translation is not None:
            # Update translation stack
            accumulated_translation = translation_stack.top().accumulated + translation
            # Adjust tmin/tmax based on translation
            new_tmin, new_tmax = adjust_bounds(ray, node, translation)
            translation_stack.push({
                'accumulated': accumulated_translation,
                'tmin': new_tmin,
                'tmax': new_tmax
            })
            
            # Translate child node center
            translated_node_center = node.center + accumulated_translation
            
            # Recurse with translated bounds
            ray_traverse_tsvdag(ray, get_node(address), transform_stack, translation_stack)
            
            translation_stack.pop()
        else:
            # Recurse normally
            ray_traverse_tsvdag(ray, get_node(address), transform_stack, translation_stack)
            
        transform_stack.pop()
```

### Parameters
*   **Translation Search Limit:** Subtrees up to $16^3$ voxels ($L=4$) [S1].
*   **Translation Range:** $[-2^L+1, +2^L-1]^3$ [S1].
*   **Translation Mask:** 3-bit mask for zero/non-zero axes [S1].
*   **Translation Bits:** 5 bits per axis with non-zero translation [S1].
*   **Pointer Sizes:** 16, 32, or 48 bits [S1].
*   **LUT Entry Size:** 64 bits [S1].
*   **Base Pointer Space:** 48 bits [S1].

### Edge Cases and Pitfalls
*   **Complex Dependencies:** Destructive translations can create multi-level indirection in the dependency graph. Due to SVDAG encoding limitations, subtrees in these cases must be duplicated [S1].
*   **Variable-Length Overhead:** GPU memory systems are not optimized for variable-length data, causing render overhead [S1].
*   **Registry Pressure:** Maintaining the additional translation stack increases GPU registry pressure or causes spilling [S1].
*   **Search Complexity:** Naive translation search is $O(NV^2)$, which is impractical. The optimized $O(NV)$ search is required [S1].
*   **Address Space Gaps:** Variable-length pointers index the address space at 16-bit intervals, leaving many indices unreferenced, reducing the effectiveness of compact encodings [S1].

## Subtopics

### [Arbitrary Permutation Transformations for SVDAG](8412-arbitrary-permutation-transformations-for-svdag.md)
Extends SSVDAG by supporting all $8!$ permutation transformations of child octants to maximize geometric redundancy removal. It differs from SSVDAG, which only supports mirror symmetries, by allowing arbitrary reordering of the eight child pointers, thereby capturing rotational symmetries and complex spatial redundancies that mirror operations alone miss.

## Operating characteristics
*   **Compression:** TSVDAG achieves 20% to 30% (up to 35%) memory improvement over SSVDAG [S1]. SVO memory usage is 516% to 1456% larger than TSVDAG across tested scenes (Bistro, Bunny, Crown, Citadel, Lucy, Dragon at 64k³) [S1]. A dense voxel grid for these scenes would require 35TB [S1].
*   **Encoding Overhead:** Pointer LUT + Huffman coding reduces size by 4% to 8% compared to Pointer LUT alone [S1]. However, enabling LUT and Huffman increases frame time by ~12% [S1].
*   **Translation Overhead:** Adding translation transformations increases frame time by ~30% [S1].
*   **Render Performance:** SVDAG with fixed 32-bit pointer encoding outperforms variable-length methods by ~10% [S1]. TSVDAG was evaluated on an NVIDIA RTX4070 GPU [S1].
*   **Preprocessing:** Fast translation search takes a couple of hours for a 64k³ scene on CPU, significantly slower than hierarchical symmetry/permutation search (<1 minute) [S1].
*   **Strengths:** Highest compression ratio among compared methods (SVO, SVDAG, SSVDAG).
*   **Limitations:** High preprocessing cost; increased rendering overhead due to variable-length pointers and translation stack; complex dependency handling requires duplication.

## Background
TSVDAG was introduced in the paper "Transform-Aware Sparse Voxel Directed Acyclic Graphs" [S1]. It builds upon SVDAG [Kämpe et al. 2013] and SSVDAG [Villanueva et al. 2016]. The method was evaluated on scenes including Bistro, Bunny, Crown, Citadel, Lucy, and Dragon at 64k³ resolution, rendered using a CUDA-based path tracer on an NVIDIA RTX4070 [S1].

## Sources
[S1] Transform-Aware Sparse Voxel Directed Acyclic Graphs, https://publications.graphics.tudelft.nl/rails/active_storage/blobs/redirect/eyJfcmFpbHMiOnsibWVzc2FnZSI6IkJBaHBBdjBUIiwiZXhwIjpudWxsLCJwdXIiOiJibG9iX2lkIn19--3e9c1bfab97521b420d59c348403286749dcf70d/I3D_authors_version.pdf, fetched 2026-11-01
