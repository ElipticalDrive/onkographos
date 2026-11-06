# Voxel Data Layout

Also called: Input Data Format, Node Data Layout, Voxel Cube, Voxel Data Structure (Naive), World Tensor

## Overview
Voxel Data Layout refers to the fundamental memory organization of voxel data in a dense 3D grid, typically implemented as a contiguous array or tensor indexed by world coordinates $[x, y, z]$. It solves the problem of mapping 3D spatial positions to linear memory addresses for efficient access during rendering or processing. Unlike hierarchical structures (e.g., [Voxel Data Structure](3982-voxel-data-structure.md)) which use pointers or bitmasks to skip empty space, this layout stores every voxel in a fixed-size slot, making it simple to implement but potentially memory-inefficient for sparse scenes. It serves as the base layer for optimizations like [Voxel Material Packing](4907-voxel-material-packing.md) and [Voxel Renderer with Top-Level Grid and Bricks](8114-voxel-renderer-with-top-level-grid-and-bricks.md).

## How it works
### Inputs and Outputs
- **Input**: 3D dimensions ($W, H, D$) defining the volume bounds, and a stream of voxel data (type, color, opacity, etc.).
- **Output**: A linear memory buffer (array) where each voxel is stored at a specific index calculated from its $(x, y, z)$ coordinates.

### Data Structures
The layout is a contiguous 1D array. The specific field composition varies by implementation:

1.  **Naive Tensor Layout [S4, unofficial]**:
    -   Structure: A 3D tensor indexed by `world[x, y, z]`.
    -   Entry: A tensor containing 4 numbers: opacity $\sigma$ followed by an RGB triplet $(R, G, B)$.
    -   Total: 4 floating-point values per voxel (typically 32-bit floats, though not explicitly stated, implied by "numbers ranging from 0.0 to 1.0").

2.  **Fixed-Width Bit-Packed Layout [S1, unofficial]**:
    -   Storage: 32-bit storage buffer per voxel.
    -   Layout:
        -   Bits 0-7: Voxel type (8 bits).
        -   Bits 8-23: Color (16 bits).
        -   Bits 24-31: Auxiliary data (8 bits).
    -   Scaling: Voxels are scaled such that edges are $1/8$ unit in length [S1, unofficial].

3.  **Binary Index Layout [S2, unofficial]**:
    -   Format: Binary file composed of 8-bit or 16-bit indices.
    -   Order: Left-to-right, bottom-to-top, back-to-front.

4.  **Gaussian Lobe Layout [S5]**:
    -   Fields per voxel:
        1.  Average vector $D$ (direction).
        2.  Standard deviation $\sigma$ (encoded via $|D|$).
        3.  Visibility/Occlusion (single average value).
        4.  Material color (opacity-weighted, alpha pre-multiplied).
        5.  Normal (pre-multiplied by opacity).

### Index Calculation
For a contiguous 1D array `_voxelData` of generic type `T`, 3D coordinates $(x, y, z)$ are mapped to a 1D index using row-major ordering (implied by "left-to-right, bottom-to-top, back-to-front" [S2, unofficial] and standard C-style array flattening [S6, unofficial]):

$$
\text{Index}(x, y, z) = x + y \cdot W + z \cdot W \cdot H
$$

Where:
-   $W$ is Width.
-   $H$ is Height.
-   $D$ is Depth (used for bounds checking).

### Steps for Access
1.  **Compute Index**: Calculate the linear index from $(x, y, z)$ using the formula above.
2.  **Bounds Check**: Verify $0 \le \text{Index} < W \cdot H \cdot D$.
3.  **Retrieve Data**:
    -   If valid, return `_voxelData[Index]`.
    -   If invalid, return a default value (e.g., $[0,0,0,0]$ for RGBA [S4, unofficial] or `default(T)` [S6, unofficial]).

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class VoxelGrid:
    def __init__(self, width, height, depth, data_type=float):
        self.width = width
        self.height = height
        self.depth = depth
        self.volume_size = width * height * depth
        # Initialize contiguous array with default values (e.g., 0.0 or 0)
        self._voxel_data = [data_type(0)] * self.volume_size

    def set_voxel(self, x, y, z, value):
        """
        Sets the voxel at (x, y, z) to value.
        Raises IndexError if out of bounds.
        """
        index = self._xyz_to_index(x, y, z)
        if 0 <= index < self.volume_size:
            self._voxel_data[index] = value
        else:
            raise IndexError(f"Voxel ({x}, {y}, {z}) out of bounds for grid {self.width}x{self.height}x{self.depth}")

    def get_voxel(self, x, y, z):
        """
        Gets the voxel at (x, y, z).
        Returns default value (0) if out of bounds.
        """
        index = self._xyz_to_index(x, y, z)
        if 0 <= index < self.volume_size:
            return self._voxel_data[index]
        else:
            # Out-of-bounds access returns [0,0,0,0] or default [S4, S6, unofficial]
            return self._voxel_data[0] # Assuming default is 0 or first element

    def try_get_voxel(self, x, y, z):
        """
        Safe access method. Returns (True, value) if valid, (False, default) if invalid.
        """
        index = self._xyz_to_index(x, y, z)
        if 0 <= index < self.volume_size:
            return True, self._voxel_data[index]
        else:
            return False, self._voxel_data[0]

    def _xyz_to_index(self, x, y, z):
        """
        Maps 3D coordinates to 1D index using row-major order.
        Order: Left-to-right (x), Bottom-to-top (y), Back-to-front (z) [S2, unofficial].
        """
        return x + y * self.width + z * self.width * self.height
```

### Parameters and Edge Cases
-   **Invalid Node/Out-of-Bounds**: Addressing a voxel position not in the world returns $[0,0,0,0]$ [S4, unofficial]. Direct access via raw index without bounds checking throws `IndexOutOfRangeException` [S6, unofficial].
-   **Root Node**: In hierarchical variants, `ROOT_NODE` is often 0 [S3, unofficial].
-   **Invalid Node Ref**: `INVALID_NODE` is `0xFFFFFFFF` [S3, unofficial].
-   **Precision**: Using 32-bit floats for RGB/Opacity allows full range $[0.0, 1.0]$ [S4, unofficial]. Bit-packed formats (8/16/8 bits) reduce precision but save memory [S1, unofficial].

## Subtopics

### [Voxel Material Packing](4907-voxel-material-packing.md)
Voxel Material Packing is a memory layout optimization that encodes a voxel's material ID and RGB color into a single 32-bit integer. This approach allows a renderer to fetch material properties and color data in a single memory access, reducing bandwidth and simplifying the ray-marching loop. It differs from [Voxel Data Layout](4076-voxel-data-layout.md) by strictly binding material identity and color to the voxel value itself, rather than storing them in separate structures or using indirect indexing for color.

### [Voxel Renderer with Top-Level Grid and Bricks](8114-voxel-renderer-with-top-level-grid-and-bricks.md)
This method renders voxel data using a two-level grid structure (top-level grid containing 8x8x8 voxel bricks) to optimize memory transfer and cache locality. It solves the bottleneck of transferring large voxel datasets to the GPU by synchronizing only changed bricks and leveraging high internal GPU bandwidth for staging. It differs from flat dense grid renderers by introducing spatial tiling for cache efficiency and dynamic update budgets.

### Voxel Representation with Gaussian Lobes
This variant stores directional information using isotropic Gaussian lobes characterized by an average vector $D$ and standard deviation $\sigma$. The variance is encoded via the norm $|D|$ such that $\sigma^2 = \frac{1 - |D|}{|D|}$ to ease interpolation [S5]. Fields include $D$, $\sigma$, visibility/occlusion, pre-multiplied material color, and opacity-pre-multiplied normal. This is more accurate than single values for representing light behavior but requires more complex interpolation logic [S5].

## Operating characteristics
-   **Time Complexity**: Access is $O(1)$ given coordinates, requiring only arithmetic for index calculation.
-   **Memory Complexity**: $O(W \cdot H \cdot D)$. Memory usage is proportional to the volume of the bounding box, regardless of sparsity.
-   **Performance**: High cache locality for contiguous traversal (e.g., ray marching along axes) if data is stored in row-major order [S2, S6, unofficial]. However, for sparse scenes, it wastes memory and bandwidth on empty voxels.
-   **Strengths**: Simple implementation, no pointer chasing, predictable memory access patterns.
-   **Limitations**: Poor scalability for large, sparse volumes. Fixed resolution requires full grid allocation.
-   **When to Use**: Dense volumes, small scenes, or as a base for hierarchical structures.
-   **When Not to Use**: Large open-world scenes with sparse objects, where hierarchical methods like [Voxel Data Structure](3982-voxel-data-structure.md) are preferred.

## Background
The naive tensor layout is a foundational concept in voxel rendering, often used in initial implementations and educational contexts. It appears in early voxel simulation projects [S1, unofficial] and tutorials on neural radiance fields (NeRF) and plenoxels, where it serves as a baseline before introducing spherical harmonics or hierarchical structures [S4, unofficial]. The binary index format with specific ordering (left-to-right, bottom-to-top, back-to-front) is used in surface reconstruction tools like SurfaceNets [S2, unofficial]. The Gaussian lobe representation was introduced in the context of voxel cone tracing for indirect illumination [S5].

## Sources
[S1] Ray-Marched Voxel Simulation, https://www.jorisar.nl/blog/VoxelPlayground/, fetched 2026-10-25 (unofficial)
[S2] SurfaceNets for Multi-Label Segmentations with Preservation of Sharp Boundaries., https://europepmc.org/article/MED/36325473, fetched 2026-10-26 (unofficial)
[S3] marceline-cramer/svo-cpu, https://github.com/marceline-cramer/svo-cpu, fetched 2026-10-27 (unofficial)
[S4] Total Internal Reflection, https://avishek.net/2022/12/05/pytorch-guide-plenoxels-nerf-part-2.html, fetched 2026-10-29 (unofficial)
[S5] PDF Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
[S6] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
