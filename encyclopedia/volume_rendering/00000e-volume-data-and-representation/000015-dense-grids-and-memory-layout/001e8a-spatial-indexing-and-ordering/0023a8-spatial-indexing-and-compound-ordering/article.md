# Spatial Indexing and Compound Ordering

Also called: Compound Ordering Index, Space-Filling Curve Indexing

## Overview
Spatial Indexing and Compound Ordering maps $N$-dimensional data (spatial coordinates plus discrete non-spatial dimensions like time or channel) into a 1D linear array to optimize cache locality and network fetching. It solves the problem of efficiently accessing subsets of multidimensional volumetric data by ensuring that points close in Euclidean distance or sharing discrete attributes are stored contiguously. It differs from simple [Arithmetic Strides](1797-spatial-indexing-and-ordering.md) by using space-filling curves for spatial locality, and from pure [Morton Code Encoding](1700-morton-code-encoding-decoding.md) by hierarchically sorting discrete dimensions as the most significant keys before applying spatial ordering within slices [S1, unofficial].

## How it works
**Inputs:**
- A set of data elements, each defined by a discrete non-spatial coordinate $d$ (e.g., time step, channel) and a spatial coordinate vector $\mathbf{x} \in \mathbb{R}^3$.
- A chunking strategy defining the spatial extent of each chunk.

**Outputs:**
- A linear array of chunks, sorted according to the compound ordering.
- Metadata containing precomputed Axis-Aligned Bounding Boxes (AABBs) for each chunk.

**Data Structures:**
- **Chunk:** A contiguous block of data elements.
- **Metadata:** For each chunk, stores an AABB representing the extent of the data it contains across all dimensions (spatial $\pm$ tolerance, and discrete slice positions).

**Algorithm Steps:**

1.  **Sort by Discrete Dimensions:**
    Group elements by their discrete non-spatial coordinates $d$. Sort the groups such that elements with the same $d$ are contiguous in memory. This ensures that accessing a specific time step or channel involves fetching contiguous memory ranges [S1, unofficial].

2.  **Apply Space-Filling Curve (SFC) within Slices:**
    Within each group of identical discrete coordinate $d$, sort the elements based on their spatial coordinates $\mathbf{x}$ using a space-filling curve.
    -   **Hilbert Curve:** Used as the default for serialization due to superior spatial locality [S1, unofficial].
    -   **Morton (Z-order) Curve:** Used for in-memory reordering during fitting/computation due to lower computational cost [S1, unofficial].
    
    The spatial index $I_{spatial}$ for a point $\mathbf{x} = (x, y, z)$ is computed by interleaving the bits of its coordinates (for Morton) or traversing the Hilbert curve. The final linear index $I$ for an element is determined by its position in the sorted discrete group plus its SFC index within that group.

3.  **Chunking:**
    Partition the sorted linear array into chunks of fixed size (e.g., 1 MB). Because the data is sorted by discrete dimensions first, chunks corresponding to a single time step are contiguous. Chunks spanning multiple time steps are minimized or handled explicitly [S1, unofficial].

4.  **Precompute Metadata (AABBs):**
    For each chunk, compute the minimum and maximum bounds across all dimensions:
    -   Spatial bounds: $\min(\mathbf{x})$ and $\max(\mathbf{x})$ for all elements in the chunk.
    -   Discrete bounds: The range of discrete coordinates $d$ present in the chunk.
    Store these as an $N$-dimensional AABB.

5.  **Query and Fetch:**
    When a view change or time step selection occurs:
    -   Define the query region $Q$ (e.g., current camera frustum and active time steps).
    -   Perform a fast AABB intersection test against the precomputed metadata of all chunks.
    -   Select only chunks where $AABB_{chunk} \cap Q \neq \emptyset$.
    -   Fetch these chunks. Because of the ordering, chunks for a new time step are adjacent to the previous one, allowing efficient prefetching of neighbor chunks [S1, unofficial].

**Parameters:**
-   **SFC Type:** Hilbert (default for storage/serialization) or Morton (for computation) [S1, unofficial].
-   **Chunk Size:** Typically 1 MB, balancing HTTP range request overhead with cache efficiency [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Precision:** Floating-point coordinates must be quantized to integer grid indices before SFC encoding.
-   **Boundary Elements:** Elements on the boundary of a chunk may require fetching adjacent chunks if the query region spans the boundary [S1, unofficial].
-   **Sparse Data:** If data is sparse, large chunks may contain empty space, reducing the selectivity of the AABB test.

## Subtopics

### Multi-Resolution Tetrahedral Region Indicator
A spatial indexing scheme for multi-resolution tetrahedral grids where a unique index encodes the intersection of tetrahedra containing a point $\mathbf{x}$ across all resolution levels. It is chosen when working with hierarchical tetrahedral meshes for analytic isosurface extraction, such as in SDF learning [S2].

**Definition:**
The region indicator $\mathbf{r}(\mathbf{x})$ is a concatenation of levelwise tetrahedron indices. At each level $\ell$, the grid cube is subdivided into six congruent tetrahedra. The index at level $\ell$ is defined as:
$$ \mathbf{r}^{(\ell)}(\mathbf{x}) := [ a^{(\ell)}(\mathbf{x}), t^{(\ell)}(\mathbf{x}) ] \in \mathbb{Z}^4 $$
where $a^{(\ell)}(\mathbf{x})$ is the anchor displacement and $t^{(\ell)}(\mathbf{x}) \in \{0, 1, 2, 3, 4, 5\}$ is the tetrahedron index containing $\mathbf{x}$ [S2].

The full region indicator is:
$$ \mathbf{r}(\mathbf{x}) := \bigoplus_{\ell=0}^{L-1} \mathbf{r}^{(\ell)}(\mathbf{x}) \in \mathbb{Z}^{4L} $$
where $\bigoplus$ denotes concatenation [S2].

**Parameters:**
-   $L$: Number of resolution levels. Typical value: 4 [S2].
-   $d$: Feature dimension per level. Typical value: 2 [S2].

**Usage:**
This index allows for efficient identification of neighboring cells when a point lies on a boundary by perturbing the tetrahedron index based on a barycentric mask [S2].

## Operating characteristics
-   **Time Complexity:** Sorting is $O(N \log N)$ where $N$ is the number of elements. Querying is $O(C)$ where $C$ is the number of chunks, but typically much faster due to spatial coherence and early rejection via AABB tests.
-   **Memory Complexity:** $O(N)$ for the data array plus $O(C)$ for metadata.
-   **Performance:** Provides excellent cache locality for spatial queries and minimizes network requests for web-based viewers by coalescing chunk fetches into few HTTP range requests [S1, unofficial].
-   **Strengths:** Optimizes for multidimensional data with discrete axes (time, channels); supports efficient streaming and prefetching.
-   **Limitations:** Requires pre-sorting and metadata generation; Hilbert curve computation is more expensive than Morton [S1, unofficial].

## Background
The method originates from the requirements of scalable web visualization of multidimensional scientific data, specifically in the Luxar viewer for microscopy data [S1, unofficial]. It combines principles of space-filling curves (Morton, Hilbert) with hierarchical sorting of discrete dimensions to enable efficient chunk-based loading over HTTP [S1, unofficial]. The multi-resolution tetrahedral variant is derived from TetraSDF, which uses region indicators for analytic isosurface extraction in neural implicit representations [S2].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, https://arxiv.org/abs/2511.16273v4, fetched 2026-10-26
