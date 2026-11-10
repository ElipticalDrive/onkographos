# Spatial Indexing and Compound Ordering

Also called: Compound Ordering Index, Space-Filling Curve Indexing

## Overview
Spatial Indexing and Compound Ordering is a data layout strategy that maps multi-dimensional data (typically 3D spatial coordinates plus discrete non-spatial dimensions like time or channel) into a linear memory structure to optimize spatial locality and I/O coalescing [S1, unofficial]. It solves the problem of efficiently fetching contiguous chunks of volumetric or splat-based data by ensuring that voxels close in Euclidean space are stored in adjacent memory locations [S1, unofficial]. Unlike simple row-major linearization, it uses a space-filling curve (SFC) for spatial sorting within discrete slices, and unlike pure SFC indexing, it hierarchically nests discrete non-spatial dimensions as the most significant sorting keys [S1, unofficial].

## How it works
The method constructs a linear index for data elements by combining discrete coordinate sorting with spatial space-filling curve ordering.

**Inputs and Outputs:**
- **Input:** A set of data elements, each defined by discrete non-spatial coordinates $D$ (e.g., time $t$, channel $c$) and spatial coordinates $\mathbf{x} = (x, y, z)$.
- **Output:** A 1D linear index $I$ for each element, and a set of precomputed metadata (AABBs) for chunks.

**Data Structures:**
1.  **Chunk:** A contiguous block of memory containing a subset of the sorted data.
2.  **Chunk Metadata:** For each chunk, an Axis-Aligned Bounding Box (AABB) defined by the minimum and maximum coordinates of all elements within the chunk across all dimensions (spatial and discrete).

**Steps:**
1.  **Discrete Sorting:** Sort all elements primarily by their discrete non-spatial coordinates $D$. Elements sharing the same $D$ values form a "slice" or "group" [S1, unofficial].
2.  **Spatial Sorting:** Within each discrete slice, sort the elements based on their spatial coordinates $\mathbf{x}$ using a space-filling curve [S1, unofficial].
    -   The mapping function $f: \mathbb{Z}^3 \to \mathbb{Z}$ maps $(x, y, z)$ to a 1D index.
    -   Two SFCs are supported:
        -   **Hilbert Curve:** Provides superior spatial locality and is the default for serialization [S1, unofficial].
        -   **Morton Code (Z-order):** Provides lower computational cost and is used for in-memory reordering during fitting [S1, unofficial].
3.  **Chunking:** Partition the sorted linear array into fixed-size chunks (e.g., 1 MB) [S1, unofficial].
4.  **Metadata Precomputation:** For each chunk, compute the AABB. The AABB is defined by the min/max values of the discrete coordinates and the spatial bounding box of the elements contained within that chunk [S1, unofficial].

**Formulas:**
Let $D$ be the discrete coordinate value (e.g., time step). Let $SFC(\mathbf{x})$ be the space-filling curve index for spatial position $\mathbf{x}$.
The primary sort key for an element is $(D, SFC(\mathbf{x}))$.
The linear index $I$ is the rank of the element in the globally sorted list defined by this key [S1, unofficial].

**Parameters:**
-   **SFC Type:** Hilbert (default for serialization) or Morton (for in-memory fitting) [S1, unofficial].
-   **Chunk Size:** Typically 1 MB, chosen to balance HTTP range request overhead and memory bandwidth [S1, unofficial].
-   **Tolerance:** A small tolerance added to the AABB extents during view frustum culling to account for floating-point precision or slight view changes [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sort_and_chunk(elements: list, discrete_keys: list, spatial_coords: list, sfc_type: str) -> list:
    """
    Sorts elements by discrete keys then spatial SFC, and returns chunks with metadata.
    """
    # Step 1: Create sortable tuples
    # Each element is associated with its discrete key and spatial SFC index
    sortable_items = []
    for i, elem in enumerate(elements):
        d_key = discrete_keys[i]
        x, y, z = spatial_coords[i]
        
        # Compute SFC index
        if sfc_type == "hilbert":
            sfc_idx = hilbert_encode(x, y, z)
        elif sfc_type == "morton":
            sfc_idx = morton_encode(x, y, z)
        else:
            raise ValueError("Unsupported SFC type")
            
        sortable_items.append({
            'index': i,
            'd_key': d_key,
            'sfc_idx': sfc_idx,
            'elem': elem,
            'x': x, 'y': y, 'z': z
        })

    # Step 2: Sort by (d_key, sfc_idx)
    # This ensures discrete slices are contiguous, and within slices, spatial locality is preserved
    sortable_items.sort(key=lambda item: (item['d_key'], item['sfc_idx']))

    # Step 3: Chunking and AABB Precomputation
    chunks = []
    chunk_size = 1024 * 1024  # 1 MB example
    current_chunk_elements = []
    current_chunk_min_d = float('inf')
    current_chunk_max_d = float('-inf')
    current_chunk_min_x = float('inf')
    current_chunk_max_x = float('-inf')
    current_chunk_min_y = float('inf')
    current_chunk_max_y = float('-inf')
    current_chunk_min_z = float('inf')
    current_chunk_max_z = float('-inf')

    for item in sortable_items:
        current_chunk_elements.append(item)
        
        # Update AABB
        current_chunk_min_d = min(current_chunk_min_d, item['d_key'])
        current_chunk_max_d = max(current_chunk_max_d, item['d_key'])
        current_chunk_min_x = min(current_chunk_min_x, item['x'])
        current_chunk_max_x = max(current_chunk_max_x, item['x'])
        current_chunk_min_y = min(current_chunk_min_y, item['y'])
        current_chunk_max_y = max(current_chunk_max_y, item['y'])
        current_chunk_min_z = min(current_chunk_min_z, item['z'])
        current_chunk_max_z = max(current_chunk_max_z, item['z'])

        # Check if chunk is full
        if len(current_chunk_elements) >= chunk_size:
            chunks.append({
                'elements': current_chunk_elements,
                'aabb': {
                    'd_min': current_chunk_min_d,
                    'd_max': current_chunk_max_d,
                    'x_min': current_chunk_min_x,
                    'x_max': current_chunk_max_x,
                    'y_min': current_chunk_min_y,
                    'y_max': current_chunk_max_y,
                    'z_min': current_chunk_min_z,
                    'z_max': current_chunk_max_z
                }
            })
            current_chunk_elements = []
            # Reset AABB bounds
            current_chunk_min_d = float('inf')
            current_chunk_max_d = float('-inf')
            current_chunk_min_x = float('inf')
            current_chunk_max_x = float('-inf')
            current_chunk_min_y = float('inf')
            current_chunk_max_y = float('-inf')
            current_chunk_min_z = float('inf')
            current_chunk_max_z = float('-inf')

    # Handle remaining elements
    if current_chunk_elements:
        chunks.append({
            'elements': current_chunk_elements,
            'aabb': {
                'd_min': current_chunk_min_d,
                'd_max': current_chunk_max_d,
                'x_min': current_chunk_min_x,
                'x_max': current_chunk_max_x,
                'y_min': current_chunk_min_y,
                'y_max': current_chunk_max_y,
                'z_min': current_chunk_min_z,
                'z_max': current_chunk_max_z
            }
        })

    return chunks

def select_chunks_for_view(chunks: list, view_d_min: float, view_d_max: float, 
                           view_x_min: float, view_x_max: float, 
                           view_y_min: float, view_y_max: float, 
                           view_z_min: float, view_z_max: float, 
                           tolerance: float) -> list:
    """
    Selects chunks overlapping the current view using precomputed AABBs.
    """
    selected = []
    for chunk in chunks:
        aabb = chunk['aabb']
        
        # Check overlap in discrete dimension
        d_overlap = not (aabb['d_max'] < view_d_min - tolerance or 
                         aabb['d_min'] > view_d_max + tolerance)
        
        # Check overlap in spatial dimensions
        x_overlap = not (aabb['x_max'] < view_x_min - tolerance or 
                         aabb['x_min'] > view_x_max + tolerance)
        y_overlap = not (aabb['y_max'] < view_y_min - tolerance or 
                         aabb['y_min'] > view_y_max + tolerance)
        z_overlap = not (aabb['z_max'] < view_z_min - tolerance or 
                         aabb['z_min'] > view_z_max + tolerance)
        
        if d_overlap and x_overlap and y_overlap and z_overlap:
            selected.append(chunk)
            
    return selected
```

**Edge Cases and Pitfalls:**
-   **Chunk Spanning Discrete Boundaries:** A chunk may span multiple timepoints or channels if the spatial sorting causes elements from adjacent discrete slices to fall into the same memory block. This is handled by the AABB test, which includes the discrete range of the chunk [S1, unofficial].
-   **Precision:** Floating-point comparisons in AABB tests require a tolerance to avoid missing chunks due to minor view changes or coordinate quantization errors [S1, unofficial].
-   **SFC Choice:** Using Morton codes for serialization may result in larger data transfers due to poorer spatial locality compared to Hilbert curves [S1, unofficial].

## Subtopics

### Multi-Resolution Tetrahedral Region Indicator
A spatial indexing scheme for multi-resolution tetrahedral grids where a unique index encodes the intersection of tetrahedra containing a point across all resolution levels [S2]. It differs from standard voxel or grid indexing by using tetrahedral subdivisions and concatenating level-wise indices to form a hierarchical region indicator [S2]. It is chosen when analytic isosurface extraction is required on multi-resolution tetrahedral meshes [S2]. The region indicator is defined as $\mathbf{r}(\mathbf{x}) := \bigoplus_{\ell=0}^{L-1} \mathbf{r}^{(\ell)}(\mathbf{x}) \in \mathbb{Z}^{4L}$, where $\mathbf{r}^{(\ell)}(\mathbf{x}) = [ a^{(\ell)}(\mathbf{x}), t^{(\ell)}(\mathbf{x}) ]$ consists of the anchor displacement and tetrahedron index at level $\ell$ [S2]. Typical parameters are $L=4$ levels and feature dimension $d=2$ per level [S2].

## Operating characteristics
-   **Time Complexity:** Sorting is $O(N \log N)$ where $N$ is the number of elements [S1, unofficial]. SFC encoding/decoding is $O(1)$ per element for Morton and $O(\log(\max(X,Y,Z)))$ for Hilbert [S1, unofficial]. AABB culling is $O(C)$ where $C$ is the number of chunks [S1, unofficial].
-   **Memory Complexity:** $O(N)$ for storing the sorted data and metadata [S1, unofficial].
-   **Performance:** Hilbert curves provide better spatial locality, leading to fewer cache misses and more efficient HTTP range requests for web viewers [S1, unofficial]. Morton codes are faster to compute, making them suitable for in-memory processing [S1, unofficial].
-   **Strengths:** Optimizes I/O for web-based visualization by coalescing range requests; supports multi-dimensional data (time, channel) efficiently [S1, unofficial].
-   **Limitations:** Requires precomputation of SFC indices and AABBs; Hilbert curve computation is more expensive than Morton [S1, unofficial].
-   **When to Use:** Web-based visualization of large multi-dimensional scientific data where network bandwidth is a constraint [S1, unofficial].
-   **When Not to Use:** Real-time rendering on GPU where data is already resident in video memory and cache locality is managed by hardware [S1, unofficial].

## Background
The method originates from the Luxar viewer architecture for scalable interactive web visualization of multidimensional scientific data [S1, unofficial]. It was developed to address the challenges of serving large Gaussian splatting datasets over the web, where efficient chunking and spatial locality are critical for performance [S1, unofficial]. The use of space-filling curves for spatial indexing is a well-established technique in computer graphics, with Morton codes being widely used for in-memory structures and Hilbert curves gaining popularity for serialization due to their superior locality [S1, unofficial].

## Sources
[S1] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
[S2] TetraSDF: Analytic Isosurface Extraction with Multi-resolution Tetrahedral Grid, https://arxiv.org/abs/2511.16273v4, fetched 2026-10-26
