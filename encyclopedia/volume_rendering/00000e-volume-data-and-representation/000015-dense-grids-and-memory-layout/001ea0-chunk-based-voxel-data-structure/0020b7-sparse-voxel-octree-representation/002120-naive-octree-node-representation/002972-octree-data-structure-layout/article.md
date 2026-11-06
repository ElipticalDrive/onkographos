# Octree Data Structure Layout

Also called: Octree Class Layout

## Overview
This layout defines the memory organization of the root `Octree` class, which manages the hierarchical voxel data structure. It solves the problem of tracking the spatial bounds and storage locations of the octree nodes and associated voxel data. It differs from its parent [Naive Octree Node Representation](2445-naive-octree-node-representation.md) by encapsulating the global tree state (bounds, node list, data pointer) in a single class, rather than focusing on the internal fields of an individual node.

## How it works
**Inputs and Outputs:**
The class is instantiated to manage a voxel scene. It outputs a root node pointer and provides access to the node list and voxel data array.

**Data Structures:**
The `Octree` class contains the following fields:
- `vec3 min`: The minimum coordinate of the bounding box.
- `vec3 max`: The maximum coordinate of the bounding box.
- `vec3 size`: The dimensions of the bounding box.
- `size_t gridlength`: The resolution of the voxel grid along one axis.
- `size_t n_nodes`: The total count of nodes in the tree.
- `size_t n_data`: The total count of voxel data elements.
- `VoxelData* data`: A pointer to the array storing voxel attributes (e.g., color, normal).
- `std::vector<Node> nodes`: A contiguous vector storing all octree nodes.

**Initialization:**
The default constructor initializes the spatial bounds and resolution:
- `min` is set to $(0, 0, 0)$.
- `max` is set to $(1, 1, 1)$.
- `size` is set to $(1, 1, 1)$.
- `gridlength` is set to $128$ [S1, unofficial].

**Accessing the Root:**
The root node is stored at the end of the `nodes` vector. The method `getRootNode()` returns a pointer to this node:
$$ \text{getRootNode()} \rightarrow \&\text{nodes.back()} $$
This implies that nodes are appended to the vector during tree construction, with the root being the last element added [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Octree:
    def __init__(self):
        self.min = vec3(0.0, 0.0, 0.0)
        self.max = vec3(1.0, 1.0, 1.0)
        self.size = vec3(1.0, 1.0, 1.0)
        self.gridlength = 128
        self.n_nodes = 0
        self.n_data = 0
        self.data = None  # Pointer to VoxelData array
        self.nodes = []   # Vector of Node objects

    def get_root_node(self):
        # Returns the last element in the nodes vector
        if not self.nodes:
            return None
        return self.nodes[-1]
```

## Subtopics

### [Octree Voxel Storage](6922-octree-voxel-storage.md)
Groups adjacent voxels into larger cells to reduce memory usage for sparse regions.

## Operating characteristics
- **Memory:** The `nodes` vector provides contiguous memory access for traversal, but the `data` pointer is separate. The total memory is proportional to the number of nodes and data elements.
- **Access:** Root access is $O(1)$ via `back()`. Node iteration requires traversing the `nodes` vector.
- **Limitations:** The layout assumes a specific construction order where the root is last. This is a non-standard octree layout (typically root is first) and may complicate recursive traversal if not handled carefully [S1, unofficial].

## Background
This layout is derived from the `cpu_voxel_raycaster` project by Forceflow [S1, unofficial]. It represents a specific implementation choice for a CPU-based raycaster, prioritizing simple vector-based storage over pointer-heavy tree structures.

## Sources
[S1] cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
