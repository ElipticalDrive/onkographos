# Octree Subdivision

Also called: LiNode::subdivide

## Overview
Octree Subdivision splits an octree node's bounding box in half along each axis to produce eight children, redistributing existing surfels [S1, unofficial]. It differs from uniform grid approaches by enabling variable granularity through adaptive refinement [S1, unofficial].

## How it works
**Inputs**: Parent node bounding box $[\mathbf{min}, \mathbf{max}]$ [S1, unofficial].
**Outputs**: Eight child nodes $C_0 \dots C_7$ with sub-bounding boxes [S1, unofficial].
**Data Structures**: Node stores $\mathbf{min}$ and $\mathbf{max}$ as 3D vectors, and an array of 8 child pointers [S1, unofficial].

1. Calculate half-size vector $\mathbf{s} = (\mathbf{max} - \mathbf{min}) \times 0.5$ [S1, unofficial].
2. Validate: if $\|\mathbf{s}\| < 10^{-8}$, print error and exit [S1, unofficial].
3. Loop $i$ from 0 to 7: [S1, unofficial].
   a. Compute bit components: $x = i \bmod 2$, $y = \lfloor i/2 \rfloor \bmod 2$, $z = \lfloor i/4 \rfloor$ [S1, unofficial].
   b. Compute child minimum: $\mathbf{min}_i = \mathbf{min} + \mathbf{s} \cdot (x, y, z)$ [S1, unofficial].
   c. Compute child maximum: $\mathbf{max}_i = \mathbf{min}_i + \mathbf{s}$ [S1, unofficial].
   d. Create child node $C_i$ with bounds $[\mathbf{min}_i, \mathbf{max}_i]$ [S1, unofficial].

**Isotopic variant**: Subdivide leaves until $0 \notin \Box F(C) \lor \langle \Box \nabla F(C), \Box \nabla F(C) \rangle > 0$, then balance the octree [S3].

**Pseudocode**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vec3:
    def __init__(self, x, y, z):
        self.x, self.y, self.z = x, y, z
    def __sub__(self, other): return Vec3(self.x - other.x, self.y - other.y, self.z - other.z)
    def __mul__(self, scalar): return Vec3(self.x * scalar, self.y * scalar, self.z * scalar)
    def __add__(self, other): return Vec3(self.x + other.x, self.y + other.y, self.z + other.z)
    def length(self): return (self.x**2 + self.y**2 + self.z**2)**0.5

def subdivide_node(parent_min, parent_max):
    half_size = (parent_max - parent_min) * 0.5
    if half_size.length() < 1e-8:
        raise ValueError("Node too small to subdivide")
    children = []
    for i in range(8):
        x = i % 2
        y = (i // 2) % 2
        z = i // 4
        child_min = parent_min + half_size * Vec3(x, y, z)
        child_max = child_min + half_size
        children.append({"min": child_min, "max": child_max})
    return children
```

**Parameters**: Epsilon threshold $10^{-8}$ [S1, unofficial].
**Edge cases**: Degenerate nodes with $\|\mathbf{s}\| < 10^{-8}$ trigger an immediate error exit rather than infinite recursion [S1, unofficial].

## Subtopics

### [Node Subdivision](9141-node-subdivision.md)
Node Subdivision is the core mechanism for refining an adaptive octree during top-down construction, transforming coarse nodes into finer 2×2×2 tiles to resolve high-frequency geometry or irradiance gradients. It differs from static subdivision by dynamically determining which nodes require refinement based on a pre-computed list of voxel fragments, and from uniform grid approaches by allowing variable granularity. This method enables efficient memory usage by only allocating child nodes where necessary, addressing the unwieldiness of uniform grids in large, complex scenes.

### Dimensional Splitting
Iterative subdivision continues until node size in voxels $\le$ leaf-node size in each dimension [S2, unofficial]. At each step, a dimension is halved or skipped based on the other two dimensions; e.g., extent $(4, 2, 2)$ subdivides only the dimension with extent 4 [S2, unofficial]. This can cause the octree to resemble a quadtree or binary tree at certain levels [S2, unofficial].

### Octree Subdivision for Isotopic Meshing
Uses a balanced octree to create an isotopic mesh of an implicit surface, ensuring topological equivalence between the surface and the octree approximation [S3]. Leaves are subdivided until the interval constraint $0 \notin \Box F(C) \lor \langle \Box \nabla F(C), \Box \nabla F(C) \rangle > 0$ is satisfied, followed by balancing [S3].

## Operating characteristics
Time complexity: Not given in the sources.
Memory complexity: Not given in the sources.
Strengths: Not given in the sources.
Limitations: Not given in the sources.
When to use: Not given in the sources.

## Background
The standard geometric subdivision logic originates from the Mr-Voxel project [S1, unofficial]. Dimensional splitting is described in the context of scalable web-based volume rendering for large biomedicinal datasets [S2, unofficial]. Isotopic subdivision for implicit surfaces is detailed in research on topological approximation of implicit curves and surfaces [S3].

## Sources
[S1] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
[S2] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S3] Isotopic approximation of implicit curves and surfaces, https://doi.org/10.1145/1057432.1057465, fetched 2026-10-29
