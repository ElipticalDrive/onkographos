# Voxel Data Layout

Also called: Input Data Format, Node Data Layout, Voxel Cube, Voxel Data Structure (Naive), World Tensor

## Overview
Voxel Data Layout organizes a dense 3D grid into contiguous memory for efficient spatial indexing during volume rendering [S4, S6, unofficial]. Unlike sparse structures that skip empty space with pointers, this layout allocates fixed memory per voxel, prioritizing predictable linear access and cache locality [S6, unofficial].

## How it works
### Inputs and Outputs
- **Input**: 3D integer coordinates $(x, y, z)$ within bounds $[0, \text{Width}-1] \times [0, \text{Height}-1] \times [0, \text{Depth}-1]$ [S6, unofficial].
- **Output**: A voxel data structure (e.g., a 32-bit integer, a struct, or a tensor) containing material/color/opacity information [S1, S4, S7, unofficial].

### Data Structures and Layout
The layout varies by implementation goal, but common patterns include:

1.  **Naive Tensor (World Tensor)**: A 3D array or flattened 1D array where each voxel stores opacity $\sigma$ and an RGB triplet [S4, unofficial].
    -   **Fields**: $\sigma$ (float), $R$ (float), $G$ (float), $B$ (float).
    -   **Size**: 4 numbers per voxel [S4, unofficial].
    -   **Indexing**: $Index = x + y \cdot \text{Width} + z \cdot \text{Width} \cdot \text{Height}$ [S6, unofficial].

2.  **Packed 32-bit Integer**: Used for performance-critical or limited-memory contexts [S7, unofficial].
    -   **Bits [31..24]**: 8-bit Material ID (max 256 materials) [S7, unofficial].
    -   **Bits [23..16]**: Red channel [S7, unofficial].
    -   **Bits [15..8]**: Green channel [S7, unofficial].
    -   **Bits [7..0]**: Blue channel [S7, unofficial].
    -   *Alternative Packing*: 8 bits Type, 16 bits Color, 8 bits Auxiliary [S1, unofficial].

3.  **Binary Index Format**: Stored in files or buffers.
    -   **Order**: Left-to-right, bottom-to-top, back-to-front [S2, unofficial].
    -   **Precision**: 8-bit or 16-bit indices [S2, unofficial].

### Numbered Steps for Access
1.  **Validate Bounds**: Check if $0 \le x < \text{Width}$, $0 \le y < \text{Height}$, and $0 \le z < \text{Depth}$ [S6, unofficial].
2.  **Compute Linear Index**:
    $$ \text{index} = x + y \cdot \text{Width} + z \cdot \text{Width} \cdot \text{Height} $$
    [S6, unofficial]
3.  **Retrieve Data**:
    -   If using a contiguous array `voxel_data` of type `T`:
        ```python
        voxel_data = voxel_data[index]
        ```
    -   If using a packed 32-bit integer `raw`:
        ```python
        material_id = (raw >> 24) & 0xFF
        r = (raw >> 16) & 0xFF
        g = (raw >> 8) & 0xFF
        b = raw & 0xFF
        ```
    [S6, S7, unofficial]
4.  **Handle Out-of-Bounds**: If the coordinate is invalid, return a default value.
    -   For the Naive Tensor: Return $[0, 0, 0, 0]$ (zero opacity, black color) [S4, unofficial].
    -   For Safe Access Methods: Return `false` and set output to `default` (e.g., 0 for bytes) [S6, unofficial].

### Formulas
-   **Index Calculation**:
    $$ I(x, y, z) = x + y \cdot W + z \cdot W \cdot H $$
    Where $W$ is Width, $H$ is Height [S6, unofficial].

-   **Variance Encoding (for Gaussian Lobes)**:
    $$ \sigma^2 = \frac{1 - |D|}{|D|} $$
    Where $|D|$ is the norm of the average direction vector $D$. This encoding eases interpolation [S5].

### Parameters
-   **Width, Height, Depth**: Integer dimensions defining the volume bounds [S6, unofficial]. Typical values: Not given in the sources.
-   **Voxel Scale**: In some implementations, voxel edges are scaled to $1/8$ unit length to increase geometric detail [S1, unofficial].
-   **Max Materials**: 256, determined by 8-bit material ID [S7, unofficial].

### Edge Cases and Pitfalls
-   **Out-of-Bounds Access**: Direct array access without bounds checking throws an `IndexOutOfRangeException` (or equivalent segfault) [S6, unofficial]. Always use a safe accessor (e.g., `TryGetVoxelData`) that checks $0 \le \text{index} < \text{Length}$ [S6, unofficial].
-   **Default Return Value**: Addressing a non-existent voxel position must consistently return a defined default (e.g., $[0,0,0,0]$) to prevent rendering artifacts or crashes [S4, unofficial].
-   **Memory Contiguity**: The layout assumes a single contiguous native array. Strided or interleaved layouts require different indexing logic [S6, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_voxel(x, y, z, width, height, depth, voxel_data):
    """
    Returns voxel data. If out of bounds, returns default value.
    For packed 32-bit integers, returns a dict with material, r, g, b.
    For tensors, returns the raw tensor.
    """
    # 1. Validate bounds
    if not (0 <= x < width and 
            0 <= y < height and 
            0 <= z < depth):
        return [0, 0, 0, 0]

    # 2. Compute linear index
    index = x + y * width + z * width * height

    # 3. Retrieve data
    raw = voxel_data[index]

    # 4. Unpack if necessary (example for 32-bit packed layout)
    if isinstance(raw, int):
        material_id = (raw >> 24) & 0xFF
        r = (raw >> 16) & 0xFF
        g = (raw >> 8) & 0xFF
        b = raw & 0xFF
        return {"material": material_id, "r": r, "g": g, "b": b}
    
    # If raw is a tensor/struct (e.g., [sigma, r, g, b]):
    return raw
```

## Subtopics

### Voxel Representation with Gaussian Lobes
A variant where each voxel stores directional light information rather than just static color. It uses an average direction vector $D$ and a standard deviation $\sigma$ (encoded via $|D|$) to model isotropic Gaussian lobes [S5]. Fields include: 1. Average vector $D$, 2. Standard deviation $\sigma$, 3. Visibility/Occlusion (single average value), 4. Material color (opacity-weighted, alpha pre-multiplied), 5. Normal (pre-multiplied by opacity) [S5]. Chosen for compactness compared to storing arbitrary distributions, though it lacks view-dependency for thin objects [S5].

### Voxel Data Volume Storage and Access
Focuses on the software interface for accessing the contiguous array. It emphasizes safe access methods (`TryGetVoxelData`) that prevent `IndexOutOfRangeException` by validating indices against `Width`, `Height`, and `Depth` before accessing `voxel_data` [S6, unofficial]. This is a critical implementation detail for robustness in C#/C++ environments [S6, unofficial].

### Voxel Material Packing
A specific 32-bit integer layout optimization that packs Material ID (8 bits), Red (8 bits), Green (8 bits), and Blue (8 bits) into a single integer [S7, unofficial]. This allows managing up to 256 distinct materials via a `MaterialManager` while minimizing memory bandwidth during ray traversal [S7, unofficial].

## Operating characteristics
-   **Time Complexity**: $O(1)$ for voxel access given coordinates, assuming contiguous memory and no hashing [S6, unofficial].
-   **Memory Complexity**: $O(W \cdot H \cdot D)$, where $W, H, D$ are dimensions. Memory usage is high for sparse scenes as empty space is allocated [S4, unofficial].
-   **Measured Performance**: Not given in the sources.
-   **Scaling**: Not given in the sources.
-   **Strengths and Limitations**: High cache locality due to contiguous storage [S6, unofficial]. Inefficient for sparse volumes; out-of-bounds checks are mandatory to prevent crashes [S6, unofficial]. The naive tensor layout (RGB + $\sigma$) is less expressive than spherical harmonic variants (28 numbers/voxel) but faster to compute [S4, unofficial].
-   **When to use it and when not**: Use for dense scenes requiring predictable linear access and simple ray marching [S1, S4, unofficial]. Avoid for highly sparse volumes where sparse structures would save memory [S4, unofficial].

## Background
The concept of a dense voxel grid is foundational in volume rendering. Early implementations used simple 3D tensors indexed by world coordinates, storing opacity and RGB triplets for demonstration purposes before evolving to more complex representations like spherical harmonics [S4, unofficial]. The "naive" structure is often the starting point for implementing ray marchers, where equidistant steps sample the grid [S1, unofficial]. Binary formats for storage often use specific scan orders (left-to-right, bottom-to-top, back-to-front) to facilitate file I/O [S2, unofficial].

## Sources
[S1] Ray-Marched Voxel Simulation, https://www.jorisar.nl/blog/VoxelPlayground/, fetched 2026-10-25 (unofficial)
[S2] SurfaceNets for Multi-Label Segmentations with Preservation of Sharp Boundaries., https://europepmc.org/article/MED/36325473, fetched 2026-10-26 (unofficial)
[S4] Total Internal Reflection, https://avishek.net/2022/12/05/pytorch-guide-plenoxels-nerf-part-2.html, fetched 2026-10-29 (unofficial)
[S5] PDF Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
[S6] Eldemarkki/Marching-Cubes-Terrain, https://github.com/Eldemarkki/Marching-Cubes-Terrain, fetched 2026-10-26 (unofficial)
[S7] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
