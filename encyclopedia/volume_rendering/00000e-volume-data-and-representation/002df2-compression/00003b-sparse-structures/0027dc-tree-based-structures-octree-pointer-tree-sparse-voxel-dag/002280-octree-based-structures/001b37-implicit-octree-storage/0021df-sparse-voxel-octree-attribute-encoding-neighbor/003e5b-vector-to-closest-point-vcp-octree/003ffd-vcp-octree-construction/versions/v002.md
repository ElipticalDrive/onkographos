# VCP Octree Construction
Also called: Bottom-up VCP Octree, Dynamic Memory Management, Hierarchical Subvolume Rendering, Hierarchical VCP Octree, Subvolume Loading

## Overview
This method constructs a hierarchical octree structure from a Vector-to-Closest-Point (VCP) grid to store VCP information only in the close vicinity of the surface, thereby lowering memory requirements while ensuring no intersection is lost during ray-casting [S1, unofficial]. It sets itself apart from flat VCP grid implementations by using a bottom-up merging strategy to eliminate empty space and by dynamically loading refined subvolumes into GPU memory based on camera proximity, enabling significantly finer resolution with reduced runtime memory traffic [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A VCP grid (scalar distances for empty blocks, VCP vectors for non-empty blocks) and a scene bounding box.
- Output: An implicit octree structure where nodes are encoded as signed 32-bit integers, and leaf data is stored in a compact 3D texture.

**Data Structures:**
1.  **Octree Nodes:** Each node is a signed 32-bit integer.
    -   Positive integer: Internal node. The value encodes the indices of its 8 children.
    -   Negative integer: Leaf node. The value is `-(leafIdx + 1)`, where `leafIdx` points to the index in the VCP leaf data structure.
2.  **VCP Leaf Data:** A 3D texture storing compressed VCP vectors.
    -   Each non-empty leaf stores $2^3 = 8$ VCP vectors.
    -   Adjacent leaves sharing grid points share VCP vectors to reduce memory usage.
    -   Vectors are stored in spherical coordinates $(r, \theta, \phi)$ packed into 32 bits:
        -   Radius $r$: 13 bits (normalized unsigned integer).
        -   Azimuthal angle $\theta$: 9 bits (normalized unsigned integer, range $[-\pi/2, \pi/2]$).
        -   Polar angle $\phi$: 10 bits (normalized unsigned integer, range $[0, 2\pi]$).

**Algorithm Steps:**
1.  **Block Evaluation:** Divide the scene bounding box into initial blocks (leaves). For each block, compute the minimum distance $d_{min}$:
    -   If the block is empty, $d_{min}$ is the minimum of all scalar per-vertex distances.
    -   If the block is non-empty, $d_{min}$ is the minimum of all VCP distances of the vertices of the non-empty blocks.
2.  **Thresholding:** Define a threshold $t$. Set $t = 2 \times d_{diag}$, where $d_{diag}$ is the distance between two diagonally neighboring grid points at the finest resolution. This ensures that an intersection with the object is never lost during hierarchy construction.
3.  **Marking Empty Blocks:** If $d_{min} > t$, mark the corresponding block as empty.
4.  **Bottom-Up Merging:** Construct the octree by merging groups of $2^3$ blocks (in 3D) or $2^2$ blocks (in 2D illustration) according to the following principles:
    -   If all blocks in a group are empty, the parent node is marked as empty.
    -   If all blocks in a group are non-empty and have the same VCP data (or can be merged), the parent node becomes an internal node pointing to the children.
    -   If blocks differ, the parent remains an internal node with distinct children.
    -   Repeat until the root is reached.
5.  **Leaf Data Compression and Sorting:**
    -   Transform each VCP vector $\vec{v}$ to spherical coordinates:
        -   $r_{raw} = \|\vec{v}\| / r_{max}$, where $r_{max}$ is a fixed maximum radius.
        -   $r = \min(r_{raw}, 1)$.
        -   Apply square root for precision: $r \leftarrow \sqrt{r}$.
        -   $\theta = \arcsin(v_y / \|\vec{v}\|)$ (azimuthal, $[-\pi/2, \pi/2]$).
        -   $\phi = \arctan2(v_z, v_x)$ (polar, $[0, 2\pi]$).
    -   Pack $r, \theta, \phi$ into 32 bits as described above.
    -   Sort leaves by octree level, then by xyz-order (x-axis greatest length) to maximize sharing of VCP vectors between adjacent blocks.
    -   Store sorted leaves in a 3D texture, allowing cells to share VCP vectors where possible.
6.  **Subvolume Construction (Dynamic Loading):**
    -   Divide the scene bounding box into subvolumes.
    -   For each subvolume, construct a separate VCP hierarchy by restricting the VCP grid to that region.
    -   During rendering, compute the set of subvolumes needed based on camera distance.
    -   Load refined subvolumes into GPU memory in the background using multithreading (e.g., DirectX 11).
    -   During ray-casting, if the distance to the camera is less than a threshold (e.g., $1/4$ of the object's bounding box diameter), perform a lookup in the refined structure.

**Parameters:**
-   $t$: Threshold for marking blocks as empty. Typically $t = 2 \times d_{diag}$ [S1, unofficial].
-   Subvolume threshold: Distance to camera for loading refined subvolumes. Typically $1/4$ of the object's bounding box diameter [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Rendering Artifacts:** If planes substitute triangles, triangle boundaries are lost. This is avoided by combining the method with trilinear interpolation, returning the VCP with the greater length as the final result [S1, unofficial].
-   **Popping Artifacts:** Occur when switching to different subvolume levels. Considered acceptable due to reduced runtime memory requirements enabling finer resolution [S1, unofficial].

## Subtopics

### [GPU Octree Encoding](10089-gpu-octree-encoding.md)
This method defines the memory layout and traversal optimization for the implicit octree used in VCP surface ray-casting. It solves the problem of efficient GPU memory access by encoding the entire octree hierarchy into a compact array of 32-bit signed integers, eliminating explicit pointer indirections. It sets itself apart from standard octree implementations by packing internal node children, empty leaf distances, and non-empty leaf indices into a single signed integer per node, enabling a flat, cache-friendly memory layout.

## Operating characteristics
-   **Time Complexity:** Construction is bottom-up, requiring a single pass over the grid to compute $d_{min}$ and subsequent passes for merging. Dynamic loading adds overhead but is performed in the background.
-   **Memory Complexity:** Significantly reduced compared to flat VCP grids by storing VCP information only near the surface. Leaf data compression further reduces memory usage to 32 bits per vector.
-   **Performance:** Enables finer resolution rendering on GPUs with limited memory. Dynamic loading allows adaptive refinement based on camera proximity.
-   **Strengths:** Low memory footprint, no visual artifacts from intersection loss, adaptive refinement.
-   **Limitations:** Potential popping artifacts during subvolume switching, complexity in implementation due to dynamic loading and custom data types.

## Background
The VCP Octree Construction method was introduced in the paper "Vector-to-Closest-Point Octree for Surface Ray-Casting" [S1, unofficial]. It builds upon the VCP surface representation, which stores a vector field where each grid point stores a vector to the closest point on the original geometry. The octree structure was developed to address the high memory requirements of storing VCP data for large scenes, enabling efficient surface ray-casting on GPUs.

## Sources
[S1] Vector-to-Closest-Point Octree for Surface Ray-Casting, https://doi.org/10.2312/vmv.20151259, fetched 2026-10-26 (unofficial)
