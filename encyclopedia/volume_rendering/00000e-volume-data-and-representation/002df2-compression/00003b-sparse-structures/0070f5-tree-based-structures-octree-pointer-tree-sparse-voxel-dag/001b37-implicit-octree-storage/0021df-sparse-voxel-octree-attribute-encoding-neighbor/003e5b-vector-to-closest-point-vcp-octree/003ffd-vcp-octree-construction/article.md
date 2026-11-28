# VCP Octree Construction
Also called: Bottom-up VCP Octree, Dynamic Memory Management, Hierarchical Subvolume Rendering, Hierarchical VCP Octree, Subvolume Loading

## Overview
This method constructs a hierarchical octree structure from a Vector-to-Closest-Point (VCP) grid to store VCP information only in the close vicinity of the surface, thereby lowering memory requirements while ensuring no intersection is lost during ray-casting [S1, unofficial]. It sets itself apart from flat VCP grid implementations by using a bottom-up merging strategy to eliminate empty space and by dynamically loading refined subvolumes into GPU memory based on camera proximity, enabling significantly finer resolution with reduced runtime memory traffic [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A VCP grid (scalar distances for empty blocks, VCP vectors for non-empty blocks) and a scene bounding box [S1, unofficial].
- Output: An implicit octree structure where nodes are encoded as signed 32-bit integers, and leaf data is stored in a compact 3D texture [S1, unofficial].

**Data Structures:**
1. **Octree Nodes:** Each node is a signed 32-bit integer [S1, unofficial].
    - Positive integer: Internal node. The value encodes the indices of its 8 children [S1, unofficial].
    - Negative integer: Leaf node. The value is `-(leafIdx + 1)`, where `leafIdx` points to the index in the VCP leaf data structure [S1, unofficial].
2. **VCP Leaf Data:** A 3D texture storing compressed VCP vectors [S1, unofficial].
    - Each non-empty leaf stores $2^3 = 8$ VCP vectors [S1, unofficial].
    - Adjacent leaves sharing grid points share VCP vectors to reduce memory usage [S1, unofficial].
    - Vectors are stored in spherical coordinates $(r, \theta, \phi)$ packed into 32 bits:
        - Radius $r$: 13 bits (normalized unsigned integer) [S1, unofficial].
        - Azimuthal angle $\theta$: 9 bits (normalized unsigned integer, range $[-\pi/2, \pi/2]$) [S1, unofficial].
        - Polar angle $\phi$: 10 bits (normalized unsigned integer, range $[0, 2\pi]$) [S1, unofficial].

**Algorithm Steps:**
1. **Block Evaluation:** Divide the scene bounding box into initial blocks (leaves) [S1, unofficial]. For each block, compute the minimum distance $d_{min}$:
    - If the block is empty, $d_{min}$ is the minimum of all scalar per-vertex distances [S1, unofficial].
    - If the block is non-empty, $d_{min}$ is the minimum of all VCP distances of the vertices of the non-empty blocks [S1, unofficial].
2. **Thresholding:** Define a threshold $t$ [S1, unofficial]. Set $t = 2 \times d_{diag}$, where $d_{diag}$ is the distance between two diagonally neighboring grid points at the finest resolution [S1, unofficial]. This ensures that an intersection with the object is never lost during hierarchy construction [S1, unofficial].
3. **Marking Empty Blocks:** If $d_{min} > t$, mark the corresponding block as empty [S1, unofficial].
4. **Bottom-Up Merging:** Construct the octree by merging groups of $2^3$ blocks according to the following principles [S1, unofficial]:
    - If all blocks in a group are empty, the parent node is marked as empty [S1, unofficial].
    - If all blocks in a group are non-empty and have the same VCP data (or can be merged), the parent node becomes an internal node pointing to the children [S1, unofficial].
    - If blocks differ, the parent remains an internal node with distinct children [S1, unofficial].
    - Repeat until the root is reached [S1, unofficial].
5. **Leaf Data Compression and Sorting:**
    - Transform each VCP vector $\vec{v}$ to spherical coordinates $(r, \theta, \phi)$ with $\theta \in [-\pi/2, \pi/2]$ and $\phi \in [0, 2\pi]$ [S1, unofficial].
    - Compute radius: $r_{raw} = \|\vec{v}\| / r_{max}$, where $r_{max}$ is a fixed maximum radius [S1, unofficial].
    - Clamp and apply square root for precision: $r = \min(r_{raw}, 1)$, then $r \leftarrow \sqrt{r}$ [S1, unofficial].
    - Pack $r, \theta, \phi$ into 32 bits as described above [S1, unofficial].
    - Sort leaves by octree level, then by xyz-order (x-axis greatest length) to maximize sharing of VCP vectors between adjacent blocks [S1, unofficial].
    - Store sorted leaves in a 3D texture, allowing cells to share VCP vectors where possible [S1, unofficial].
6. **Subvolume Construction (Dynamic Loading):**
    - Divide the scene bounding box into subvolumes [S1, unofficial].
    - For each subvolume, construct a separate VCP hierarchy by restricting the VCP grid to that region [S1, unofficial].
    - During rendering, compute the set of subvolumes needed based on camera distance [S1, unofficial].
    - Load refined subvolumes into GPU memory in the background using multithreading (e.g., DirectX 11) [S1, unofficial].
    - During ray-casting, if the distance to the camera is less than a threshold (e.g., $1/4$ of the object's bounding box diameter), perform a lookup in the refined structure [S1, unofficial].

**Parameters:**
- $t$: Threshold for marking blocks as empty. Typically $t = 2 \times d_{diag}$ [S1, unofficial].
- Subvolume threshold: Distance to camera for loading refined subvolumes. Typically $1/4$ of the object's bounding box diameter [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Rendering Artifacts:** If planes substitute triangles, triangle boundaries are lost. This is avoided by combining the method with trilinear interpolation, returning the VCP with the greater length as the final result [S1, unofficial].
- **Popping Artifacts:** Occur when switching to different subvolume levels. Considered acceptable due to reduced runtime memory requirements enabling finer resolution [S1, unofficial].

## Subtopics

### [GPU Octree Encoding](10089-gpu-octree-encoding.md)
This method defines the memory layout and traversal optimization for the implicit octree used in VCP (Vector-to-Closest-Point) surface ray-casting. It solves the problem of efficient GPU memory access by encoding the entire octree hierarchy into a compact array of 32-bit signed integers, eliminating explicit pointer indirections. It sets itself apart from standard octree implementations by packing internal node children, empty leaf distances, and non-empty leaf indices into a single signed integer per node, enabling a flat, cache-friendly memory layout.

## Operating characteristics
- **Time Complexity:** Construction requires a single pass over the grid to compute $d_{min}$ and subsequent passes for merging [G]. Dynamic loading adds background overhead but does not interrupt rendering [S1, unofficial].
- **Memory Complexity:** Significantly reduced compared to flat VCP grids by storing VCP information only near the surface [S1, unofficial]. Leaf data compression further reduces memory usage to 32 bits per vector [S1, unofficial].
- **Performance:** Enables finer resolution rendering on GPUs with limited memory [S1, unofficial]. Dynamic loading allows adaptive refinement based on camera proximity [S1, unofficial].
- **Strengths:** Low memory footprint, no visual artifacts from intersection loss, adaptive refinement [S1, unofficial].
- **Limitations:** Potential popping artifacts during subvolume switching, complexity in implementation due to dynamic loading and custom data types [S1, unofficial].

## Background
The VCP Octree Construction method was introduced in the paper "Vector-to-Closest-Point Octree for Surface Ray-Casting" [S1, unofficial]. It builds upon the VCP surface representation, which stores a vector field where each grid point stores a vector to the closest point on the original geometry [S1, unofficial]. The octree structure was developed to address the high memory requirements of storing VCP data for large scenes, enabling efficient surface ray-casting on GPUs [S1, unofficial].

## Sources
[S1] Vector-to-Closest-Point Octree for Surface Ray-Casting, https://doi.org/10.2312/vmv.20151259, fetched 2026-10-26 (unofficial)
