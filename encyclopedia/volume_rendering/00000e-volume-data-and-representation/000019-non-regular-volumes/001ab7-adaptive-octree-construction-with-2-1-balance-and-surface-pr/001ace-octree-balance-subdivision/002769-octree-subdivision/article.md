# Octree Subdivision

Also called: LiNode::subdivide

## Overview
Octree Subdivision splits an octree node into eight children by halving its bounding box along each axis and redistributing associated data elements like surfels [S1, unofficial]. It provides the local geometric refinement step required for adaptive sparse voxel representations [S1, unofficial].

## How it works
### Inputs and Outputs
*   **Input:** A parent node defined by its bounding box corners $\mathbf{min}$ and $\mathbf{max}$, and a collection of data elements (e.g., surfels) associated with that node [S1, unofficial].
*   **Output:** An array of eight child nodes, each covering a sub-region of the parent's volume, with the data elements redistributed into the appropriate children [S1, unofficial].

### Data Structures
Each node stores $\mathbf{min}$ and $\mathbf{max}$ as 3D vectors, and a `children` array of size 8 [S1, unofficial]. In GPU implementations, the `children` pointer may be packed such that its most significant bit indicates whether the node is flagged for subdivision [S3, unofficial].

### Steps
1.  **Calculate half-extent:** Compute $\mathbf{size} = (\mathbf{max} - \mathbf{min}) \times 0.5$ [S1, unofficial].
2.  **Validate:** If $\|\mathbf{size}\| < 0.00000001$, print an error and exit [S1, unofficial].
3.  **Create children:** Loop $i$ from 0 to 7. Compute $x = i \bmod 2$, $y = \lfloor i / 2 \rfloor \bmod 2$, $z = \lfloor i / 4 \rfloor$ [S1, unofficial]. Compute child minimum $\mathbf{min}_i = \mathbf{min} + \mathbf{size} \times (x, y, z)$ and maximum $\mathbf{max}_i = \mathbf{min}_i + \mathbf{size}$ [S1, unofficial]. Instantiate child node $i$ with $(\mathbf{min}_i, \mathbf{max}_i)$ [S1, unofficial].
4.  **Redistribute data:** Iterate through each data element in the parent node, determine which of the eight child bounding boxes contains it, and append it to that child's list [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class vector3:
    def __init__(self, x, y, z): self.x, self.y, self.z = x, y, z
    def __sub__(self, other): return vector3(self.x - other.x, self.y - other.y, self.z - other.z)
    def __mul__(self, scalar): return vector3(self.x * scalar, self.y * scalar, self.z * scalar)
    def __add__(self, other): return vector3(self.x + other.x, self.y + other.y, self.z + other.z)
    def length(self): return (self.x**2 + self.y**2 + self.z**2)**0.5

class LiNode:
    def __init__(self, min, max):
        self.min = min
        self.max = max
        self.children = []
        self.surfels = []

class Error(Exception): pass

def subdivide(parent_node):
    size = (parent_node.max - parent_node.min) * 0.5
    if size.length() < 0.00000001:
        raise Error("Node too small to subdivide")
    parent_node.children = []
    for i in range(8):
        x = i % 2
        y = (i // 2) % 2
        z = i // 4
        child_min = parent_node.min + size * vector3(x, y, z)
        child_max = child_min + size
        child_node = LiNode(min=child_min, max=child_max)
        parent_node.children.append(child_node)
    for surfel in parent_node.surfels:
        for child in parent_node.children:
            if child.min.x <= surfel.pos.x < child.max.x and \
               child.min.y <= surfel.pos.y < child.max.y and \
               child.min.z <= surfel.pos.z < child.max.z:
                child.surfels.append(surfel)
                break
```

### Parallel GPU Subdivision
This variant executes subdivision in three distinct phases using atomic operations [S3, unofficial].
1.  **Flagging:** Launch one thread per entry in the voxel-fragment list [S3, unofficial]. Each thread traverses the octree from the root to the target level and flags the destination node by setting the most significant bit of its children pointer [S3, unofficial].
2.  **Allocation:** Allocate a $2 \times 2 \times 2$ tile of subnodes for each flagged node within the octree pool, typically using a shared atomic counter to assign unique memory addresses [S3, unofficial].
3.  **Initialization:** Execute a separate pass where one thread handles each newly created node, initializing them primarily by setting child pointers to null [S3, unofficial].

### Parameters and Edge Cases
*   **Leaf Size:** Iterative subdivision stops when the node size in voxels is $\le$ the specified leaf-node size in each dimension [S2, unofficial].
*   **Precision:** The method uses a hardcoded threshold of $0.00000001$ for the half-extent length to prevent degenerate splits [S1, unofficial].
*   **Degenerate Input:** If volume dimensions or leaf sizes cause non-uniform splitting, the structure may resemble a quadtree or binary tree at certain levels [S2, unofficial].

## Subtopics

### Dimensional Splitting
A variant where subdivision is not uniform across all axes [S2, unofficial]. At each step, a dimension is halved only if its extent exceeds the target leaf size relative to the other dimensions [S2, unofficial]. For example, if a node extent is $(4, 2, 2)$, only the dimension with extent 4 is subdivided [S2, unofficial]. This results in anisotropic nodes and can cause the structure to resemble a quadtree or binary tree in certain levels [S2, unofficial]. Choose this when the volume has strong anisotropy or when memory efficiency for non-cubic data is prioritized over strict octree regularity [S2, unofficial].

### Node Subdivision
A parallel implementation detail where subdivision is driven by atomic operations on a voxel-fragment list rather than explicit geometric traversal [S3, unofficial]. It separates the concerns of identifying nodes to split (flagging), memory allocation (atomic counter), and initialization (null pointers) [S3, unofficial]. Choose this for high-throughput GPU-based volume construction where thousands of fragments need to be inserted simultaneously [S3, unofficial].

### Octree Subdivision for Isotopic Meshing
Uses a balanced octree to create an isotopic mesh of an implicit surface, ensuring isotopy between the surface and the octree approximation [S4]. The subdivision process continues until for each leaf $C$, $0 \notin \Box F(C) \quad \vee \quad \langle \Box \nabla F(C), \Box \nabla F(C) \rangle > 0$ [S4]. After subdivision, the octree must be balanced [S4].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Allows adaptive resolution and captures fine details without wasting memory on empty space [S1, unofficial].
*   **Limitations:** Frequent subdivision increases memory overhead from node headers [S3, unofficial]. Anisotropic splitting complicates traversal logic [S2, unofficial].
*   **When to use:** When volume data has varying frequency content or when the bounding box is large but sparse [S2, unofficial].

## Background
The method originates from adaptive sparse voxel octree implementations [S1, unofficial]. The geometric splitting approach is documented in the Mr-Voxel repository [S1, unofficial]. The parallel GPU variant using atomic counters and fragment lists is detailed in the OpenGL Insights book on sparse voxelization [S3, unofficial]. Dimensional splitting for anisotropic volumes is discussed in the context of web-based volume rendering of biomedical data [S2, unofficial]. The isotopic meshing variant ensures topological correctness for implicit surfaces [S4].

## Sources
[S1] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S3] PDF K14190_text - Icare3D, https://icare3d.org/research/OpenGLInsights-SparseVoxelization.pdf, fetched 2026-10-29 (unofficial)
[S4] Isotopic approximation of implicit curves and surfaces, https://doi.org/10.1145/1057432.1057465, fetched 2026-10-29
