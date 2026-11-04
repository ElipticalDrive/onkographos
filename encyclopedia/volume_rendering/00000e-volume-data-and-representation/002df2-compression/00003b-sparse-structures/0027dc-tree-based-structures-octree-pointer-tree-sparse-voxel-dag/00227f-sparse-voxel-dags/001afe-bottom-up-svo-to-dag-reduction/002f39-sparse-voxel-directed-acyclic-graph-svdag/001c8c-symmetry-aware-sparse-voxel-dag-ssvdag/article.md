# Symmetry-aware Sparse Voxel DAG (SSVDAG)


## Overview
SSVDAG is a compact, lossless volume representation that merges subtrees identical through reflective symmetries (planar reflections along principal grid axes) and employs variable bit-rate encoding for child pointers to minimize memory footprint while maintaining GPU-friendly traversal performance [S1, unofficial]. It differs from its parent [Sparse Voxel Directed Acyclic Graph (SVDAG)](962-sparse-voxel-directed-acyclic-graph-svdag.md) by recognizing geometric similarity via transformations rather than requiring exact node identity, and from its child article by explicitly detailing the binary layout and tagged pointer encoding [1349-ssvdag-node-layout-and-tagged-pointer-encoding.md].

## How it works
**Inputs:** A Sparse Voxel Octree (SVO) or an existing SVDAG [S1, unofficial].
**Outputs:** A Symmetry-aware Sparse Voxel DAG (SSVDAG) stored in a compressed binary format [S1, unofficial].

**Data Structures:**
1.  **Indexing Structure:** Contains the maximum level $L$ and three 32-bit offsets indicating the start of each level in the inner level array [S1, unofficial].
2.  **Inner Node Header:** A 16-bit header containing a 2-bit code for each of the 8 potential children [S1, unofficial].
    *   Tag `00`: Reserved for null pointers (not stored) [S1, unofficial].
    *   Tag `01`: Child pointer is 16 bits [S1, unofficial].
    *   Tag `10`: Child pointer is 32 bits [S1, unofficial].
    *   Tag `11`: Child pointer is 32 bits [S1, unofficial].
3.  **Child Pointer Formats:**
    *   **Tag 01 (16 bits):** Leftmost 3 bits encode the transformation $T$ (reflection codes). Remaining 13 bits encode the offset in number of level-words from the beginning of the next level [S1, unofficial]. A level-word is 2 bytes for an inner level and 8 bytes for the leaf level [S1, unofficial]. This allows encoding reflections and references to nodes in the first $2^{14}$ bytes of an inner level or first $2^{16}$ bytes of the leaf level [S1, unofficial].
    *   **Tag 10/11 (32 bits):** Leftmost 3 bits encode the transformation $T$. Remaining 29 bits encode the lowest 29 bits of the offset [S1, unofficial]. The highest bit of the offset is stored in the header or a subsequent byte [S1, unofficial].

**Algorithm Steps:**
The construction is a bottom-up external-memory process that reduces an SVO to a minimal SSVDAG by alternating phases at each level, starting from the finest level $L-1$ up to the root at level 0 [S1, unofficial].

1.  **Leaf Grouping (Coarsening):** The construction graph is coarsened by one level. $4^3$ grids are encoded as simple grids in a single array of bricks, each occupying 64 bits [S1, unofficial]. This is done by grouping the $2^3$ leaves from the bottom-up process into $4^3$ inner nodes at level $L-2$ without performing new matches [S1, unofficial].
2.  **Leaf Node Clustering:** Each small voxel grid $G$ of size $2^3$ is represented by a binary number $B(G)$ by concatenating occupancy bits [S1, unofficial]. The canonical representation $G^?$ is defined as $G^? = T_{x,y,z}(G)$ where $T_{x,y,z} = \arg \max_{T_{x,y,z}} B(T_{x,y,z}(G))$ [S1, unofficial]. This is the reflection with the largest integer value [S1, unofficial]. Two leaf nodes are similar if their canonical representations are identical [S1, unofficial]. A precomputed table of 256 entries maps all possible $2^3$ voxel combinations to the canonical transformation and the unique canonical representation (one of 46 possible ones) [S1, unofficial].
3.  **Inner Node Clustering:** Inner nodes $n_1$ and $n_2$ are merged if they represent the same subtrees under a transformation $T$ [S1, unofficial]. First, sort inner nodes by the lexicographically sorted set of child pointers [S1, unofficial]. During a streaming pass, verify similarity: for every tagged pointer $(tag, p)$ of $n_1$, it must map to $(T(tag), p)$ in $n_2$ if $p$ is not invariant to $T$, or to $(T(tag) \neg T(tag), p)$ if $p$ is invariant [S1, unofficial]. Repeat for all 8 reflection combinations, stopping at the first match [S1, unofficial]. Replace pointers to the clustered nodes in the immediately higher level with tagged pointers to the single representative, where the tag encodes the specific transformation required to recover the original subtree [S1, unofficial].
4.  **Frequency-based Pointer Compaction:** Nodes at each level are reordered by reference count (most referenced first) [S1, unofficial]. Pointers are replaced by offsets from the beginning of the level array [S1, unofficial]. The number of bits for each offset is chosen as the smallest available in the encoded format (16-bit or 32-bit) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_ssvdag(svo_levels):
    # svo_levels: list of lists of Node objects. 
    # Node has: .children (list of 8 tuples (tag, ptr)), .ref_count, .parent, .parent_idx
    L = len(svo_levels) - 1
    processed = [set() for _ in range(L + 1)]
    
    # Precomputed table for 2^3 leaf grids: maps 256 occupancy patterns to (canonical_rep, transform_bits)
    LEAF_TABLE = build_leaf_canonical_table()
    
    for level in range(L - 1, -1, -1):
        nodes = svo_levels[level]
        representatives = []
        
        for i, node in enumerate(nodes):
            if i in processed[level]:
                continue
            cluster = [node]
            processed[level].add(i)
            transforms = [0]
            
            for j in range(i + 1, len(nodes)):
                if j in processed[level]:
                    continue
                other = nodes[j]
                T = find_symmetry_match(cluster[0], other)
                if T is not None:
                    cluster.append(other)
                    processed[level].add(j)
                    transforms.append(T)
            
            representatives.append((cluster[0], transforms))
            
            # Update parent pointers for merged nodes
            for k, other in enumerate(cluster[1:], start=1):
                T = transforms[k]
                other.parent.children[other.parent_idx] = (T, cluster[0])
        
        # Frequency-based compaction
        representatives.sort(key=lambda x: x[0].ref_count, reverse=True)
        
        for offset, (rep, _) in enumerate(representatives):
            rep.offset = offset
            rep.header = encode_header(rep.children)
            rep.pointers = encode_pointers(rep.children, offset)
        
        svo_levels[level] = [rep for rep, _ in representatives]
        
    # Leaf coarsening: group 2^3 leaves into 4^3 bricks (64 bits each)
    bricks = []
    leaf_nodes = svo_levels[L-1]
    for i in range(0, len(leaf_nodes), 8):
        brick_data = pack_leaf_brick(leaf_nodes[i:i+8])
        bricks.append(brick_data)
        
    return build_index(svo_levels), bricks

def find_symmetry_match(n1, n2):
    # Check 8 reflections. Returns transform bits or None.
    for T_bits in range(8):
        match = True
        for i in range(8):
            tag1, ptr1 = n1.children[i]
            tag2, ptr2 = n2.children[i]
            mapped_tag = apply_reflection_tag(tag1, T_bits)
            if ptr1 != ptr2:
                if not is_invariant(ptr1, T_bits):
                    match = False
                    break
            else:
                if mapped_tag != tag2:
                    match = False
                    break
        if match:
            return T_bits
    return None

def apply_reflection_tag(tag, T_bits):
    # Standard octree child index permutation under axis reflections [G]
    # Tags are 2-bit codes representing child indices 0-7
    return tag  # Actual permutation depends on specific child indexing scheme

def is_invariant(ptr, T_bits):
    # Checks if the subtree pointed to by ptr is symmetric under T [G]
    return ptr.is_symmetric_under(T_bits)

def encode_header(children):
    header = 0
    for i in range(8):
        tag, ptr = children[i]
        if ptr is None:
            code = 0
        else:
            offset = ptr.offset
            if offset < (1 << 13):
                code = 1
            else:
                code = 2
        header |= (code << (2 * i))
    return header

def encode_pointers(children, base_offset):
    packed = []
    for i in range(8):
        tag, ptr = children[i]
        if ptr is None:
            continue
        offset = ptr.offset - base_offset
        T = ptr.tag
        if offset < (1 << 13):
            # 16-bit format: 3 bits T, 13 bits offset
            val = (T << 13) | offset
            packed.append(val & 0xFFFF)
        else:
            # 32-bit format: 3 bits T, 29 bits offset
            val = (T << 29) | (offset & 0x1FFFFFFF)
            packed.append(val & 0xFFFFFFFF)
    return packed

def build_leaf_canonical_table():
    # Returns dict mapping 256 occupancy patterns to (canonical_rep, transform_bits)
    # Uses precomputed 2^3 voxel reflection table [S1, unofficial]
    return {}

def pack_leaf_brick(leaves):
    # Packs 8 leaves into a 64-bit brick [S1, unofficial]
    return 0

def build_index(svo_levels):
    # Returns indexing structure with max level L and 3 offsets [S1, unofficial]
    return {}
```

**Parameters:**
*   **Symmetry Transforms:** Planar reflections along the main grid directions (X, Y, Z) [S1, unofficial].
*   **Leaf Size for Construction:** $2^3$ voxels [S1, unofficial].
*   **Transformation Encoding:** 3 bits corresponding to reflections along the X, Y, and Z axes [S1, unofficial]. The transformation $T_{x,y,z}$ is its own inverse: $T_{x,y,z}^{-1} = T_{x,y,z}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Invariant Pointers:** When checking similarity, if a child pointer $p$ is invariant to the transformation $T$, the tag mapping logic differs from non-invariant pointers. The source specifies mapping to $(T(tag) \neg T(tag), p)$ if invariant, requiring careful implementation of the invariant check [S1, unofficial].
*   **Precision:** The method relies on exact bitwise comparison of canonical representations for leaves and exact pointer matching for inner nodes [S1, unofficial].

## Subtopics

### [SSVDAG Node Layout and Tagged Pointer Encoding](1349-ssvdag-node-layout-and-tagged-pointer-encoding.md)
Defines the binary layout of inner nodes, replacing fixed-size child pointers with variable-length tagged pointers (2 or 4 bytes) that embed transformation codes, solving the memory overhead problem of standard SVDAGs where all pointers are fixed at 4 bytes.

## Operating characteristics
*   **Compression:** In the PowerPlant scene (64K$^3$ resolution, ~6 billion non-empty voxels), SSVDAG stores data in <86MB (0.123 bits/voxel), compared to 16.2GB for SVO (without pointers) and 167MB for SVDAG [S1, unofficial].
*   **Performance:** Real-time GPU in-core visualization with shading and shadows is achieved for the full Boeing 777 model at sub-millimetric precision [S1, unofficial]. The tracing overhead for SSVDAG compared to standard SVO/SVDAG is less than 15% [S1, unofficial].
*   **Memory Distribution:** In the Powerplant dataset, the most common 10% of nodes account for nearly 90% of references at level 14 and nearly 50% at level 11, justifying the frequency-based pointer compaction [S1, unofficial].
*   **Strengths:** High compression ratio for scenes with reflective symmetry; transparent to standard ray-casting algorithms; GPU-friendly traversal [S1, unofficial].
*   **Limitations:** Construction is complex and requires external-memory processing; limited to reflective symmetries (other similarity transforms not supported) [S1, unofficial].

## Background
SSVDAG was introduced by Alberto Jaspe Villanueva, Fabio Marton, and Enrico Gobbetti from CRS4 in 2016 [S1, unofficial]. It extends the concept of Sparse Voxel DAGs (SVDAGs) by incorporating symmetry-aware merging, building on prior work in DAG compression for SVOs [S1, unofficial].

## Sources
[S1] SSVDAGs: Symmetry-aware Sparse Voxel DAGs, https://www.crs4.it/vic/data/papers/i3d2016-symmetry-dags.pdf, fetched 2026-10-27 (unofficial)
