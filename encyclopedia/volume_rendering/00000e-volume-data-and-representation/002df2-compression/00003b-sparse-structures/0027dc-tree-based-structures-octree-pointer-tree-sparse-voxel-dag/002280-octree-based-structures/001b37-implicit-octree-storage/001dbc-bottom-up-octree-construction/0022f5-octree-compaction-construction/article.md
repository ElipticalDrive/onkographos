# Octree Compaction & Construction

Also called: Binvox Octree Compaction, Compact Octree Layout

## Overview
This method compacts a logically built, pointer-based octree into a contiguous array layout to accelerate traversal [S1, unofficial]. It differs from sibling methods by focusing on physical memory layout and compact relative/far pointer encoding rather than bit-stream storage or bounding volume acceleration alone [S1, S3, unofficial].

## How it works
The method consists of two phases: building the logical tree from voxel data, and compacting that tree into a contiguous array [S1, unofficial].

### Inputs and Outputs
- **Input**: A 3D scalar array (voxels) representing the volume data [S1, unofficial].
- **Output**: A contiguous array of compacted nodes (`CNode*`) and a root node pointer referencing the start of this array [S1, unofficial].

### Data Structures
The logical tree uses nodes with the following fields:
- `type`: Enum indicating node state (`NT_FILLED` for solid leaves, `NT_PARTIAL` for internal nodes) [S1, unofficial].
- `children`: Array of 8 pointers to child nodes (or `NULL`) [S1, unofficial].
- `farAdr`: Offset for far pointers, initialized to 0 [S1, unofficial].
- `childAdr`: Offset for child pointers, initialized to 0 [S1, unofficial].
- `nid`: Unique node identifier [S1, unofficial].

The compacted node (`CNode`) layout in memory:
- `valid_mask`: 8-bit integer, bit $i$ set if child $i$ exists [S1, unofficial].
- `leaf_mask`: 8-bit integer, bit $i$ set if child $i$ is `NT_FILLED` [S1, unofficial].
- `child_ptr`: 16-bit integer. If the most significant bit (bit 15) is set, it is a far pointer; otherwise, it is a short relative offset [S1, unofficial].
- `far_ptr`: 16-bit integer storing the actual child pointer when `child_ptr` indicates a far reference [G].

### Parameters
- `_shortPtrTop`: Threshold value for short vs. far pointer encoding. If a child's relative offset exceeds this value, a far pointer is allocated. Exact numeric value not specified in the sources [S1, unofficial].

### Algorithm Steps

#### Phase 1: Logical Tree Construction (`readFromBinvox_rec`)
1. **Base Case**: If `width == 1`, check the voxel at index `y + z*voxels_width + x*voxels_width*voxels_width` [S1, unofficial].
   - If non-zero, create an `NT_FILLED` leaf node and return it [S1, unofficial].
   - If zero, delete the node and return `NULL` [S1, unofficial].
2. **Recursive Step**:
   - Calculate `child_width = width / 2` [S1, unofficial].
   - Recursively build 8 children in the following order (octants):
     1. $(x, y, z)$
     2. $(x, y, z + child\_width)$
     3. $(x, y + child\_width, z)$
     4. $(x, y + child\_width, z + child\_width)$
     5. $(x + child\_width, y, z)$
     6. $(x + child\_width, y, z + child\_width)$
     7. $(x + child\_width, y + child\_width, z)$
     8. $(x + child\_width, y + child\_width, z + child\_width)$ [S1, unofficial].
3. **Consolidation**:
   - Check `all_empty` (all children are `NULL`) and `all_filled` (all children are `NT_FILLED`) [S1, unofficial].
   - If `all_empty`: Delete the node and return `NULL` [S1, unofficial].
   - If `all_filled`: Delete all children, set `node->type = NT_FILLED`, and return the node [S1, unofficial].
   - Otherwise: Set `node->type = NT_PARTIAL` [S1, unofficial].
   - Assign `node->nid = nextNid++` and return the node [S1, unofficial].

#### Phase 2: Compaction (`compactTree`)
1. Initialize `rootNode->farAdr = 0` and `rootNode->childAdr = 0` [S1, unofficial].
2. Call `compactAllocate(rootNode, 1)` to compute the total `dataSize` required [S1, unofficial].
3. Allocate memory: `data = (CNode*)malloc(dataSize * sizeof(CNode))` [S1, unofficial].
4. Call `compactNode(rootNode, dataSize)` to fill the array [S1, unofficial].

##### Subroutine: `compactAllocate(node, pos)`
Computes the memory footprint of the subtree rooted at `node` [S1, unofficial].
1. Initialize `pos` [S1, unofficial].
2. Iterate 8 children:
   - Reset `child->farAdr = 0` and `child->childAdr = 0` [S1, unofficial].
   - If `child` exists and `child->type == NT_PARTIAL`, recursively call `pos = compactAllocate(child, pos)` [S1, unofficial].
3. Second loop (8 children):
   - If `child` exists and `child->childAdr` is set:
     - Compute `gchildAdrMax = pos - child->childAdr + 16` [S1, unofficial].
     - If `gchildAdrMax > _shortPtrTop`, set `child->farAdr = pos` and increment `pos` [S1, unofficial].
4. Third loop (8 children):
   - If `child` exists and `child->type == NT_PARTIAL`, set `node->childAdr = pos` and increment `pos` [S1, unofficial].
5. Return `pos` [S1, unofficial].

##### Subroutine: `compactNode(node, dataSize)`
Fills the compacted array from the end (`dataSize`) backwards [S1, unofficial].
1. Iterate 8 children:
   - If `child` exists and `child->type == NT_PARTIAL`, recursively call `compactNode(child, childAdr)` and decrement `childAdr` [S1, unofficial].
   - Assert `pos <= dataSize` [S1, unofficial].
2. Set `cnode = &data[dataSize - pos]` [S1, unofficial].
3. Initialize `cnode->valid_mask = 0` and `cnode->leaf_mask = 0` [S1, unofficial].
4. Iterate 8 children:
   - If `node->children[i]` exists, set `cnode->valid_mask |= (1 << i)` [S1, unofficial].
   - If `node->children[i]->type == NT_FILLED`, set `cnode->leaf_mask |= (1 << i)` [S1, unofficial].
5. Compute `childPtr = pos - node->childAdr` [S1, unofficial].
6. If `node->farAdr` is set:
   - Compute `farPtr = pos - node->farAdr` [S1, unofficial].
   - Assert `farPtr < _shortPtrTop` [S1, unofficial].
   - Set `farPtrEntry->far_ptr = childPtr` [S1, unofficial].
   - Set `cnode->child_ptr = (1 << 15) | farPtr` [S1, unofficial].
7. Else:
   - Set `cnode->child_ptr = childPtr` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Node Types
NT_FILLED = 1
NT_PARTIAL = 2

class LogicalNode:
    def __init__(self, type_, children=None):
        self.type = type_
        self.children = children if children else [None] * 8
        self.farAdr = 0
        self.childAdr = 0
        self.nid = 0

class CompactNode:
    def __init__(self):
        self.valid_mask = 0
        self.leaf_mask = 0
        self.child_ptr = 0
        self.far_ptr = 0  # Stores actual child pointer for far references [G]

def read_from_binvox_rec(x, y, z, width, voxels, voxels_width, next_nid_ref):
    """Recursively builds logical octree from voxel array [S1, unofficial]."""
    if width == 1:
        idx = y + z * voxels_width + x * voxels_width * voxels_width
        if voxels[idx] != 0:
            return LogicalNode(NT_FILLED)
        else:
            return None
    
    child_width = width // 2
    children = []
    
    # Octant offsets in order [S1, unofficial]
    offsets = [
        (0, 0, 0), (0, 0, 1), (0, 1, 0), (0, 1, 1),
        (1, 0, 0), (1, 0, 1), (1, 1, 0), (1, 1, 1)
    ]
    
    for dx, dy, dz in offsets:
        child = read_from_binvox_rec(
            x + dx * child_width,
            y + dy * child_width,
            z + dz * child_width,
            child_width,
            voxels, voxels_width,
            next_nid_ref
        )
        children.append(child)
        
    all_empty = all(c is None for c in children)
    all_filled = all(c is not None and c.type == NT_FILLED for c in children)
    
    if all_empty:
        return None
    elif all_filled:
        return LogicalNode(NT_FILLED)
    else:
        node = LogicalNode(NT_PARTIAL, children)
        node.nid = next_nid_ref[0]
        next_nid_ref[0] += 1
        return node

def compact_allocate(node, pos, short_ptr_top):
    """Computes memory size for subtree [S1, unofficial]."""
    if node is None or node.type == NT_FILLED:
        return pos
        
    # First pass: allocate space for partial children
    for i in range(8):
        child = node.children[i]
        if child is not None:
            child.farAdr = 0
            child.childAdr = 0
            if child.type == NT_PARTIAL:
                pos = compact_allocate(child, pos, short_ptr_top)
                
    # Second pass: handle far pointers for grandchildren
    for i in range(8):
        child = node.children[i]
        if child is not None and child.childAdr != 0:
            gchild_adr_max = pos - child.childAdr + 16
            if gchild_adr_max > short_ptr_top:
                child.farAdr = pos
                pos += 1
                
    # Third pass: allocate space for direct children
    for i in range(8):
        child = node.children[i]
        if child is not None and child.type == NT_PARTIAL:
            node.childAdr = pos
            pos += 1
            
    return pos

def compact_node(node, pos, data, short_ptr_top):
    """Fills compacted array from end [S1, unofficial]."""
    if node is None or node.type == NT_FILLED:
        return pos
        
    # Recursively compact children first
    for i in range(8):
        child = node.children[i]
        if child is not None and child.type == NT_PARTIAL:
            pos = compact_node(child, pos, data, short_ptr_top)
            
    assert pos <= len(data)
    
    cnode = data[len(data) - pos]
    cnode.valid_mask = 0
    cnode.leaf_mask = 0
    
    for i in range(8):
        child = node.children[i]
        if child is not None:
            cnode.valid_mask |= (1 << i)
            if child.type == NT_FILLED:
                cnode.leaf_mask |= (1 << i)
                
    child_ptr = pos - node.childAdr
    
    if node.farAdr != 0:
        far_ptr = pos - node.farAdr
        assert far_ptr < short_ptr_top
        # farPtrEntry points to data[node.farAdr] [S1, unofficial]
        data[node.farAdr].far_ptr = child_ptr
        cnode.child_ptr = (1 << 15) | far_ptr
    else:
        cnode.child_ptr = child_ptr
        
    return pos

def compact_tree(root_node, voxels, voxels_width, short_ptr_top=32768):
    """Main entry point for compaction [S1, unofficial]."""
    next_nid = [0]
    rootNode = read_from_binvox_rec(0, 0, 0, 1024, voxels, voxels_width, next_nid)
    
    if rootNode is None:
        return None, None
        
    # Compute exact size [S1, unofficial]
    dataSize = compact_allocate(rootNode, 1, short_ptr_top)
    
    if dataSize <= 0:
        return None, None
        
    # Initialize data array
    data = [CompactNode() for _ in range(dataSize)]
    
    compact_node(rootNode, dataSize, data, short_ptr_top)
    
    return rootNode, data
```

### Edge Cases and Pitfalls
- **Precision**: The relative pointer arithmetic relies on `pos` and `childAdr` being consistent. Off-by-one errors in `pos` initialization or incrementation will corrupt the tree structure [S1, unofficial].
- **Boundaries**: The `_shortPtrTop` threshold must be chosen carefully. If set too low, excessive far pointers are allocated, increasing memory usage. If set too high, relative offsets may overflow the 15-bit field [S1, unofficial].
- **Degenerate Input**: Empty volumes result in `NULL` roots, which must be handled by the caller before compaction [S1, unofficial].

## Subtopics

### Min/Max Tree Construction for Ray Tracing
This variant extends the compacted octree with a min/max tree to accelerate ray tracing by culling empty space. It differs from the base compaction by computing bounding volumes for nodes based on forward-adjacent voxels, not just the node's own children [S3, unofficial]. It is chosen when ray tracing performance is critical and the scene has large empty regions. The storage overhead is approximately 22% additional footprint on top of the raw scalar octree-compressed data [S3, unofficial]. Construction involves computing min/max pairs for leaf nodes based on cell corner values and forward neighbors, then recursively computing parent min/max pairs from children [S3, unofficial]. Leaf nodes may omit min/max pairs to save storage, forcing neighbor-finding at runtime [S3, unofficial].

## Operating characteristics
- **Time Complexity**: Construction is $O(N)$ where $N$ is the number of logical nodes, as each node is visited a constant number of times during allocation and compaction [S1, unofficial].
- **Memory Complexity**: The compacted layout reduces pointer overhead significantly compared to a standard pointer-based tree. The exact compression ratio depends on the tree balance and the `_shortPtrTop` threshold [S1, unofficial].
- **Performance**: Provides speedups of 25% to 45% when combined with smaller kernels in splatting methods by reducing the cost of handling invisible material [S2].
- **Strengths**: Cache-friendly layout, reduced memory footprint, fast traversal due to contiguous memory [S1, unofficial].
- **Limitations**: Requires two-pass construction (logical build then compact). Far pointers add complexity to traversal logic [S1, unofficial].

## Background
The method originates from the svo-workbench project [S1, unofficial]. It was designed to address the high cost of handling invisible material in splatting and raycasting by using compacted octree encoding alongside a simultaneous occlusion map/volume traversal scheme [S2]. The approach builds on earlier volume rendering evaluation work that highlighted the need for efficient sparse structures [S2].

## Sources
[S1] skyfex/svo-workbench, https://github.com/skyfex/svo-workbench, fetched 2026-10-26 (unofficial)
[S2] A Practical Evaluation of Popular Volume Rendering ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/PracticalEvaluationofVolumeRenderingAlgorithms.pdf, fetched 2026-10-29
[S3] Interactive isosurface ray tracing of large octree volumes, https://www.researchgate.net/publication/310054812_Interactive_isosurface_ray_tracing_of_large_octree_volumes, fetched 2026-10-29 (unofficial)
