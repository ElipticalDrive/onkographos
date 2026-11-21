# Hierarchical Data Structure Storage

Also called: Octree storage layout

## Overview
Hierarchical Data Structure Storage is a method for organizing illumination samples in participating media into a spatial hierarchy to accelerate rendering when lighting conditions change [S1]. It solves the problem of efficiently storing and traversing complex scattering data by grouping samples into a tree structure, allowing for fast updates and localized recomputation [S1]. It is distinguished from sibling methods like [Packed Tensor Representation](1611-packed-tensor-representation.md) by its explicit separation of single and multiple scattering into two parallel tree structures, which increases data locality at the cost of structural redundancy [S1].

## How it works
**Inputs:** Illumination samples computed for current lighting conditions [S1].
**Outputs:** Two 1D buffer arrays representing the hierarchical trees for single and multiple scattering [S1].

**Data Structures:**
The method uses a Structure of Arrays (SoA) for leaf nodes and a compact 1D buffer for internal nodes [S1].

1.  **Leaf Node Layout (Sample Table):**
    Illumination samples are stored in groups of eight [S1]. For each sample, the following fields are stored in a SoA layout:
    *   Position ($\mathbf{p}$)
    *   Direction ($\mathbf{d}$)
    *   Incoming Radiance ($L_{in}$)
    *   Scaled Surface Area ($A_{scaled}$)
    *   Interval Length along the Ray ($\Delta t$) [S1].

2.  **Internal Node Layout (Tree Buffer):**
    The hierarchy is stored as a 1D buffer where each node is represented by two `float4` vectors [S1]. This compact representation encodes the bounding box, child index, and child count [S1].
    *   `min_childIdx` (`float4`):
        *   $x, y, z$: The minimum coordinates $(x_{min}, y_{min}, z_{min})$ of the node's Axis-Aligned Bounding Box (AABB) [S1].
        *   $w$: The index of the first child in the buffer [S1].
    *   `max_nChild` (`float4`):
        *   $x, y, z$: The maximum coordinates $(x_{max}, y_{max}, z_{max})$ of the node's AABB [S1].
        *   $w$: The number of children of the node [S1].

3.  **Tree Construction:**
    Two separate trees are built: one for single scattering and one for multiple scattering [S1]. They share the same internal topology (parent-child relationships) but store different semantic data at the nodes [S1].

    *   **Single Scattering Nodes:**
        *   Store only the AABB of the node [S1].
        *   The AABB is the union of the AABBs of its children [G].

    *   **Multiple Scattering Nodes:**
        *   Store the average position, average direction, average incoming radiance, volume of influence, and depth of the node [S1].
        *   These values are computed as the averages of the corresponding values of the children nodes [S1].

4.  **Algorithm Steps:**
    1.  Compute illumination samples for the current lighting state [S1].
    2.  Group samples into sets of eight and populate the leaf sample table using the SoA layout [S1].
    3.  Build the single scattering tree:
        *   Create leaf nodes from the sample groups [S1].
        *   Recursively compute the AABB for each internal node as the union of its children's AABBs [G].
        *   Store nodes in the 1D buffer using the `min_childIdx` and `max_nChild` encoding [S1].
    4.  Build the multiple scattering tree:
        *   Create leaf nodes from the sample groups [S1].
        *   Recursively compute the average position, direction, radiance, volume of influence, and depth for each internal node from its children's values [S1].
        *   Store nodes in a second 1D buffer using the same `min_childIdx` and `max_nChild` encoding [S1].

**Parameters:**
*   Group size: 8 samples per leaf node [S1].
*   Tree depth: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Redundancy:** Maintaining two separate trees for single and multiple scattering introduces redundancy, as the structural hierarchy is stored twice [S1]. This is a deliberate trade-off to increase data locality during traversal [S1].
*   **Lighting Changes:** The hierarchy must be rebuilt or updated whenever lighting conditions change; it is static for a given lighting state [S1].

## Subtopics

### LCC Format Family
An industrial-grade data format family evolved from LCC11 to LCC22 [S2, unofficial]. LCC1 supports spatial chunking and Level-of-Detail (LoD) organization for progressive loading and large-scale scene visualization [S2, unofficial]. LCC2 decouples data storage from LoD partitioning, using a flexible hierarchical LoD tree to index different payload types and storage backends [S2, unofficial]. Combined with compact binary layout and parameter quantization, it reduces storage by up to 90% compared to standard PLY files [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Building the hierarchy requires computing illumination samples and traversing the tree to aggregate node values [S1]. Traversal is optimized via a two-step process (coarse subtree extraction followed by fine traversal), reducing the main computational cost [S1].
*   **Memory Complexity:** The method stores two full copies of the tree structure (one for single, one for multiple scattering), doubling the memory overhead for the hierarchy itself compared to a single tree [S1]. The leaf data is stored in a compact SoA format [S1].
*   **Strengths:** High data locality during scattering computation due to separate trees for different scattering orders [S1]. Efficient updates for static lighting scenarios [S1].
*   **Limitations:** Redundant storage of the tree structure [S1]. Requires rebuilding the hierarchy when lighting changes [S1].
*   **When to use:** When rendering participating media with complex single and multiple scattering where lighting changes are infrequent, and data locality is critical for performance [S1].

## Background
The method addresses the computational cost of evaluating scattering in participating media by precomputing and spatially organizing illumination samples [S1].

## Sources
[S1] Point-Based Rendering for Homogeneous Participating Media with Refractive Boundaries, https://doi.org/10.1109/TVCG.2017.2768525, fetched 2026-10-26
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
